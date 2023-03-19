---
title: 二分查找-变体1
layout: default
tags: algo
permalink: /docs/algo/binary_search_variant_01
parent: Algorithms
nav_order: 5
---
# 二分查找-查找第一个值等于给定值的元素
## 描述
给你一个按照非递减顺序排列的整数数组 nums，和一个目标值 target。请你找出给定目标值在数组中的开始位置。

如果数组中不存在目标值 target，返回 -1。

你必须设计并实现时间复杂度为 O(log n) 的算法解决此问题。

示例 1：

```
输入：nums = [5,7,7,8,8,10], target = 8
输出：3
```

示例 2：

```
输入：nums = [5,7,7,8,8,10], target = 6
输出：-1
```

示例 3：

```
输入：nums = [], target = 0
输出：-1
```

## 解决步骤

1. 在二分查找-简单实现基础上进行改造
1. 特殊节点：`nums[mid] === target`  处特殊处理，判断 `mid` 是否为第一个元素或 `mid-1` 的元素是否不等于目标值
1. 特殊节点：如不满足步骤 2 中的条件，则说明第一个值等于给定值的元素在 `mid` 前面，此时缩小范围，即为 `[low, mid-1]`

## 编码实现
```javascript
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number[]}
 */
const searchFirstPosition = function (nums, target) {
    let low = 0;
    let high = nums.length - 1;
    while (low <= high) {
        const mid = Math.floor(low + (high - low) / 2);
        if (nums[mid] === target) {
            if (mid === 0 || nums[mid - 1] !== target) {
                return mid;
            } else {
                high = mid - 1;
            }
        } else if (nums[mid] < target) {
            low = mid + 1;
        } else {
            high = mid - 1;
        }
    }
    return -1;
};
```
