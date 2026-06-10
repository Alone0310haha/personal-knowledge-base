# 283. 移动零

给定一个数组 nums，编写一个函数将所有 0 移动到数组的末尾，同时保持非零元素的相对顺序。

请注意 ，必须在不复制数组的情况下原地对数组进行操作。

[移动零](https://leetcode.cn/problems/move-zeroes/)

## 题解1

用快慢指针，实际上两个指针只会出现4种情况，只有0 1这种情况是需要交换的。不过为了代码统一 1 1也可以做交换冗余操作。
left表示前序已经处理好的数组的尾部，right表示当前还未处理的数据头部。每当right不为0的时候就把left和right的值交换。


```java
class Solution {
    public void moveZeroes(int[] nums) {
        int left = 0, right = 0;
        while(right < nums.length) {
            if(nums[right] != 0) {
                swap(nums, left, right);
                left++;
            }
            right++;
        }
    }

    void swap(int[] nums, int left, int right) {
        int temp = nums[left];
        nums[left] = nums[right];
        nums[right] = temp;
    }
}
```

还有一种方式是找到所有非零的元素，后面补0.

