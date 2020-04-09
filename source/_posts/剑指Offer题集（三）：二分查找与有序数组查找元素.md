---
title: 剑指Offer题集（三）：二分查找与有序数组查找元素
comments: true
top: false
date: 2020-04-06 16:25:33
tags:
	- Binary Search
	- Sorted Array
categories:
	- 剑指Offer
---

在无序的数组中查找目标值是否存在，只能通过遍历数组的方式，其时间复杂度为O(N)。

在**有序**的数组中查找目标值是否存在，可以通过遍历数组的方式，时间复杂度为O(N)。考虑到有序这一特性，还可以通过**二分查找**，使时间复杂度降至**O(log N)**。

<!--more-->

#### 面试题11. 旋转数组的最小数字

##### 1. 题目

假设按照升序排序的数组在预先未知的某个点上进行了旋转。例如，数组 `[0, 1, 2, 4, 5, 6, 7]` 可能变为 `[4, 5, 6, 7, 0, 1, 2]` ，最小的数字是 `0` 。

注意数组中**可能存在重复的元素**。

[力扣-《剑指Offer》-面试题11. 旋转数组的最小数字](https://leetcode-cn.com/problems/xuan-zhuan-shu-zu-de-zui-xiao-shu-zi-lcof/)

[力扣-154. 寻找旋转排序数组中的最小值 II](https://leetcode-cn.com/problems/find-minimum-in-rotated-sorted-array-ii/)

##### 2. 思路

（1）遍历数组，时间复杂度为O(N)。

（2）利用**数组部分有序**的特性，采用二分法，时间复杂度为O(log N)。

设置数组的左边界 `left` 和右边界 `right` 指针，及中间元素 `mid` 指针。将 `numbers[mid]` 与 `numbers[right]` 比较，会有如下三种情况：

- `numbers[mid] > numbers[right]` 。当前数组的中间元素和右边界不在同一侧，即最小元素值应该在 `mid` 和 `right` 之间，故令 `left = mid + 1`，继续二分查找。
- `numbers[mid] < numbers[right]` 。当前数组的中间元素和右边界在同一侧，即最小元素值应该在 **mid之前（含mid）**，故令 `right = mid`，继续二分查找。
- `numbers[mid] = numbers[right]` 。此时无法断定最小元素值是在 `mid` 的哪一侧，故令 `right = right - 1`，继续二分查找。若数组所有元素均相同，则时间复杂度恶化为O(N)。

##### 3. 实现

```java
class Solution {
    public int minArray(int[] numbers) {
        int left = 0;
        int right = numbers.length - 1;
        int mid;
        // 循环，直至left==right
        while(left < right){
            mid = left + (right-left)/2;
            if(numbers[mid] > numbers[right]){
                // 目标元素在右边
                left = mid + 1;
            }else if(numbers[mid] < numbers[right]){
                // 目标元素在左边
                right = mid;
            }else{
                right--;
            }
        }
        return numbers[left];
    }
}
```

算法分析：

- 时间复杂度：平均为O(log N)，最坏为O(N)。
- 空间复杂度：O(1)。

##### 4. 数组不含重复值的情况

[153. 寻找旋转排序数组中的最小值](https://leetcode-cn.com/problems/find-minimum-in-rotated-sorted-array/)

**数组不含重复值**：此时不存在上述 `numbers[mid] = numbers[right]` 的情形，算法的时间复杂度稳定为O(log N)。

```java
class Solution {
    public int findMin(int[] nums) {
        // 二分法
        int left = 0;
        int right = nums.length-1;
        int mid = 0;
        while(left < right){
            mid = left + (right-left)/2; // 防加和溢出
            if(nums[mid] > nums[right]){
                // 目标元素在右边
                left = mid + 1;
            }else{
                // 目标元素在左边
                right = mid;
            }
            /*
            不同于154题的元素可重复，本题元素不可重复，即
            不存在"nums[mid]==nums[right]"的情况，因此使得
            算法的时间复杂度稳定为O(logN)
            */
        }
        return nums[left];
    }
}
```

---

#### 面试题53-I. 在排序数组中查找数字 I

##### 1. 题目

在有序的数组中找出指定值的左右边界，指定值可有多个，也可不存在。要求算法的时间复杂度为O(log N)。

 [力扣-《剑指Offer》-面试题53-I. 在排序数组中查找数字 I](https://leetcode-cn.com/problems/zai-pai-xu-shu-zu-zhong-cha-zhao-shu-zi-lcof/)

  [力扣-34. 在排序数组中查找元素的第一个和最后一个位置](https://leetcode-cn.com/problems/find-first-and-last-position-of-element-in-sorted-array/)

##### 2. 思路

**有序数组 + O(log N)的时间复杂度 == 二分法**

不同于普通的二分法查找目标值是否存在，此类题不仅需要明确目标值是否存在，还要确定存在的目标值的左右索引。

设 `left`、`right`、`mid` 指针。显然，有如下三种情况：

（1）`nums[mid] < target`，目标值可能位于数组右侧，令 `left = mid + 1`，继续二分查找。

（2）`nums[mid] > target`，目标值可能位于数组左侧，令 `right = mid - 1`，继续二分查找。

（3）`nums[mid] = target`，匹配到目标值。此时，需要进一步确定目标值的左右边界。

- 对于左边界：若 `mid > 0 && nums[mid] == nums[mid-1]` 成立，令 `right = mid - 1`，继续二分查找；否则，`mid` 为左边界，直接返回。
- 对于右边界：若 `mid < nums.length - 1 && nums[mid] == nums[mid+1]` 成立，令 `right = mid - 1`，继续二分查找；否则，`mid` 为右边界，直接返回。

##### 3. 实现

```java
class Solution {

    // 二分法，使得时间复杂度为O(logN)以达到题目要求。
    public int[] searchRange(int[] nums, int target) {
        int[] res = {-1, -1};
        if(nums==null || nums.length==0 || target < nums[0] || target > nums[nums.length-1]){
            return res;
        }

        int leftIndex = binarySearch(nums, target, true);
        res[0] = leftIndex;
        int rightIndex = binarySearch(nums, target, false);
        res[1] = rightIndex;
        return res;
    }

    // flag标志位：flag为true时表示，当匹配到目标值时，进入左边寻找左边界；
    // flag为false时表示，当匹配到目标值时，进入右边寻找右边界
    private static int binarySearch(int[] arr, int target, boolean flag){
        int left = 0;
        int right = arr.length - 1;
        int mid = 0;

        while(left <= right){
            mid = left + (right-left)/2;
            if(arr[mid]==target){
                // 匹配到目标值
                if(flag){
                    // 寻找左边界
                    if(mid > 0 && arr[mid]==arr[mid-1]){
                        right = mid - 1;
                    }else{
                        return mid; // mid即为左边界
                    }
                }else{
                    // 寻找右边界
                    if(mid < arr.length - 1 && arr[mid]==arr[mid+1]){
                        left = mid + 1;
                    }else{
                        return mid; // mid即为右边界
                    }
                }
            }else if(arr[mid] > target){
                // 目标值可能在当前数组的左边
                right = mid - 1;
            }else{
                // 目标值可能在当前数组的右边
                left = mid + 1;
            }
        }
        return -1;
    }
}
```

算法分析：

- 时间复杂度：O(log N)。
- 空间复杂度：O(1)。

---

