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
+ 根据kernel矩阵重新计算图像中每个像素的值，*h(k, j)* 为kernel，使用`filter2D`进行卷积操作
    ![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220609163143.png)
    ```
    CV_EXPORTS_W void filter2D( InputArray src, 
                                OutputArray dst, 
                                int ddepth, 
                                InputArray kernel, 
                                Point anchor = Point(-1,-1), 
                                double delta = 0, 
                                int borderType = BORDER_DEFAULT );
    ```
+ 使用图像卷积进行图像模糊（平滑）
    1. 简单的图像平滑，Kernel模型如下  
    ![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220609163417.png)
    2. 高斯模糊，根据距当前像素点的距离决定平滑的权重，一维高斯核的图像如下
    ![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220609163925.png)  
    二维高斯公式，其中μ为平均值，σ为方差
    ![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220609164447.png)
    3. 中值滤波，将像素点周围区域内的像素进行排序，采用中间位置的像素值作为当前像素点的值
    4. 双边滤波，权重由两部分来决定，第一部分是类似于二维高斯，另一部分由颜色差异来决定，这样可以比较好的保留边缘信息。

+ 使用图像卷积进行形态学操作，主要针对阈值化后的图像
    1. 膨胀 Dilate，使用Kernel范围内的最大值取代当前像素值  
    膨胀二值化图像  
    ![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220610150255.png)  
    膨胀灰度图像  
    ![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220610150405.png)
    1. 腐蚀 Erode，使用Kernel范围内的最小值取代当前像素值  
    腐蚀二值化图像  
    ![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220610150527.png)  
    腐蚀灰度图像  
    ![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220610150618.png)
    2. 开操作 Opening，先腐蚀后膨胀，去除暗黑背景中细小的噪点，原理 *dst = open(src, element) = dilate(erode(src, element))*， *element*为kernel
    3. 闭操作 Closing，先膨胀后腐蚀，去除明亮背景中的细小噪点，原理 *dst = close(src, element) = erode(dilate(src, element))*
    4. 形态梯度 Morphological Gradient，膨胀图像与腐蚀图像的差，用来找出物体轮廓，原理 *dst = morphy(src, element) = dilate(src, element) - erode(src, element)*
    5. 高帽 Top Hat， 


#### 图像融合
+ 图像融合公式`dst = src1*alpha + src2*beta + gamma`，使用`addWeighted()`进行融合。
    ```
    void cv::addWeighted( InputArray src1,
                          double alpha,
                          InputArray src2,
                          double beta,
                          double gamma,
                          OutputArray dst,
                          int dtype = -1 );	
    ```

#### 改变图像对比度和亮度
+ 公式`g(x)=αf(x)+β`改变α的值可以改变图像的对比度，改变β的值可以改变图像的亮度
+ Gamma校准，使用查找表，按照如下公式对像素值进行一个非线性的转换 *O = (I / 255)^γ × 255*，γ越大整体亮度降低。
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220609161302.png)
