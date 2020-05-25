# kafkatemplate对kafkaProducer的封装

## kafkatemplate发送方法

``` java
public ListenableFuture<SendResult<K, V>> sendDefault(@Nullable V data)
public ListenableFuture<SendResult<K, V>> sendDefault(K key, @Nullable V data)
public ListenableFuture<SendResult<K, V>> sendDefault(Integer partition, K key, @Nullable V data)
public ListenableFuture<SendResult<K, V>> sendDefault(Integer partition, Long timestamp, K key, @Nullable V data)
public ListenableFuture<SendResult<K, V>> send(String topic, @Nullable V data)
public ListenableFuture<SendResult<K, V>> send(String topic, K key, @Nullable V data)
public ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, K key, @Nullable V data)
public ListenableFuture<SendResult<K, V>> send(String topic, Integer partition, Long timestamp, K key,@Nullable V data)
public ListenableFuture<SendResult<K, V>> send(ProducerRecord<K, V> record)
public ListenableFuture<SendResult<K, V>> send(Message<?> message)
```

这些发送方法内部生成一个`ProducerRecord`对象，传递给`dosend`方法，该对象包含了发送给kafka的所有信息

``` java
public class ProducerRecord<K, V> {

    private final String topic;
    private final Integer partition;
    private final Headers headers;
    private final K key;
    private final V value;
    private final Long timestamp;
```

`doSend`方法：通过`getTheProducer`获得`KafkaProducer`对象，`KafkaProducer`对象发送数据`ProducerRecord`

``` java
protected ListenableFuture<SendResult<K, V>> doSend(final ProducerRecord<K, V> producerRecord) {

    // 获取KafkaPruducer对象
    final Producer<K, V> producer = getTheProducer(producerRecord.topic());
    this.logger.trace(() -> "Sending: " + producerRecord);
    final SettableListenableFuture<SendResult<K, V>> future = new SettableListenableFuture<>();
    Object sample = null;
    if (this.micrometerEnabled && this.micrometerHolder == null) {
        this.micrometerHolder = obtainMicrometerHolder();
    }
    if (this.micrometerHolder != null) {
        sample = this.micrometerHolder.start();
    }

    // 执行kafkaProducer的send方法，发送数据
    Future<RecordMetadata> sendFuture =
            producer.send(producerRecord, buildCallback(producerRecord, producer, future, sample));
    // May be an immediate failure
    if (sendFuture.isDone()) {
        try {
            sendFuture.get();
        }
        catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new KafkaException("Interrupted", e);
        }
        catch (ExecutionException e) {
            throw new KafkaException("Send failed", e.getCause()); // NOSONAR, stack trace
        }
    }
    if (this.autoFlush) {
        flush();
    }
    this.logger.trace(() -> "Sent: " + producerRecord);
    return future;
}
```

`getTheProducer`方法，前面的是开启事务的Producer创建，暂不了解；对于非事务的Producer创建主要在最后两个if判断中，都是获取当前`KafkaTemplate`的`producerFactory`，`producerFactory`用来创建KafkaProducer。`getProducerFactory(topic)`方法通过传入的topic来确定producerFactory，默认是返回this.producerFactory，可以在子类中重写具体的策略。剩下的就转到了`producerFactory`的`createProducer`方法。

``` java
protected Producer<K, V> getTheProducer(@SuppressWarnings("unused") @Nullable String topic) {
    // 开启事务时创建事务Producer
    boolean transactionalProducer = this.transactional;
    if (transactionalProducer) {
        boolean inTransaction = inTransaction();
        Assert.state(this.allowNonTransactional || inTransaction,
                "No transaction is in process; "
                    + "possible solutions: run the template operation within the scope of a "
                    + "template.executeInTransaction() operation, start a transaction with @Transactional "
                    + "before invoking the template method, "
                    + "run in a transaction started by a listener container when consuming a record");
        if (!inTransaction) {
            transactionalProducer = false;
        }
    }
    if (transactionalProducer) {
        Producer<K, V> producer = this.producers.get();
        if (producer != null) {
            return producer;
        }
        KafkaResourceHolder<K, V> holder = ProducerFactoryUtils
                .getTransactionalResourceHolder(this.producerFactory, this.transactionIdPrefix, this.closeTimeout);
        return holder.getProducer();
    }
    else if (this.allowNonTransactional) {
        return this.producerFactory.createNonTransactionalProducer();
    }
    // 创建非事务Producer
    else if (topic == null) {
        return this.producerFactory.createProducer();
    }
    else {
        return getProducerFactory(topic).createProducer();
    }
}
```

`createProducer`方法：`ProducerFactory`接口只有一个默认实现类`DefaultKafkaProducerFactory`，又调用了`doCreateProducer`方法

``` java
public Producer<K, V> createProducer() {
    return createProducer(this.transactionIdPrefix);
}
public Producer<K, V> createProducer(@Nullable String txIdPrefixArg) {
    String txIdPrefix = txIdPrefixArg == null ? this.transactionIdPrefix : txIdPrefixArg;
    return doCreateProducer(txIdPrefix);
}
```

`doCreateProducer`方法：

1. 第一个if中是创建事务Producer
2. 第二个if中为每个线程创建一个Producer，Kafka文档中建议多线程共享一个Producer，当然也可以每个线程创建一个
3. synchronized同步块中是创建非事务的Producer，保证多线程也只会创建一个Producer实例，这个Producer实例通过`CloseSafeProducer`类代理

``` java
private Producer<K, V> doCreateProducer(@Nullable String txIdPrefix) {
    if (txIdPrefix != null) {
        if (this.producerPerConsumerPartition) {
            return createTransactionalProducerForPartition(txIdPrefix);
        }
        else {
            return createTransactionalProducer(txIdPrefix);
        }
    }
    if (this.producerPerThread) {
        CloseSafeProducer<K, V> tlProducer = this.threadBoundProducers.get();
        if (this.threadBoundProducerEpochs.get() == null) {
            this.threadBoundProducerEpochs.set(this.epoch.get());
        }
        if (tlProducer != null && this.epoch.get() != this.threadBoundProducerEpochs.get()) {
            closeThreadBoundProducer();
            tlProducer = null;
        }
        if (tlProducer == null) {
            tlProducer = new CloseSafeProducer<>(createKafkaProducer(), this::removeProducer,
                    this.physicalCloseTimeout, this.beanName);
            for (Listener<K, V> listener : this.listeners) {
                listener.producerAdded(tlProducer.clientId, tlProducer);
            }
            this.threadBoundProducers.set(tlProducer);
            this.threadBoundProducerEpochs.set(this.epoch.get());
        }
        return tlProducer;
    }
    synchronized (this) {
        // 创建一个CloseSafeProducer对象
        if (this.producer == null) {
            this.producer = new CloseSafeProducer<>(createKafkaProducer(), this::removeProducer,
                    this.physicalCloseTimeout, this.beanName);
            this.listeners.forEach(listener -> listener.producerAdded(this.producer.clientId, this.producer));
        }
        return this.producer;
    }
}
```

`createKafkaProducer`方法：根据clientIdPrefix处理配置问题，clientId就是创建的Producer的名字标识

``` java
protected Producer<K, V> createKafkaProducer() {
    Map<String, Object> newConfigs;
    if (this.clientIdPrefix == null) {
        newConfigs = new HashMap<>(this.configs);
    }
    else {
        newConfigs = new HashMap<>(this.configs);
        newConfigs.put(ProducerConfig.CLIENT_ID_CONFIG,
                this.clientIdPrefix + "-" + this.clientIdCounter.incrementAndGet());
    }
    checkBootstrap(newConfigs);
    return createRawProducer(newConfigs);
}
protected Producer<K, V> createRawProducer(Map<String, Object> rawConfigs) {
    return new KafkaProducer<>(rawConfigs, this.keySerializerSupplier.get(), this.valueSerializerSupplier.get());
}
```

`CloseSafeProducer`代理了KafkaProducer实例，在创建对象时主要传入了KafkaProducer实例、关闭KafkaProducer的行为：this::removeProducer，其send方法就是调用调用代理的KafkaProducer的send方法。Callback回调中处理的时事务Producer的异常问题。

``` java
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
    LOGGER.trace(() -> toString() + " send(" + record + ")");
    return this.delegate.send(record, new Callback() {

        @Override
        public void onCompletion(RecordMetadata metadata, Exception exception) {
            if (exception instanceof OutOfOrderSequenceException) {
                CloseSafeProducer.this.producerFailed = exception;
                close(CloseSafeProducer.this.closeTimeout);
            }
            callback.onCompletion(metadata, exception);
        }

    });
}
```

CloseSafeProducer，对于非事务Producer来说只有send方法是有用的，其中的close等相关方法都是针对事务Producer。

非事务Producer是通过`closeDelegate`来关闭的，该方法在`destroy`方法中调用。在spring应用结束运行时会调用destroy方法，然后Producer的所有资源都会被释放。
