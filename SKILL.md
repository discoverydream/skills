---
name: "customer-flow-tracker"
description: "店铺客流统计系统开发技能。基于TensorFlow.js实现实时人流检测、追踪、区域统计。Invoke when user wants to develop a customer flow/people counting system or pedestrian tracking application."
---

# 店铺客流统计系统开发技能

本技能提供基于TensorFlow.js的店铺客流统计系统完整开发方案。

## 技能功能

1. **需求分析与方案设计** - 使用超级分析框架进行需求分析
2. **前端实现** - HTML/CSS/JS实现实时视频流处理
3. **人体检测** - TensorFlow.js COCO-SSD模型集成
4. **人员追踪** - IoU匹配算法实现轨迹追踪
5. **区域统计** - 支持绊线和区域进出统计
6. **数据分析** - 时段分析、驻留时长统计

## 项目结构

```
customer-flow-tracker/
├── index.html          # 主页面
├── css/
│   └── style.css       # 样式文件
└── js/
    ├── config.js       # 配置文件
    ├── detector.js     # 人体检测模块
    ├── tracker.js      # 人员追踪模块
    ├── zoneManager.js  # 区域管理模块
    ├── analytics.js    # 时段统计分析
    ├── dwellTime.js    # 驻留时长分析
    ├── heatmap.js      # 热力图模块
    ├── recorder.js     # 录像模块
    ├── ui.js           # UI管理
    └── app.js          # 主应用
```

## 核心配置 (config.js)

```javascript
const CONFIG = {
    detection: {
        modelType: 'lite_mobilenet_v2',
        scoreThreshold: 0.5,
        maxDetections: 20,
        interval: 2000
    },
    tracking: {
        maxDistance: 150,
        maxHistory: 50,
        maxLostFrames: 30
    },
    zones: {
        enabled: true,
        list: []
    },
    analytics: {
        timeSlots: {
            granularity: 'hour',
            peakHours: [11, 12, 13, 17, 18, 19, 20]
        }
    }
};
```

## 核心模块说明

### 1. detector.js - 人体检测
- 使用TensorFlow.js COCO-SSD模型
- 提供getCenter()和getBottomCenter()方法
- 支持getDistance()计算框距离

### 2. tracker.js - 人员追踪
- 基于IoU匹配的人员追踪
- 支持跨线检测和区域检测
- 集成zoneManager进行多区域统计
- 集成flowAnalytics进行时段分析
- 集成dwellTimeAnalyzer进行驻留分析

### 3. zoneManager.js - 区域管理
- 支持绊线(line)和区域(area)两种类型
- 提供鼠标/触摸绘制接口
- 线段相交检测算法
- 穿越方向判断

### 4. analytics.js - 时段分析
- 按小时/天统计客流
- 高峰时段分析
- 数据导出(JSON/CSV)

### 5. dwellTime.js - 驻留分析
- 驻留时长分布统计
- 支持多个区间: 0-30秒、30秒-2分钟、2-5分钟等
- 异常驻留检测

## 使用场景

- 店铺选址时评估客流量
- 店铺转让时提供数据支撑
- 日常经营客流监控
- 商场铺位对比分析

## 依赖库

- TensorFlow.js: https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.17.0
- COCO-SSD: https://cdn.jsdelivr.net/npm/@tensorflow-models/coco-ssd@2.2.3

## 开发建议

1. 先实现基础的人体检测和追踪
2. 添加跨线/区域检测功能
3. 集成时段分析和驻留统计
4. 添加数据可视化和导出功能
