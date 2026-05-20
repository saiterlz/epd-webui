# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

墨水屏控制前端网页应用，通过 Web Bluetooth API 与 ESP32/nRF5 墨水屏设备通信。支持图片绘制、抖动处理、假期同步、课表/待办/名片编辑、WiFi二维码生成等功能。

## 使用方式

直接在浏览器中打开 `index.html` 即可（需通过 HTTP 服务器访问，非 file:// 协议）。推荐使用 Chrome/Edge 浏览器。

```bash
# 方式1: Python 简易服务器
python -m http.server 8080

# 方式2: Node.js
npx serve .
```

## 模块架构

```
index.html          # 主页面，CSS 加载，模块脚本加载顺序很重要
├── js/pcfFont.js   # PCF字体加载器（文泉驿点阵字）
├── js/qrcode.min.js # QR码生成库
├── js/dithering.js # 图像抖动处理（VIP版，多算法支持）
├── js/ble_transfer.js # BLE传输模块（CRC校验、断点续传）
├── js/crop.js      # 图片裁剪工具
├── js/paint.js     # PaintManager：画笔/文字/课表/待办/名片/WiFi
└── js/main.js      # 主逻辑：蓝牙连接、命令发送、EPD控制
```

### JS 模块职责

| 文件 | 职责 |
|------|------|
| `main.js` | 全局变量、蓝牙连接、命令发送（EpdCmd）、假期同步、事件绑定 |
| `ble_transfer.js` | BleTransfer 单例：CRC16校验、批传、断点续传、状态查询 |
| `paint.js` | PaintManager 类：画布工具、撤销/重做、历史记录、localStorage |
| `dithering.js` | 抖动算法（Floyd-Steinberg/Bayer/Hybrid等）、颜色匹配、对比度调整 |
| `crop.js` | CropManager 类：图片裁剪、缩放、移动 |

### BLE 命令协议（EpdCmd）

| 命令 | 功能 | 数据格式 |
|------|------|---------|
| 0x00 | 设置GPIO | `[0x00, mosi, sclk, cs, dc, rst, busy]` |
| 0x01 | 初始化 | `[0x01, modelId]` |
| 0x02 | 清屏 | `[0x02]` |
| 0x05 | 刷新 | `[0x05]` |
| 0x06 | 深度睡眠 | `[0x06]` |
| 0x20 | 设置时间 | timestamp(4B) + timezone + mode |
| 0x30/0x31 | 写图片 | 普通/CRC模式 |
| 0x32 | 查询状态 | 返回传输 bitmap |
| 0x33 | 重置传输 | sessionId |
| 0xB6 | 设置假期 | year + count + [flag<<12\|m<<8\|d,...] |
| 0xB7/B8 | 残影消除 | 开始/停止 |

Service UUID: `62750001-d828-918d-fb46-b6c11c675aec`
Characteristic UUID: `62750002-d828-918d-fb46-b6c11c675aec`

### 图片格式

200x200像素，每像素2bit，每字节存储4个像素，总大小10000字节。
颜色编码：白=0x55, 黄=0xAA, 红=0xFF, 黑=0x00。

### 假期数据格式

`holiday-cn/YYYY.json` 包含年份和日期数组，每条记录 `{date: "YYYY-MM-DD", isOffDay: bool}`。
isOffDay=true 表示休息日（红色），false 表示调休上班日（蓝色）。

## 关键实现细节

### 传输流程（main.js sendimg）
1. 检测画布模式（普通图片/课表/待办/名片/WiFi）
2. 特殊内容调用 `paintManager.redrawAll()` 禁用抖动
3. 普通图片执行 `convertDithering()` 抖动处理
4. 选择传输方式：固件>=0x20 用 CRC 校验（`writeImageCRC`），否则用普通传输（`writeImage`）
5. 四色屏先发彩色层再发黑白层；三色屏先发黑白层再发红色层
6. 发送 REFRESH 命令刷新屏幕

### 抖动算法选择（dithering.js）
- `hybrid`: 默认推荐，平衡质量和性能
- `floydSteinberg`: 经典算法，质量高但较慢
- `atkinson`: Apple 使用，较亮
- `bayer`: 规则抖动，速度快

### PCF字体
字体文件位于 `font/wenquanyi_*.pcf`，用于课表、待办等固定宽度渲染。
大小：9/10/11/12pt，b=粗体。

## 屏幕驱动支持

通过 `epddriver` 下拉框选择，支持多种尺寸和颜色模式：

| ID | 型号 | 颜色 |
|----|------|------|
| 0x01 | DEPG0150 | 黑白 |
| 0x02 | SSD1619 | 黑白 |
| 0x03 | UC8159 | 三色 |
| 0x04 | JD79660 | 四色 |
| ... | (共20+种) | ... |

画布尺寸（canvasSizes）与驱动匹配检查在 `sendimg()` 中进行。