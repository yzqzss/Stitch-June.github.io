---
title: 消息中间件Kafaka - PHP操作使用Kafka
tags: []
id: '19'
categories:
  - - uncategorized
date: 2019-10-27 15:31:57
---

*   PHP使用Kafka
    *   安装libkafka
    *   安装rdkafka
    *   php操作kafka

> **我们需要安装libkafka和rdkafka**

1.  **下载**
    
    去GitHub上克隆下来
    
    `git clone https://github.com/edenhill/librdkafka.git`
    
2.  **安装**
    
    `cd librdkafka/`
    
    `./configure && make && make install`
    
    安装成功界面 没有报错就是安装成功
    
    ![](http://blog.gaobinzhan.com/uploads/article/20190320/b15c81e5bde9960bfef7b84cbe3cad2d.png)
    

1.  **下载**
    
    `git clone https://github.com/arnaud-lb/php-rdkafka`
    
    `cd php-rdkafka/`
    
2.  **为php安装扩展**
    
    在php-rdkafka这个目录下
    
    `phpize`
    
    然后会生成源代码安装的脚本
    
    把php-config的位置改成自己php-config的位置
    
    `./configure --with-php-config=/usr/local/php/bin/php-config`
    
    编译安装
    
    `make && make install`
    
    成功后会出现一个文件夹
    
    ![](http://blog.gaobinzhan.com/uploads/article/20190320/fa71db3854245b8515d407e100416a84.png)
    
    这个位置就是保存的我们刚刚安装的扩展
    
    进入该目录
    
    `cd /usr/local/php/lib/php/extensions/no-debug-non-zts-20170718/`
    
    会发现出现个rdkafka.so文件  
    ![](http://blog.gaobinzhan.com/uploads/article/20190320/f97be8b3452e9a45e7445c0a956cdc04.png)
    
    修改php.ini文件加入 这里的路径就是写自己rdkafka.so文件的路径
    
    `extension=/usr/local/php/lib/php/extensions/no-debug-non-zts-20170718/rdkafka.so`
    
    重启php
    
    `php-m`
    
    出现rdkafka就是安装成功
    
    ![](http://blog.gaobinzhan.com/uploads/article/20190320/58564b0be9d3f46a3fe2d663973bfca2.png)
    

> **运行前先开启我们的zookeeper和kafka 上篇文章有如何开启**

1.  **运行producer**  
    kafka默认端口9092
    
    `vim producer.php`