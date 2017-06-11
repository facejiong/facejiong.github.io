---
title: TensorFlow入门
date: 2017-06-10 23:01:39
tags: TensorFlow
categories: 机器学习
---

## 一个使用梯度下降实现线性回归的例子
```
import tensorflow as tf
import numpy as np

# 使用NumPy生成100个随机数, y = x * 0.1 + 0.3
x_data = np.random.rand(100).astype(np.float32)
y_data = x_data * 0.1 + 0.3

# 目的是求出方程y_data = W * x_data + b的W和b的值
# 声明变量W和b
W = tf.Variable(tf.random_uniform([1], -1.0, 1.0))
b = tf.Variable(tf.zeros([1]))
y = W * x_data + b

# 求最小均方差
# reduce_mean 求平均值
# square 求平方
# 梯度下降优化
loss = tf.reduce_mean(tf.square(y - y_data))
optimizer = tf.train.GradientDescentOptimizer(0.5)
# 使用梯度下降算法求loss最小值
train = optimizer.minimize(loss)

# 对所有变量进行初始化
init = tf.global_variables_initializer()

# 启动graph，执行阶段
sess = tf.Session()
sess.run(init)

# 拟合直线
for step in range(501):
    sess.run(train)
    if step % 20 == 0:
        print(step, sess.run(W), sess.run(b))

# 最佳结果是W: [0.1], b: [0.3]

# 结束后关闭Session
sess.close()
```
终端输出：
```
0 [ 0.86394864] [-0.13441049]
20 [ 0.34454051] [ 0.1738376]
40 [ 0.17663138] [ 0.26046464]
60 [ 0.12401388] [ 0.28761086]
80 [ 0.10752521] [ 0.29611763]
100 [ 0.10235816] [ 0.29878339]
120 [ 0.10073899] [ 0.29961875]
140 [ 0.10023157] [ 0.29988053]
160 [ 0.10007257] [ 0.29996258]
180 [ 0.10002275] [ 0.29998827]
200 [ 0.10000715] [ 0.29999632]
```
## TensorFlow的几个概念
Session：用于执行graph的上下文环境
Graph：计算任务，由许多节点组成
节点：代表不同的操作，被分配到cpu或gpu执行
Variable：变量

TensorFlow分两个阶段，构建阶段和执行阶段
## 梯度下降
先随机对W赋值，然后改变W的值，使loss按梯度下降的方向进行减少
{% katex [displayMode] %}



{% endkatex %}
