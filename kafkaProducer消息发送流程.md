# kafkaProducer 消息发送流程

## `send`方法

kafkaProducer的send方法内部经过`this.interceptors`处理消息`ProducerRecord`后，使用doSend方法发送消息

``` java
public Future<RecordMetadata> send(ProducerRecord<K, V> record) {
    return send(record, null);
}
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
    // intercept the record, which can be potentially modified; this method does not throw exceptions
    ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
    return doSend(interceptedRecord, callback);
}
```

## `dosend`方法

方法信息量很大，只要分以下几步：

1. 获取kafka集群的元数据
2. 序列化key、value，验证序列化后消息是否过大
3. 将消息交予`accumulator`处理

主要看一下`accumulator`是如何处理信息的

``` java
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
    // 对应一个topic的一个partition
    TopicPartition tp = null;
    try {
        // 如果已关闭则抛出异常
        throwIfProducerClosed();
        // first make sure the metadata for the topic is available
        long nowMs = time.milliseconds();
        ClusterAndWaitTime clusterAndWaitTime;
        // 获取元数据
        try {
            clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);
        } catch (KafkaException e) {
            if (metadata.isClosed())
                throw new KafkaException("Producer closed while send in progress", e);
            throw e;
        }
        // 
        nowMs += clusterAndWaitTime.waitedOnMetadataMs;
        long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
        Cluster cluster = clusterAndWaitTime.cluster;
        byte[] serializedKey;
        try {
            // 序列化key
            serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in key.serializer", cce);
        }
        byte[] serializedValue;
        try {
            // 序列化value
            serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in value.serializer", cce);
        }
        // 计算具体发送到哪一个partition，然后创建TopicPartition对象
        int partition = partition(record, serializedKey, serializedValue, cluster);
        tp = new TopicPartition(record.topic(), partition);

        setReadOnly(record.headers());
        Header[] headers = record.headers().toArray();

        // 确保发送的消息不会过大
        int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                compressionType, serializedKey, serializedValue, headers);
        ensureValidRecordSize(serializedSize);
        long timestamp = record.timestamp() == null ? nowMs : record.timestamp();
        if (log.isTraceEnabled()) {
            log.trace("Attempting to append record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
        }
        // producer callback will make sure to call both 'callback' and interceptor callback
        Callback interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);

        if (transactionManager != null && transactionManager.isTransactional()) {
            transactionManager.failIfNotReadyForSend();
        }
        // accumulator将消息添加到缓存，然后返回结果
        RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
                serializedValue, headers, interceptCallback, remainingWaitMs, true, nowMs);
        // 第一次append时传入的时true，在append失败后会返回到这里
        // 然后重新计算partition，然后再次append，这次传入的是false
        // 如果再失败就会在append方法中创建新的batch
        if (result.abortForNewBatch) {
            int prevPartition = partition;
            partitioner.onNewBatch(record.topic(), cluster, prevPartition);
            partition = partition(record, serializedKey, serializedValue, cluster);
            tp = new TopicPartition(record.topic(), partition);
            if (log.isTraceEnabled()) {
                log.trace("Retrying append due to new batch creation for topic {} partition {}. The old partition was {}", record.topic(), partition, prevPartition);
            }
            // producer callback will make sure to call both 'callback' and interceptor callback
            interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);

            result = accumulator.append(tp, timestamp, serializedKey,
                serializedValue, headers, interceptCallback, remainingWaitMs, false, nowMs);
        }

        if (transactionManager != null && transactionManager.isTransactional())
            transactionManager.maybeAddPartitionToTransaction(tp);

        if (result.batchIsFull || result.newBatchCreated) {
            log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
            this.sender.wakeup();
        }
        return result.future;
        // handling exceptions and record the errors;
        // for API exceptions return them in the future,
        // for other exceptions throw directly
    } catch (ApiException e) {
        log.debug("Exception occurred during message send:", e);
        if (callback != null)
            callback.onCompletion(null, e);
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        return new FutureFailure(e);
    } catch (InterruptedException e) {
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        throw new InterruptException(e);
    } catch (BufferExhaustedException e) {
        this.errors.record();
        this.metrics.sensor("buffer-exhausted-records").record();
        this.interceptors.onSendError(record, tp, e);
        throw e;
    } catch (KafkaException e) {
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        throw e;
    } catch (Exception e) {
        // we notify interceptor about all exceptions, since onSend is called before anything else in this method
        this.interceptors.onSendError(record, tp, e);
        throw e;
    }
    }
```

## RecordAccumulator类

该类负责将消息添加到缓存区，在缓冲区消息等待被批量发送

主要字段

``` java
// ProducerBatch存放消息，达到该大小就创建新的ProducerBatch
private final int batchSize;
// ProducerBatch自创建在缓存中存在的最长时间
private final int lingerMs;
// batches缓存消息，每个TopicPartition对应一个ProducerBatch双端队列
// 消息根据opicPartition缓存到对应的ProducerBatch队列中
private final ConcurrentMap<TopicPartition, Deque<ProducerBatch>> batches;
```

`append`方法：主要有以下几个操作：

1. 获取消息应该存放的Deque
2. 在同步块中tryAppend方法尝试存放消息，因为KafkaProducer是多线程共享的，所以多线程向deque中存放消息需要同步
3. appendResult为null说明在第一次存放时，没有创建缓存区，创建缓存
4. 在缓存使用前再次尝试存放消息，如果成功，则在finally中释放缓存
5. 如果再次失败，则使用缓存创建新的ProducerBatch并添加到队列尾部

``` java
public RecordAppendResult append(TopicPartition tp,
                                    long timestamp,
                                    byte[] key,
                                    byte[] value,
                                    Header[] headers,
                                    Callback callback,
                                    long maxTimeToBlock,
                                    boolean abortOnNewBatch,
                                    long nowMs) throws InterruptedException {
    appendsInProgress.incrementAndGet();
    ByteBuffer buffer = null;
    if (headers == null) headers = Record.EMPTY_HEADERS;
    try {
        Deque<ProducerBatch> dq = getOrCreateDeque(tp);
        synchronized (dq) {
            if (closed)
                throw new KafkaException("Producer closed while send in progress");
            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq, nowMs);
            if (appendResult != null)
                return appendResult;
        }

        if (abortOnNewBatch) {
            return new RecordAppendResult(null, false, false, true);
        }
        // magic有0、1、2多个版本，如果指定压缩方式，在0、1版本每个batch只会存放一条消息，2及以上版本已经无视了这个条件可以存放多条
        // 最新的版本默认返回2
        byte maxUsableMagic = apiVersions.maxUsableProduceMagic();
        // 计算创建buffer的大小，如果消息大小大于batchSize，则取消息的大小
        int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));
        log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
        // buffer分配空间，暂时还没有使用
        buffer = free.allocate(size, maxTimeToBlock);
        // 再次尝试存放消息，如果成功则返回，如果失败则创建新的ProducerBatch使用buffer的空间
        nowMs = time.milliseconds();
        synchronized (dq) {
            if (closed)
                throw new KafkaException("Producer closed while send in progress");

            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq, nowMs);
            if (appendResult != null) {
                return appendResult;
            }
            // 创建ProducerBatch
            MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
            ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, nowMs);
            // 然后再次尝试存放消息
            FutureRecordMetadata future = Objects.requireNonNull(batch.tryAppend(timestamp, key, value, headers,
                    callback, nowMs));
            // 将ProducerBatch添加到队列的末尾
            dq.addLast(batch);
            // 这里存放kafka服务器还没有返回ack的batch，包括已发送的和还没发送的
            incomplete.add(batch);
            // 如果buffer实际被使用了，将该字段设为null，帮助在finally中清理内存
            // 因为在buffer使用之前，调用了tryAppend，如果这次成功就会直接返回，那么buffer就会在finally中被清理
            buffer = null;
            // 返回结果
            return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true, false);
        }
    } finally {
        if (buffer != null)
            free.deallocate(buffer);
        appendsInProgress.decrementAndGet();
    }
}
```

`tryAppend`方法：

``` java
private RecordAppendResult tryAppend(long timestamp, byte[] key, byte[] value, Header[] headers,
                                        Callback callback, Deque<ProducerBatch> deque, long nowMs) {
    // 获取队列中的最后一个ProducerBatch
    ProducerBatch last = deque.peekLast();
    // 如果存在ProducerBatch，将消息尝试添加到其中
    if (last != null) {
        FutureRecordMetadata future = last.tryAppend(timestamp, key, value, headers, callback, nowMs);
        if (future == null)
            last.closeForRecordAppends();
        else
            return new RecordAppendResult(future, deque.size() > 1 || last.isFull(), false, false);
    }
    // 如果last为null 说明该队列是空的，返回null，在append方法中创建ProducerBatch
    return null;
}
```

## ProducerBatch类

`ProducerBatch.tryAppend`方法：

``` java
public FutureRecordMetadata tryAppend(long timestamp, byte[] key, byte[] value, Header[] headers, Callback callback, long now) {
    // 当前batch没有足够空间，返回null，表示需要创建新的batch
    if (!recordsBuilder.hasRoomFor(timestamp, key, value, headers)) {
        return null;
    } else {
        Long checksum = this.recordsBuilder.append(timestamp, key, value, headers);
        this.maxRecordSize = Math.max(this.maxRecordSize, AbstractRecords.estimateSizeInBytesUpperBound(magic(),
                recordsBuilder.compressionType(), key, value, headers));
        this.lastAppendTime = now;
        FutureRecordMetadata future = new FutureRecordMetadata(this.produceFuture, this.recordCount,
                                                                timestamp, checksum,
                                                                key == null ? -1 : key.length,
                                                                value == null ? -1 : value.length,
                                                                Time.SYSTEM);
        // we have to keep every future returned to the users in case the batch needs to be
        // split to several new batches and resent.
        thunks.add(new Thunk(callback, future));
        this.recordCount++;
        return future;
    }
}
```
