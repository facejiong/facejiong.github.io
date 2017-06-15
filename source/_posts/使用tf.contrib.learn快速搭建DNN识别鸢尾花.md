---
title: 使用tf.contrib.learn快速搭建DNN识别鸢尾花
date: 2017-06-13 21:17:08
tags: TensorFlow
categories: 机器学习
---
tf.contrib.learn是TensorFlow提供高级API。下面是使用DNN预测鸢尾花卉数据集的例子。
先分析下鸢尾花卉数据集,0/1/2分别代表Setosa,versicolor,virginica三个种类的花

| Sepal.Length（花萼长度） | Sepal.Width（花萼宽度） | Petal.Length（花瓣长度）| Petal.Width（花瓣宽度） | 种类 |
|:--------|:---------|:-------|:-------|
| 7.9 | 3.8 | 6.4 |　2.0 | 0/1/2 |


１、载入数据
２、构造神经网络分类器
３、利用训练数据拟合模型
４、评估模型的精确性
５、新的样本分类

```
from __future__ import absolute_import
from __future__ import division
from __future__ import print_function

import tensorflow as tf
import numpy as np

IRIS_TRAINING = "iris_training.csv"
IRIS_TEST = "iris_test.csv"

# 加载数据集
training_set = tf.contrib.learn.datasets.base.load_csv_with_header(
    filename=IRIS_TRAINING,
    target_dtype=np.int,
    features_dtype=np.float32)
test_set = tf.contrib.learn.datasets.base.load_csv_with_header(
    filename=IRIS_TEST,
    target_dtype=np.int,
    features_dtype=np.float32)

print(training_set)
# 特征
feature_columns = [tf.contrib.layers.real_valued_column("", dimension=4)]

# 构造３层DNN网络，每层分别是10,20,10个节点
classifier = tf.contrib.learn.DNNClassifier(feature_columns=feature_columns,
                                            hidden_units=[10, 20, 10],
                                            n_classes=3,
                                            model_dir="/tmp/iris_model")

# 拟合模型，迭代2000步
classifier.fit(x=training_set.data,
               y=training_set.target,
               steps=2000)

# 计算精度
accuracy_score = classifier.evaluate(x=test_set.data,
                                     y=test_set.target)["accuracy"]
print('Accuracy: {0:f}'.format(accuracy_score))

# 对２个新样本预测
new_samples = np.array(
    [[6.4, 3.2, 4.5, 1.5], [5.8, 3.1, 5.0, 1.7]], dtype=float)
y = list(classifier.predict(new_samples, as_iterable=True))
print('Predictions: {}'.format(str(y)))
```
输出
```
Accuracy: 0.966667
Predictions: [1, 1]
```
