# leetcode 128

给定一个未排序的整数数组 nums ，找出数字连续的最长序列（不要求序列元素在原数组中连续）的长度。

请你设计并实现时间复杂度为 O(n) 的算法解决此问题。

## 题解1

首先肯定不能纯排序，最快的快排都要O(nlogn)。核心方法就是遍历数组中的每一个元素x，判断x+1,x+2,...是否在序列中。
遍历整个序列用了O(n)，再判断一次O(n)，所以要O(n^2)，不符合题目要求。
又假设如果x~x+5都在序列中，那x~x+5肯定比x+1~x+5长。所以处理了x之后，x+1~x+5就不需要再处理了。这样就能省掉部分元素二次判断的时间。
所以在遍历序列的时候，先判断x-1是否在序列中，如果不在，才处理x。如果在，就跳过x。
判断x-1在序列中，可以用哈希表先把整个序列存一次，这样后续判断就是O(1)。总时间复杂度就是O(n)。

```java
import java.util.Set;

class Solution {
    public int longestConsecutive(int[] nums) {
        // 先存储到集合中
        Set<Integer> set = new HashSet<>(nums.length);
        int result = 0;
        for (int num : nums) {
            set.add(num);
        }
        for (int num : set) {
            if (!set.contains(num - 1)) {
                int curNum = num;
                int co = 1;
                while (set.contains(curNum + 1)) {
                    curNum++;
                    co++;
                }
                result = Math.max(result, co);
            }
        }
        return result;
    }
}
```


