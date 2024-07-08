---
layout: '[post]'
title: LeetCode724
date: 2024-07-08 19:15:29
tags: [算法, 数组, 前缀和]
categories: LeetCode
index_img: /images/leetcode.jpeg
---
给你一个整数数组 nums ，请计算数组的 中心下标 。

数组 中心下标 是数组的一个下标，其左侧所有元素相加的和等于右侧所有元素相加的和。

如果中心下标位于数组最左端，那么左侧数之和视为 0 ，因为在下标的左侧不存在元素。这一点对于中心下标位于数组最右端同样适用。

如果数组有多个中心下标，应该返回 最靠近左边 的那一个。如果数组不存在中心下标，返回 -1 。

示例 1：

输入：nums = [1, 7, 3, 6, 5, 6]
输出：3
解释：
中心下标是 3 。
左侧数之和 sum = nums[0] + nums[1] + nums[2] = 1 + 7 + 3 = 11 ，
右侧数之和 sum = nums[4] + nums[5] = 5 + 6 = 11 ，二者相等。
示例 2：

输入：nums = [1, 2, 3]
输出：-1
解释：
数组中不存在满足此条件的中心下标。
示例 3：

输入：nums = [2, 1, -1]
输出：0
解释：
中心下标是 0 。
左侧数之和 sum = 0 ，（下标 0 左侧不存在元素），
右侧数之和 sum = nums[1] + nums[2] = 1 + -1 = 0 。
提示：

1 <= nums.length <= 104
-1000 <= nums[i] <= 1000
注意：本题与主站 1991 题相同：https://leetcode-cn.com/problems/find-the-middle-index-in-array/

Related Topics
数组
前缀和

👍 644
👎 0
```js
//leetcode submit region begin(Prohibit modification and deletion)
/**
 * 找到數組中的中心索引
 * 中心索引是指數組中間的索引,左側元素之和等於右側元素之和
 * @param {number[]} nums - 輸入的數組
 * @returns {number} - 如果存在中心索引,返回其索引值;否則返回-1
 */
var pivotIndex = function (nums) {
  // 初始化左側和為0
  let left = 0;
  // 初始化右側和為數組元素之和減去第一個元素
  let right = nums.reduce((a, b) => a + b, 0) - nums[0];
  // 如果左側和等於右側和,返回0(第一個索引)
  if (left === right) return 0;
  // 遍歷數組,從第二個元素開始
  for (let i = 1; i < nums.length; i++) {
    // 更新左側和
    left += nums[i - 1];
    // 更新右側和
    right -= nums[i];
    // 如果左側和等於右側和,返回當前索引
    if (left === right) return i;
  }
  // 如果沒有找到中心索引,返回-1
  return -1;
};
//leetcode submit region end(Prohibit modification and deletion)

```