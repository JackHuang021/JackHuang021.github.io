---
title: OpenCV图像容器Mat
author: Jack
tags:
  - OpenCV Mat
categories:
  - OpenCV
abbrlink: d2c86892
date: 2022-06-08 08:46:27
---

#### 图像的存储
![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220608085010.png)  
借助官方这张很有代表性的图片，在计算机中以像素值的方式存储图像中的每个像素点，所有像素值以类似矩阵的二维数组形式进行存储。官网给OpenCV的定义：OpenCV是一个计算机视觉库，主要的目的就是在这些像素值上进行处理、计算，所以首先就需要学习OpenCV是怎么存储、处理这些图像的。
<!-- more -->

#### Mat介绍
+ OpenCV起始于2001年，最开始是基于C语言编写，图像存储在C结构体`IplImage`。
+ OpenCV2.0基于C++进行重写，图像以Mat类进行存储。
+ Mat类包含两部分数据：  
    1. 图像数据头部分，包含图像尺寸、存储方式、图像数据指针等信息
    2. 图像数据部分
+ 每个Mat对象都独立的保存数据头部分，但是可能共享图像数据部分。
+ Mat的拷贝构造只会拷贝数据头部分，数据部分不会进行拷贝。
    ```
    Mat A, C;                          // creates just the header parts
    A = imread(argv[1], IMREAD_COLOR); // here we'll know the method used (allocate matrix)
    Mat B(A);                          // Use the copy constructor
    C = A;                             // Assignment operator
    ```
+ Mat类有一个引用计数机制，进行拷贝构造时引用加1，Mat对象析构时引用减1，当引用变为0时，释放数据部分内存。
+ 当需要进行数据部分拷贝时，OpenCV提供了`cv::Mat::clone()`和`cv::Mat::copyTo`两个方法

#### Mat数据存储方式
+ 根据颜色空间和数据类型来选择图像数据存储方式

#### Mat对象操作
+ 使用`cv::Mat::Mat`构造函数进行创建`Mat img(2, 2, CV_8UC3, Scalar(0, 0, 255))`，指定图像的行和列大小、像素数据类型、像素颜色通道，如`CV_8UC3`表示8位无符号数据，颜色通道为3，由以下方式进行定义  
`CV_[The number of bits per item][Signed or Unsigned][Type Prefix]C[The channel number]`
+ 类似MATLAB方式创建单位矩阵`cv::Mat::eyes`、零矩阵`cv::Mat::zeros`、元素全为1的矩阵`cv::Mat::ones`
    ```
    Mat E = Mat::eye(4, 4, CV_64F);
    cout << "E = " << endl <+ Mat对象的打印
    1. 默认输出< " " << E << endl << endl;
    Mat O = Mat::ones(2, 2, CV_32F);
    cout << "O = " << endl << " " << O << endl << endl;
    Mat Z = Mat::zeros(3,3, CV_8UC1);
    cout << "Z = " << endl << " " << Z << endl << endl;
    ```

