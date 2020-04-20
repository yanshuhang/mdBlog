# 寻找两个有序数组的中位数

## 问题描述

给定两个大小为 m 和 n 的有序数组 nums1 和 nums2。

请你找出这两个有序数组的中位数，并且要求算法的时间复杂度为 O(log(m + n))。

你可以假设 nums1 和 nums2 不会同时为空。

示例 1:

``` text
nums1 = [1, 3]
nums2 = [2]

则中位数是 2.0
```

示例 2:

``` text
nums1 = [1, 2]
nums2 = [3, 4]

则中位数是 (2 + 3)/2 = 2.5
```

## 解答

在一个排序好的数组中寻找中位数，找其中间的数即可(最多判断下奇数、偶数)  
在两个这样的数组a、b中找中位数依然可以使用这样的思—中位数将数组分为了数量相等的两部分left、right，其中a的长度为m，b的长度为n(我们默认a的长度更小)。那么：

* leftLen = (m+n+1)/2，当总数为奇数时，在left我们多放一个
* rightlen = (m+n)-leftLen，这个数用不到

我们假设在left中有i个a中的元素、j个b中的元素，i可以是1-m的任意数，而且有`j=leftLen-i`的关系，由于a、b都是排序好的数组，那么i个a中的元素必定是`a[0]到a[i-1]`，j个b中的元素必为`b[0]到b[j-1]`

那么，我们就可以在a中遍历找到这个i，在有序数组中遍历使用二分法最为方便，唯一的问题是如何判断找到了那个正确的i，这里需要判断a[i-1]、a[i]、b[j-1]、b[j]之间的大小关系，可以确定的是a[i-1]<a[i],b[j-1]<b[j]：

1. b[j-1] > a[i]，left中b的部分有比right中大的数，所以i应该更大，所以i往上继续二分查找
2. a[i-1] > b[j]，left中a的部分有比right中大的数，所以i应该更小，所以i往下继续二分查找
3. 上面两个条件都不符合了，我们就找到了合适的i，此时left中的数都比right中的小

剩下的就是还有一些边界上的判断和奇、偶数的判断：

1. i和j有可能达到了边界0、m、n，简单的判断下就可以得出left中的最大数：maxLeft
2. m+n为奇数，由于我们开始就再left中多放了一个，直接返回maxLeft即可
3. m+n为偶数，需要找出right中的最小数：minRight，跟第1步逻辑一致，然后返回(maxLeft+minRight)/2.0即可

``` java
public double solution(int[] nums1, int[] nums2) {
    int m = nums1.length;
    int n = nums2.length;
    // 确保m<=n，如果m>n 在nums1中查找i时，j有可能是负数
    if (m > n) {
        int[] temp = nums1;
        nums1 = nums2;
        nums2 = temp;

        int tmp = m;
        m = n;
        n = tmp;
    }
    // iMin、iMax二分查找i，halfLen是左半部的长度，m+n是偶数是halfLen为一半，奇数时为一半+1
    int iMin = 0, iMax = m, halfLen = (m + n + 1) / 2;
    // i为nums1在左半部的个数、j为nums2在左半部的个数
    while (iMin <= iMax) {
        // 二分查找i，确定j
        int i = (iMin + iMax) / 2;
        int j = halfLen - i;
        // nums1[i-1]必小于nums[i]，所以判断nums2[j-1]即可
        // 如果成立，表面左半部存在比右半部大的数，所以j要更小、i要更大才行
        // i只有可能小于或等于iMax，如果相等那么说明i已经找到了
        // else if 中逻辑一致
        if (i < iMax && nums2[j-1] > nums1[i]) {
            iMin = i + 1;
        } else if (i > iMin && nums1[i-1] > nums2[j]) {
            iMax = i - 1;
        } else { // i刚刚好，下面只需要判断下边界即可
            int maxLeft = 0;
            // i=0 说明 左半部全是nums的元素，取nums2[j-1]
            // j=0 取nums1[i-1]
            // 如果两个数组的元素都有，取最大值
            if (i == 0) {
                maxLeft = nums2[j-1];
            } else if (j == 0) {
                maxLeft = nums1[i-1];
            } else {
                maxLeft = Math.max(nums1[i-1], nums2[j-1]);
            }
            // 找到了左半部的最大值 maxLeft，如果m+n是奇数，返回该值即可
            if ((m + n) % 2 == 1) {
                return maxLeft;
            }
            // 如果m+n是偶数 需要找到右半部的最小值，跟上方的逻辑一样
            int minRight = 0;
            if (i == m) {
                minRight = nums2[j];
            } else if (j == n) {
                minRight = nums1[i];
            } else {
                minRight = Math.min(nums1[i], nums2[j]);
            }
            return (maxLeft + minRight) / 2.0;
        }
    }
    return 0.0;
```
