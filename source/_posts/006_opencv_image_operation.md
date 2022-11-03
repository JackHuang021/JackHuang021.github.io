---
title: OpenCV图像操作
author: Jack
tags:
  - OpenCV Image
categories:
  - OpenCV
abbrlink: 117a3b0c
date: 2022-06-08 11:05:11
---

#### 图像读取和保存
+ 图像读取`Mat img = imread(filename)`
+ 图像保存`imwrite(filename, img)`
<!-- more -->

#### 像素级操作
+ 获取单通道灰度图像(x, y)位置像素值`Scalar intensity = img.at<uchar>(Point(x, y));`
+ 修改像素值`img.at<uchar>(Point(x, y)) = 128`
+ 获取3通道BGR颜色空间图像(x, y)位置像素值
    ```
    Vec3b vector = img.at<Vec3b>(Point(x, y));
    uchar blue = vector.val[0];
    uchar green = vector.val[1];
    uchar red = vector.val[2];
    ```

#### 图像基本操作
+ 选取图像某个区域
    ```
    Rect r(10, 10, 100, 100);
    Mat smallImg = img(r);
    ```
+ 颜色空间转换
    ```
    Mat img = imread("image.jpg");
    Mat gray;
    cvtColor(img, gray, COLOR_BGR2GRAY);
    ```
+ 数据类型转换`src.convertTo(dst, CV_32F)`
+ 图像显示
    ```
    Mat img = imread("image.jpg");
    namedWindow("image", WINDOW_AUTOSIZE);
    imshow("image", img);
    waitKey();
    ```
