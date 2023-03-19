---
title: 二分查找-变体3
layout: default
tags: algo
permalink: /docs/algo/binary_search_variant_03
parent: Algorithms
nav_order: 7
---
# 二分查找-查找第一个大于等于给定值的元素
## 描述
在有序数组中，查找第一个大于等于给定值的元素。比如，数组中存储的这样一个序列：3，4，6，7，10。如果查找第一个大于等于 5 的元素，那就是 6。
## 解决步骤

1. 在二分查找-简单实现基础上进行改造
2. 只存在两个分支，即大于等于或小于
3. 特殊节点：`nums[mid] >= target`  处特殊处理，判断 `mid` 是否为第一个元素或 `mid-1` 的元素是否小于目标值
4. 特殊节点：如不满足步骤 2 中的条件，则说明目标元素在 `mid` 前面，此时缩小范围，即为 `[low, mid-1]`

## 编码实现
```javascript
/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number[]}
 */
const searchFirstGtePosition = function (nums, target) {
    let low = 0;
    let high = nums.length - 1;
    while (low <= high) {
        const mid = Math.floor(low + (high - low) / 2);
        if (nums[mid] >= target) {
            if (mid === 0 || nums[mid - 1] < target) {
                return mid;
            } else {
                high = mid - 1;
            }
        } else {
            low = mid + 1;
        }
    }
    return -1;
};
```
