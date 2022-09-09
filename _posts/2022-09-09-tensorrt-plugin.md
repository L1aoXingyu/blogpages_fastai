---
toc: true
layout: post
description: 如何在 TensorRT 中使用自定义的插件
comments: true
categories: [deep learning, deployment, tensorRT, inference, onnx]
title: "TensorRT 使用 Custom Plugin"
# image:
---

## 0x0 Introduction

在模型开发的流程中，除了训练模型之外，另外一个同样重要的部分就是模型部署，常见的部署方式有两种，一种是直接使用原生的训练框架做推理，另外一种是使用硬件厂商提供的加速器。
一般情况下，可以选择简单易用的第一种方式，但是在对性能有极致要求的边端设备上，就只能选择第二种方式，这个时候就需要设计到模型从训练框架到加速器的转换流程。

目前工业界比较常见的流程是通过 onnx 作为模型的中间表达(IR)，即所有的训练框架(PyTorch, TensorFlow, etc.) 在训练完成之后都转换成 onnx，然后再转换成加速器的 IR，比如 TensorRT 的 engine，虽然这个流程转换并不容易，中间也有很多坑，但是这个流程确实是一个比较通用。
