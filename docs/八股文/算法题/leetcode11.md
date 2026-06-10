# leetcode 11

给定一个长度为 n 的整数数组 height 。有 n 条垂线，第 i 条线的两个端点是 (i, 0) 和 (i, height[i]) 。

找出其中的两条线，使得它们与 x 轴共同构成的容器可以容纳最多的水。

返回容器可以储存的最大水量。

[盛最多水的容器](https://leetcode.cn/problems/container-with-most-water/)

题解：

两个指针对向移动，因为两个指针之间的距离是越来越近的，所以如果前一个指针的高度更矮，那结果只会更小。所以应当移动两指针中值较小的指针。

```java
class Solution {
    public int maxArea(int[] height) {
        int left = 0, right = height.length  - 1;
        int result = 0;
        while(left < right) {
            result = Math.max(result, Math.min(height[left], height[right]) * (right - left));
            if(height[left] < height[right]) {
                left++;
            } else {
                right--;
            }
        }
        return result;
    }
}
```
