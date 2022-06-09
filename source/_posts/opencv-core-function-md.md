---
title: OpenCV core模块记录
author: jack
tags:
  - OpenCV Core
categories:
  - OpenCV
abbrlink: 6465c2cc
date: 2022-06-08 15:24:15
---

#### 基于Mat类的图像操作
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220608085010.png)
+ Mat类分为两个数据部分：数据头部分（保存矩阵大小、矩阵存储方式等信息）、矩阵数据部分， 一般的Mat对象复制构造仅拷贝数据头部分，矩阵数据部分共享。也可以通过`cv::Mat::copyTo`和`cv::Mat::clone`进行深度拷贝。
<!-- more -->
+ Mat构造函数`Mat img(2, 2, CV_8UC3, Scalar(0, 0, 255))`， 需要明确矩阵大小、矩阵存储数据类型、像素颜色通道数、像素值。
+ CV_8UC3含义表示8位无符号数据，颜色通道为3，含义参考：`CV_[The number of bits per item][Signed or Unsigned][Type Prefix]C[The channel number]`
+ 几种特殊矩阵的构造，`cv::Mat::eyes`单位矩阵、`cv::Mat::zeros`零矩阵、`cv::Mat::ones`全1矩阵

#### 图像卷积操作
+ 根据kernel矩阵重新计算图像中每个像素的值，使用`filter2D`进行卷积操作
    ```
    CV_EXPORTS_W void filter2D( InputArray src, 
                                OutputArray dst, 
                                int ddepth, 
                                InputArray kernel, 
                                Point anchor = Point(-1,-1), 
                                double delta = 0, 
                                int borderType = BORDER_DEFAULT );
    ```
+ 两张图像融合`dst = src1*alpha + src2*beta + gamma`，使用`addWeighted()`进行融合。
    ```
    void cv::addWeighted( InputArray src1,
                          double alpha,
                          InputArray src2,
                          double beta,
                          double gamma,
                          OutputArray dst,
                          int dtype = -1 );	
    ```

+ 改变图像对比度和亮度`g(x)=αf(x)+β`改变α的值可以改变图像的对比度，改变β的值可以改变图像的亮度
