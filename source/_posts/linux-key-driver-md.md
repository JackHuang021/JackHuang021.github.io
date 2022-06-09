---
title: Linux Input子系统按键连按驱动调试记录
author: Jack
abbrlink: e19e1ecc
date: 2022-06-09 14:10:48
tags:
 - Linux Driver 按键驱动
categories:
 - Linux
---

#### Linux Input子系统介绍
> 按键、鼠标、键盘、触摸屏等都属于输入(input)设备, Linux 内核为此专门做了一个叫做 input子系统的框架来处理输入事件。输入设备本质上还是字符设备,只是在此基础上套上了 input 框架,用户只需要负责上报输入事件,比如按键值、坐标等信息, input 核心层负责处理这些事件。

![](https://cdn.jsdelivr.net/gh/JackHuang021/images@master/images20220609141734.png)
<!-- more -->

#### input驱动编写流程
+ 注册input_dev
    1. 使用*input_allocate_device*申请一个*input_dev*结构体
    2. 初始化*input_dev*的事件类型以及事件值
    3. 使用*input_register_device*函数向系统注册*input_dev*
    4. **按键需要实现连按时*evbit*需要设置*EV_REP*标志，调按键驱动时在调连按功能时在这里卡了很久**
    ```
    chip->input = input_allocate_device();
    if (!chip->input) 
    {
        printk(KERN_ERR "Unable to allocate the input device !!\n");
        return -ENOMEM;
    }
    chip->input->name = "pca953x_button";
    set_bit(EV_KEY, chip->input->evbit);     /* 设置产生按键事件 */ 
    set_bit(EV_REP, chip->input->evbit);     /* 设置重复事件  */ 
    for(i = 0;i < ZKEY_NUM + 5; i++)	
    {
        set_bit(button_info[i].code, chip->input->keybit);	//支持具体按键键码
    }
    status = input_register_device(chip->input);   //注册input设备
    if(status)
    {
        printk(KERN_ERR "input_register_device\n");
        return -ENOMEM;
    }
    ```
#### 上报输入事件
+ 使用*input_event*函数上报指定的事件及对应的值，函数原型如下
    ```
    void input_event(struct input_dev *dev,
                     unsigned int type,
                     unsigned int code,
                     int value)
    ```
    dev:需要上报的 input_dev。  
    type: 上报的事件类型,比如 EV_KEY。  
    code:事件码,也就是我们注册的按键值,比如 KEY_0、KEY_1 等等。  
    value:事件值,比如 1 表示按键按下,0 表示按键松开。

+ 上报按键事件，Linux内核也提供了具体的上报函数*input_report_key*
    ```
    static inline void input_report_key(struct input_dev *dev, unsigned int code, int value)
    {
        input_event(dev, EV_KEY, code, !!value);
    }
    ```
    type:事件类型,比如 EV_KEY,表示此次事件为按键事件,此成员变量为 16 位。  
    code:事件码,比如在 EV_KEY 事件中 code 就表示具体的按键码,如:KEY_0、KEY_1等等这些按键。此成员变量为 16 位。  
    value:值,比如 EV_KEY 事件中 value 就是按键值,表示按键有没有被按下,如果为1的话说明按键按下,如果为0的话说明按键没有被按下或者按键松开了。

