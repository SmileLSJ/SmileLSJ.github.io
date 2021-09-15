---
layout:     post
title:      "RocketMQ技术内幕学习-针对点-02--ServiceThread"
date:       2021-09-01 12:00:00
author:     "LSJ"
header-img: "img/post-bg-2015.jpg"
tags:
    RocketMQ
    CountDownLatch2
    ServiceThread
---



## ServiceThread

* 通过了一套通用的线程框架具有下面特点
  * 线程可以进行等待多久就之后可以继续执行
  * 还提供了唤醒线程执行的机制



## CountDownLatch2

* 和CountDownLatch区别

  ![image-20210827154432246](../../../../照片/typora/image-20210827154432246.png)