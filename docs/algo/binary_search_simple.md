---
title: 二分查找-简单
layout: default
tags: algo
permalink: /docs/algo/binary_search_simple
parent: Algorithms
nav_order: 4
---
# 二分查找-简单
## 描述
给定一个 n 个元素有序的（升序）整型数组 nums 和一个目标值 target  ，写一个函数搜索 nums 中的 target，如果目标值存在返回下标，否则返回 -1。

示例 1:
```
输入: nums = [-1,0,3,5,9,12], target = 9
输出: 4
解释: 9 出现在 nums 中并且下标为 4
示例 2:
```
```
输入: nums = [-1,0,3,5,9,12], target = 2
输出: -1
解释: 2 不存在 nums 中因此返回 -1
```

## 解决步骤

1. 因为数组为已排序数组，故每次搜索可以从中间节点开始，并判断中间节点值，再选择向前或向后迭代查找
2. 定义低位 low，高位 high，循环条件：low <= high
3. 取 mid 中点，未避免运算溢出，可将求解 mid 做转换，在 JavaScript 中需要向下取整，否则可能会得到一个小数
4. 判断 nums[mid] 值，如相等则直接返回
5. nums[mid] < target, 则说明搜索区间在 [mid+1, high], 故更新 low
6. nums[mid] > target, 则说明搜索区间在 [low, mid-1], 故更新 high
7. 注意：简单的二分查找时基于没有重复元素的情况下进行的

## 编码实现
```javascript
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number}
 */
const search = function (nums, target) {
    let low = 0;
    let high = nums.length - 1;
    while (low <= high) {
        const mid = Math.floor(low + (high - low) / 2);
        if (nums[mid] === target) {
            return mid;
        } else if (nums[mid] < target) {
            low = mid + 1;
        } else {
            high = mid - 1;
        }
    }
    return -1;
};
```
