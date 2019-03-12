# 集成 ECharts

本章节主要讲解 `ol5` 与 `ECharts` 如何进行集成，将会从以下几个方面进行深入：

- [ol-echarts简介](##ol-echarts简介)
- [插件使用](##插件使用)
- [原理剖析](##原理剖析)

当然在阅读本章节之前，需要你已经了解 `ol5` 的图层、事件系统、投影和坐标变换。

## [ol-echarts](https://github.com/sakitam-fdd/ol3Echarts/blob/master/packages/ol-echarts)简介

  `ol-echarts` 是已经封装完成的 `ol5` 与 `ECharts`集成的类库，暂时已经支持了 `echarts` 的所有map组件类型，对普通不支持坐标系的图表也兼容了饼图，柱状图，折线图。
通过这个插件，你可以直接将`Echarts`集成到 `Openlayers 5`。但是除了使用外，我们还希望读者能从中获得更多的东西，触类旁通，真正的剖析原理去了解 ol5 内部的一些组成，可能会
带给你一些不一样的知识。

## 简单使用

  首先在使用前你应该已经安装了 `ol echarts ol-echarts` 这几个必要依赖，并且如果使用了 `gl` 渲染模式还需要新增 `echarts-gl` 依赖。
然后和正常流程一样，我们需要先创建一个地图：

```ecmascript 6
import 'ol/ol.css';
import { Map, View } from 'ol';
import TileLayer from 'ol/layer/Tile';
import XYZ from 'ol/source/XYZ';

const map = new Map({
  target: '#map',
  view: new View({
    center: [113.53450137499999, 34.44104525],
    projection: 'EPSG:4326',
    zoom: 5 // resolution
  }),
  layers: [
    new TileLayer({
      source: new XYZ({
        url: 'http://cache1.arcgisonline.cn/arcgis/rest/services/ChinaOnline' +
        'StreetPurplishBlue/MapServer/tile/{z}/{y}/{x}'
      })
    })
  ]
});
```

然后我们需要初始化一个 `ECharts` 图层，主体代码如下

```ecmascript 6
import EChartsLayer from 'ol-echarts';

const option = {}; // 为标准的ECharts的配置，不需要做特殊处理

const chart = new EChartsLayer(option, {
  forcedRerender: false, // 强制重绘，会调用 ECharts 的 clear() 方法清空图层
  forcedPrecomposeRerender: false, // 强制在 map 触发 precompose（准备渲染，未开始渲染）事件时进行 ECharts 图层的重绘
  hideOnZooming: false, // 在地图缩放时隐藏 ECharts 图层，这样在一些场景下可以提高性能。
  hideOnMoving: false, // 在地图移动时隐藏 ECharts 图层，这样在一些场景下可以提高性能。
  hideOnRotating: false, // 在地图旋转时隐藏 ECharts 图层，这样在一些场景下可以提高性能。
});

chart.appendTo(map); // 将 ECharts 图层添加到地图上
```

通过以上步骤，我们就能简单实现了一下 `Openlayers 5` 和 `ECharts` 结合展示散点图的示例。

## 原理剖析

### 核心原理

  其实我们在考虑这个问题时首先需要知道不管是 `ol` 还是 `echarts` 内的图形绘制和渲染在浏览器端的交互变化实际上都是在内部进行实时的重绘。
当我们理解清楚这个原理我们就可以考虑是否可以通过对 `ol` 视图和 `echarts` 的视图进行同步就能达到将 `ol` 和 `echarts` 结合起来进行展示。
而且幸运的是[echarts](https://github.com/apache/incubator-echarts/blob/master/extension/bmap/README.md) 官方已经给出了和百度地图
结合的相关代码，我们可以通过剖析相关代码来加快我们对核心原理的理解。
