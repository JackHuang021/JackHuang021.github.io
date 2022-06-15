---
title: Qt中串口数据处理机制
author: Jack
tags:
  - Qt QSerialPort 串口
categories:
  - Qt
abbrlink: 6e893f38
date: 2022-06-15 10:02:55
---

在嵌入式项目或者工控项目中经常会用到串口通讯，用到串口通讯可能就涉及到串口私有协议（类似 包头 + 帧类型 + 帧长度 + 帧数据 + 校验和 的形式）的解析。在Qt中经常用到`QSerialPort`类来进行串口数据收发，`QSerialPort`在串口数据可读时会释放`readyRead()`信号，接到这个信号再调用`readAll()`将缓冲区的数据全部读出来（串口数据量比较大，这个过程一般都是在一个独立的接收线程中进行处理）。<!-- More -->但是这个`readyRead()`信号释放时，缓冲区的数据长度是不固定的（一般是几十个字节会释放一次读信号），不过串口私有协议各个帧的长度基本都是固定的，这就导致处理串口数据时需要对`readAll()`中读出来的数据进行链接，然后进行数据解析。  
记录一下最近调试Qt串口通讯时用到的方法，这个方法也适用于任何系统的串口数据接收，可以达到不错的处理速度，保证不丢帧的接收。该方法的处理机制比较简单，在槽函数中接收到串口数据后，释放信号将读到的数据传递出去，接收信号方逐字节的按照协议对这些数据进行处理，这样只要`QSerialPort`将串口上的数据都完整的读出来了，应该是不会出现丢帧的情况。

#### 串口初始化及数据接收
+ 串口类进行初始化后，转移到`thread`线程中进行处理，在`readSerialData()`中将数据读取后将数据传递出去。 
    serialport.h
    ```
    class SerialPort : public QObject
    {
        Q_OBJECT
    public:
        explicit SerialPort(QObject *parent = nullptr);
        ~SerialPort();

    public slots:
        void startThread(const QString &port);
        void closeSerialPort();
        void writeSerialData(const QByteArray &data);

    private slots:
        void openSerialPort();
        void readSerialData();

    signals:
        void serialStateChanged(bool state);
        void dataReaded(const QByteArray &data);

    private:
        QString port;
        qint32 baudrate;
        QSerialPort *serial;
        QThread *thread;
    };

    ```
    serialport.cpp
    ```
    #include "serialport.h"

    /**
    * @brief SerialPort::SerialPort
    * @param parent
    */
    SerialPort::SerialPort(QObject *parent) : QObject(parent)
    {
        serial = new QSerialPort;
        thread = new QThread;
        this->moveToThread(thread);
        serial->moveToThread(thread);
        connect(thread, &QThread::started, this, &SerialPort::openSerialPort);
    }

    /**
    * @brief SerialPort::~SerialPort
    */
    SerialPort::~SerialPort()
    {
        if (serial->isOpen())
        {
            serial->close();
        }
        serial->deleteLater();
        thread->quit();
        thread->wait();
        thread->deleteLater();
    }

    /**
    * @brief SerialPort::startThread
    * @param port serial port name(QSerialPortInfo::name)
    */
    void SerialPort::startThread(const QString &port)
    {
        this->port = port;
        qDebug() << "current port is " << port;
        if (!thread->isRunning())
        {
            thread->start();
            qDebug() << "serialport thread start: " << QThread::currentThread();
        }
    }

    /**
    * @brief SerialPort::openSerialPort
    */
    void SerialPort::openSerialPort()
    {
        if (serial->isOpen())
        {
            qDebug() << "serial is opened, return.";
            return;
        }
        serial->setPortName(port);
        serial->setBaudRate(115200);
        serial->setDataBits(QSerialPort::Data8);
        serial->setStopBits(QSerialPort::OneStop);
        serial->setParity(QSerialPort::NoParity);
        serial->setFlowControl(QSerialPort::NoFlowControl);
        if (serial->open(QIODevice::ReadWrite))
        {
            qDebug() << "serial open success." << QThread::currentThread();
            connect(serial, &QSerialPort::readyRead,
                    this, &SerialPort::readSerialData, Qt::QueuedConnection);
        }
        else
        {
            qDebug() << "serial open failed.";
        }
    }

    /**
    * @brief SerialPort::readSerialData
    * read 16 bytes each time
    */
    void SerialPort::readSerialData()
    {
        QByteArray buf = serial->readAll();
        emit dataReaded(buf);
    }

    /**
    * @brief SerialPort::writeSerialData
    * @param data  buff to write
    */
    void SerialPort::writeSerialData(const QByteArray &data)
    {
        serial->write(data);
    }

    /**
    * @brief SerialPort::closeSerialPort
    */
    void SerialPort::closeSerialPort()
    {
        qDebug() << "close serialport.";
        serial->clear();
        serial->close();
        if (thread->isRunning())
        {
            thread->quit();
            thread->wait();
        }
    }
    ```

#### 串口数据处理
+ 数据处理类接收到串口类传递的数据后，按照协议逐字节的进行处理，帧头的处理比较麻烦，因为帧头一般是两个字节，可能被缓冲到前后两个数据缓冲区，所以需要对前后两个帧头的位置关系进行判断。
```
void uartRecvProcess(const QByteArray& data)
{
    int index = 0;
    static int firstHeadPos = -1;
    static int step = 0;
    static int frameLen = 0;
    static int frameDataIndex = 0;
    static int lastLen = 0;
    static uint8_t frameData[MAX_FRAME_LEN] = {0};

#ifdef PRINT_UART_DEBUG
    for (int i = 0; i < data.size(); i++)
    {
        printf("0x%02x ", (uint8_t)data.at(i));
    }
    printf("\n");
#endif

    while (index < data.size())
    {
        uint8_t temp = data.at(index);

        switch (step)
        {
        // 查找第一个帧头
        case 0:
            if (temp == 0x55)
            {
                step = 1;
                firstHeadPos = index;
                frameData[frameDataIndex++] = temp;
                lastLen = data.size();
            }
            break;

        // 查找第二个帧头
        case 1:
            if ((temp == 0xAA) &&
                ((index == firstHeadPos + 1) ||
                ((firstHeadPos == lastLen -1) && index == 0)))
            {
                step = 2;
                frameData[frameDataIndex++] = temp;
            }
            else
            {
                step = 0;
                firstHeadPos = -1;
                frameDataIndex = 0;
            }
            break;

        // 帧类型判断
        case 2:
            if(temp == CMD_AD)
            {
                frameLen = FRAME_SIZE_AD;
                frameData[frameDataIndex++] = temp;
                step = 3;
            }
            else if(temp == CMD_IO)
            {
                frameLen = FRAME_SIZE_IO;
                step = 3;
                frameData[frameDataIndex++] = temp;
            }
            else
            {
                //去掉帧头，重新查找
                printf("unknow uart frame type\n");
                step = 0;
                frameLen = 0;
                firstHeadPos = -1;
                frameDataIndex = 0;
            }
            break;

        // 帧数据接收及解析
        case 3:
            if (frameDataIndex < frameLen)
                frameData[frameDataIndex++] = temp;

            if (frameDataIndex == frameLen)
            {
                if(frameData[frameLen - 1] != checkSum(frameData, frameLen - 1))
                {
                    printf("type: %d, checksum error! received is 0x%02x, calculated is 0x%02x\n",
                           frameLen, frameData[frameLen - 1], checkSum(frameData, frameLen - 1));
                    step = 0;
                    frameLen = 0;
                    firstHeadPos = -1;
                    frameDataIndex = 0;
                }
                else
                {
                    printf("type: %d, uart data handler\n", frameLen);
                    uartDataHandler(frameData, frameLen);
                    step = 0;
                    frameLen = 0;
                    firstHeadPos = -1;
                    frameDataIndex = 0;
                }
            }
            break;

        default:
            break;
        }

        index ++;
    }
}
```

