---
title: 二分查找-变体4
layout: default
tags: algo
permalink: /docs/algo/binary_search_variant_04
parent: Algorithms
nav_order: 8
---
# 二分查找-查找最后一个小于等于给定值的元素
## 描述
查找最后一个小于等于给定值的元素。比如，数组中存储了这样一组数据：3，5，6，8，9，10。最后一个小于等于 7 的元素就是 6。
## 解决步骤

1. 在二分查找-简单实现基础上进行改造
2. 只存在两个分支，即小于等于或大于
3. 特殊节点：`nums[mid] <= target`  处特殊处理，判断 `mid` 是否为最后一个元素或 `mid+1` 的元素是否大于目标值
4. 特殊节点：如不满足步骤 2 中的条件，则说明目标元素在 `mid` 后面，此时缩小范围，即为 `[mid+1, high]`

## 编码实现
```javascript

/**
 * @param {number[]} nums
 * @param {number} target
 * @return {number[]}
 */
const searchLastLtePosition = function (nums, target) {
    let low = 0;
    let high = nums.length - 1;
    while (low <= high) {
        const mid = Math.floor(low + (high - low) / 2);
        if (nums[mid] <= target) {
            if (mid === nums.length - 1 || nums[mid + 1] > target) {
                return mid;
            } else {
                low = mid + 1;
            }
        } else {
            high = mid - 1;
        }
    }
    return -1;
};
```
