---
title: 验证IP地址
layout: default
tags: algo
permalink: /docs/algo/validate-ip-address
parent: Algorithms
---
# 验证IP地址

## 描述

[题目地址-468. 验证IP地址](https://leetcode.cn/problems/validate-ip-address/description/)

给定一个字符串 queryIP。如果是有效的 IPv4 地址，返回 "IPv4" ；如果是有效的 IPv6 地址，返回 "IPv6" ；如果不是上述类型的 IP 地址，返回 "Neither" 。

有效的IPv4地址 是 “x1.x2.x3.x4” 形式的IP地址。 其中 0 <= xi <= 255 且 xi 不能包含 前导零。例如: “192.168.1.1” 、 “192.168.1.0” 为有效IPv4地址， “192.168.01.1” 为无效IPv4地址; “192.168.1.00” 、 “192.168@1.1” 为无效IPv4地址。

一个有效的IPv6地址 是一个格式为“x1:x2:x3:x4:x5:x6:x7:x8” 的IP地址，其中:

1 <= xi.length <= 4
xi 是一个 十六进制字符串 ，可以包含数字、小写英文字母( 'a' 到 'f' )和大写英文字母( 'A' 到 'F' )。
在 xi 中允许前导零。
例如 "2001:0db8:85a3:0000:0000:8a2e:0370:7334" 和 "2001:db8:85a3:0:0:8A2E:0370:7334" 是有效的 IPv6 地址，而 "2001:0db8:85a3::8A2E:037j:7334" 和 "02001:0db8:85a3:0000:0000:8a2e:0370:7334" 是无效的 IPv6 地址。

示例 1：

```
输入：queryIP = "172.16.254.1"
输出："IPv4"
解释：有效的 IPv4 地址，返回 "IPv4"
```

示例 2：

```
输入：queryIP = "2001:0db8:85a3:0:0:8A2E:0370:7334"
输出："IPv6"
解释：有效的 IPv6 地址，返回 "IPv6"
```

示例 3：

```
输入：queryIP = "256.256.256.256"
输出："Neither"
解释：既不是 IPv4 地址，又不是 IPv6 地址
```

## 解决步骤

1. 分 IPV4 和 IPV6 进行单独处理，判断条件，字符串中是否包含 `.`
2. 整体思想：双指针实现，即根据前后指针截取 IP 地址中的每一段内容，按规则对每一个段进行校验
3. 对于 IPV4 步骤如下
   - IPV4 固定4段，故循环4次
   - 每次循环找到 `last+1` 后最近的 `.` 所在的索引，此处需特殊判断最后一段，最后一段不存在 `.`
   - 判断 `cur-last-1` 长度是否符合 IPV4 单段长度范围，此处可以避免未找到或长度过长导致后续加法溢出情况
   - 对单段字符串计算 `num`
   - 判断 `num` 是否合法，其中特殊判断是否存在前导零
   - 更新 `last` 值
4. 对于 IPV6 步骤如下
   - IPV6 固定8段，故循环8次
   - 同步骤3中获取cur
   - 判断每段内容是否符合 IPV6 规范
   - 更新 `last` 值

## 编码实现

```javascript

'use strict'
/**
 * @param {string} queryIP
 * @return {string}
 */
const validIPAddress = function (queryIP) {
    if (queryIP.includes(".")) {
        // 定义 "." 开始寻找的位置
        let last = -1;
        // IPV4 只存在4段
        for (let i = 0; i < 4; i++) {
            // 在 last+1 处开始寻找"."在索引第一次出现的位置
            let cur = i === 3 ? queryIP.length : queryIP.indexOf(".", last + 1);
            // 边界判断，若其小于1或大于3则为 Neither
            if (cur - last - 1 < 1 || cur - last - 1 > 3) {
                return "Neither";
            }
            // 每段结果记录
            let num = 0;
            // 遍历每段上的字符
            for (let j = last + 1; j < cur; j++) {
                if (queryIP[j] < '0' || queryIP[j] > '9') {
                    return "Neither";
                }
                num = num * 10 + parseInt(queryIP[j]);
            }
            if (num > 255) {
                return "Neither";
            }
            // 判断是否存在前导零情况
            if (num === 0 && cur - last - 1 > 1) {
                return "Neither";
            }
            if (num > 0 && queryIP[last + 1] === '0') {
                return "Neither";
            }
            last = cur;
        }
        return "IPv4";
    } else {
        let last = -1;
        for (let i = 0; i < 8; i++) {
            let cur = i === 7 ? queryIP.length : queryIP.indexOf(":", last + 1);
            if (cur - last - 1 < 1 || cur - last - 1 > 4) {
                return "Neither";
            }
            for (let j = last + 1; j < cur; j++) {
                const temp = queryIP[j];
                if (!isDigit(temp) && !('a' <= queryIP[j].toLowerCase() && queryIP[j].toLowerCase() <= 'f')) {
                    return "Neither";
                }
            }
            last = cur;
        }
        return "IPv6";
    }
};
// 判断字符是否为数字
const isDigit = (c) => {
    return parseInt(c).toString() !== "NaN";
}

```
