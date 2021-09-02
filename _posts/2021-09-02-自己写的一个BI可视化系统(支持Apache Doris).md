---
layout: post
title: "自己开源的一个大数据BI可视化系统(支持Apache Doris)"
date: 2021-09-02
description: "自己开源的一个大数据BI可视化系统(支持Apache Doris)"
tag: Apache Doris , 数据可视化
---

## 介绍

这是一个可自由拖拽的BI可视化系统

后端框架使用了若依

去年疫情期间没事随手写的一个，如果你觉得好，别忘了加个星，谢谢

这个完全支持Apache doris

## 功能

1. 按项目管理数据看板
2. 看板具备分享功能
3. 可以自由拖拽实现数据看板
4. 自由拖拽实现图表开发
5. 提供数据报表开发工具
6. 提供sql开发控制台
7. 数据下钻（按维度下钻）
8. 数据源管理
9. 元数据管理
10. 用户管理

目录结构：

![img](https://pic1.zhimg.com/80/v2-75405bc31c03775108b4fccd0d3ec64c_1440w.jpg)



mobile ：手机端，手机端只是查看，不具备设计功能

ui：pc端

doc：这里是数据库脚本

## 编译

进入到前端页面目录（ui,mobile）

```text
npm install  
npm run dev  
npm run build
```

## 系统截图

## 登录界面



![img](https://pic4.zhimg.com/80/v2-55ded8811e7ffd8674ecddd4108d960f_1440w.jpg)

默认用户名密码：admin/123456

## 首页

![img](https://pic4.zhimg.com/80/v2-031eb010f6f0049a9e870367d367a28f_1440w.jpg)

图表设计

![img](https://pic3.zhimg.com/80/v2-47cc8b79dab0c07e1cac8451a0103f36_1440w.jpg)

## 数据看板

![img](https://pic2.zhimg.com/80/v2-a6884b83dca8eb800398850dc06b22b1_1440w.jpg)

## 数据下钻

![img](https://pic4.zhimg.com/80/v2-64e053b27b63ca5e45b4fb9791ec13b7_1440w.jpg)

## 数据报表设计



![img](https://pic2.zhimg.com/80/v2-4d0869627a16549462f653394881d68d_1440w.jpg)

## 元数据管理

![img](https://pic3.zhimg.com/80/v2-d2ba531da49ecc770f30188ba67d411a_1440w.jpg)



代码地址：

[GitHub - hf200012/oceanus.bigithub.com/hf200012/oceanus.bi![img](https://pic2.zhimg.com/v2-ab5bbe1db66feededad9dc9379454c85_180x120.jpg)](https://link.zhihu.com/?target=https%3A//github.com/hf200012/oceanus.bi)