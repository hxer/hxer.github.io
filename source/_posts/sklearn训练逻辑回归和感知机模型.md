---
title: sklearn训练逻辑回归和感知机模型
date: 2017-04-12 13:21:12
categories: 机器学习
tags:
    - 感知机
    - 逻辑回归
    - sklearn
---

调用sklearn内置的逻辑回归和感知机训练鸢尾花（Iris）数据集。
<!-- more -->

```python
# -*- coding: utf-8 -*-
# author: janes
# date: 2017年4月11日 18:30

from sklearn import datasets
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import Perceptron
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score
import matplotlib.pyplot as plt
from matplotlib.colors import ListedColormap
import numpy as np


def plot_decision_regions(X, y, classifier, test_idx=None, resolution=0.02):
    # setup marker generator and color map
    markers = ('s', 'x', 'o', '^', 'v')
    colors = ('red', 'blue', 'lightgreen', 'gray', 'cyan')
    cmap = ListedColormap(colors[:len(np.unique(y))])
    # plot the decision surface
    x1_min, x1_max = X[:, 0].min() - 1, X[:, 0].max() + 1
    x2_min, x2_max = X[:, 1].min() - 1, X[:, 1].max() + 1
    xx1, xx2 = np.meshgrid(np.arange(x1_min, x1_max, resolution),
                           np.arange(x2_min, x2_max, resolution))
    Z = classifier.predict(np.array([xx1.ravel(), xx2.ravel()]).T)
    Z = Z.reshape(xx1.shape)
    plt.contourf(xx1, xx2, Z, alpha=0.4, cmap=cmap)
    plt.xlim(xx1.min(), xx1.max())
    plt.ylim(xx2.min(), xx2.max())
    # plot class samples
    for idx, cl in enumerate(np.unique(y)):
        plt.scatter(x=X[y == cl, 0], y=X[y == cl, 1], alpha=0.8, c=cmap(idx),
                    marker=markers[idx], label=cl)
    # highlight test samples
    if test_idx:
        X_test, y_test = X[test_idx, :], y[test_idx]
        plt.scatter(X_test[:, 0], X_test[:, 1], c='', alpha=1.0, linewidth=1,
                    marker='o', s=55, label='test set')


def test_perceptron():
    iris = datasets.load_iris()
    # choose petal length and petal width as feature
    X = iris.data[:, [2, 3]]
    y = iris.target
    # 随机拿出数据集中30%的部分做测试,
    # 设置random_state(not None), 相当于设置随机数种子，每次运行随机抽样的结果相同
    X_train, X_test, y_train, y_test = train_test_split(X, y,
                                                        test_size=0.3,
                                                        random_state=None)

    # 为了追求机器学习和最优化算法的最佳性能，进行特征缩放
    sc = StandardScaler()
    sc.fit(X_train)
    X_train_std = sc.transform(X_train)
    # 用"同样的参数"来标准化测试集，使得测试集和训练集之间有可比性
    X_test_std = sc.transform(X_test)

    # n_iter：可以理解成梯度下降中迭代的次数
    # eta0：可以理解成梯度下降中的学习率
    # random_state：设置随机种子(not None)，为了每次迭代都有相同的训练集顺序
    ppn = Perceptron(n_iter=100, eta0=0.05, random_state=None)
    ppn.fit(X_train_std, y_train)

    # 分类测试集，将返回一个测试结果的数组
    y_pred = ppn.predict(X_test_std)

    # plt.scatter(np.arange(0, len(y_pred)), y_pred, c='b', marker='|',)
    # plt.scatter(np.arange(0, len(y_test)), y_test, c='r', marker='_',)
    # plt.show()

    # 计算模型在测试集上的准确性
    score = accuracy_score(y_test, y_pred)
    return score


def test_logistic():
    iris = datasets.load_iris()
    # choose petal length and petal width as feature
    X = iris.data[:, [2, 3]]
    y = iris.target
    # 随机拿出数据集中30%的部分做测试,
    # 设置random_state(not None), 相当于设置随机数种子，每次运行随机抽样的结果相同
    X_train, X_test, y_train, y_test = train_test_split(X, y,
                                                        test_size=0.3,
                                                        random_state=None)

    # 为了追求机器学习和最优化算法的最佳性能，进行特征缩放
    sc = StandardScaler()
    sc.fit(X_train)
    X_train_std = sc.transform(X_train)
    # 用"同样的参数"来标准化测试集，使得测试集和训练集之间有可比性
    X_test_std = sc.transform(X_test)

    lr = LogisticRegression(C=1000.0, random_state=0)
    lr.fit(X_train_std, y_train)
    # 查看第一个测试样本属于各个类别的概率
    # predict_proba返回的是一个两列的矩阵，矩阵的每一行代表的是对一个事件的预测结果，
    # 第一列代表该事件不会发生的概率，第二列代表的是该事件会发生的概率
    lr.predict_proba(X_test_std[0, :].reshape(1, -1))

    # vstack(): Stack arrays in sequence vertically(row wise).
    X_combined_std = np.vstack((X_train_std, X_test_std))
    y_combined = np.hstack((y_train, y_test))
    plot_decision_regions(X_combined_std, y_combined, classifier=lr,
                          test_idx=range(105, 150))
    plt.xlabel('petal length [standardized]')
    plt.ylabel('petal width [standardized]')
    plt.legend(loc='upper left')
    plt.show()


if __name__ == '__main__':
    # for _ in range(30):
    #     print(test_perceptron())
    test_logistic()
```
