---
title: js-each，map，reduce实现
date: 2017-10-16 02:29:16
tags: lodash
categories: js
---
```
// 减少一级数组嵌套
function flatten(array) {
  const length = array == null ? 0 : array.length
  return length ? baseFlatten(array, 1) : []
}
// 减少到一维
function flattenDeep(array) {
  const length = array == null ? 0 : array.length
  return length ? baseFlatten(array, INFINITY) : []
}
// 减少depth维
function flattenDepth(array, depth) {
  const length = array == null ? 0 : array.length
  if (!length) {
    return []
  }
  depth = depth === undefined ? 1 : +depth
  return baseFlatten(array, depth)
}
/**
 * @private
 * @param {Array} array 需要处理的数组
 * @param {number} depth 需要减少的维度
 * @param {boolean} [predicate=isFlattenable] 每个元素回调函数，判断是否一维，默认isFlattenable
 * @param {boolean} [isStrict] Restrict to values that pass `predicate` checks.
 * @param {Array} [result=[]] 初始结果值
 * @returns {Array} 返回结果
 */
function baseFlatten(array, depth, predicate, isStrict, result) {
  predicate || (predicate = isFlattenable)
  result || (result = [])

  if (array == null) {
    return result
  }

  for (const value of array) {
    if (depth > 0 && predicate(value)) {
      if (depth > 1) {
        // Recursively flatten arrays (susceptible to call stack limits).
        baseFlatten(value, depth - 1, predicate, isStrict, result)
        console.log(value)
      } else {
        result.push(...value)
      }
    } else if (!isStrict) {
      result[result.length] = value
    }
  }
  return result
}
```
