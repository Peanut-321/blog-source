---
title: "SRCNN FPGA- Accelerated"
date: 2026-10-13 00:00:00
categories:
  - Tech Note
---

## 1. SRCNN 网络结构与计算分析

## 2. Golden Reference 与数值验证

## 3. 定点化与 `ap_fixed` 位宽设计

## 4. HLS 加速器设计

### 4.1 Conv1：9×9 空间卷积

### 4.2 Conv2：1×1 通道混合

### 4.3 Conv3：5×5 重建层

### 4.4 Line Buffer 与 Sliding Window

### 4.5 Pipeline、Unroll 与 DATAFLOW

## 5. PL 端 AXI 与 DMA 接口

## 6. PS 端 PYNQ Host

## 7. PL/PS 系统集成

## 8. 正确性、性能与资源评估
