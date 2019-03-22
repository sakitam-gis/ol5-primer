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

  首先针对 `ol` 图层和 `echarts` 图层两者叠加一般我们有两种方式：
  * 一是直接创建一个页面元素再去实例化一个 `echarts` 容器，然后通过 `ol` 地图视图变化抛出的事件去同步 `ECharts` 图表的变化。这种做法的好处是
  可以使用 `ECharts` 本身自带的 `ToolTip` 和控件等，缺点是会造成性能的损失。
  * 另外一种方案是直接使用 `ImageCanvas` 去创建内置可合并的 canvas 图层，这种方案可以在一定程度上提升渲染性能，但是会损失 `ECharts` 内置的一些交互。

下面我们主要以第一种方案梳理一下具体实现的流程。

1、首先我们需要一个能够实例化 `ECharts` 的容器，这个容器的存放位置也是有一定要求的。我们需要考虑的是首先上面叠加的 Dom 图层不能遮挡下面的图层，另外
上面 Dom 图层的事件需要穿透到下部，所以我们考虑直接将容器默认创建在 `ol-overlaycontainer` 容器内，主要代码如下：

![ol-echarts-dom](../images/ol-echarts-dom.png)

此外还需要注意，容器创建位置也支持自定义，但是默认都是在上图地图容器内；一般情况下不需要自定义位置，采用默认即可。

```ecmascript 6
_createLayerContainer (map, options) {
  const viewPort = map.getViewport()
  const container = (this.$container = document.createElement('div'));
  container.style.position = 'absolute';
  container.style.top = '0px';
  container.style.left = '0px';
  container.style.right = '0px';
  container.style.bottom = '0px';
  let _target = getTarget(options['target'], viewPort);
  if (_target && _target[0] && _target[0] instanceof Element) {
    _target[0].appendChild(container);
  } else {
    let _target = getTarget('.ol-overlaycontainer', viewPort);
    if (_target && _target[0] && _target[0] instanceof Element) {
      _target[0].appendChild(container);
    } else {
      viewPort.appendChild(container);
    }
  }
}
```

2、当然我们在实例化 `ECharts` 之前，我们需要考虑如何将 `Openlayers` 的坐标系统注册到 `ECharts` 的坐标系统上。其实参考
`BMap` 的官方集成方案 [BMapCoordSys](https://github.com/apache/incubator-echarts/blob/master/extension/bmap/BMapCoordSys.js) 
我们可以看到 `ECharts` 实际上也提供可扩展的接口，所以我们可以依照其核心思想注册一个 `Openlayers` 的坐标系统，主要源码请查看[RegisterCoordinateSystem](https://github.com/sakitam-fdd/ol3Echarts/blob/e74bb74317/packages/ol-echarts/src/coordinate/RegisterCoordinateSystem.js)。
我们在这章只去分析一些关键的点即可。

```ecmascript 6
// 注册openlayers坐标系统，_getCoordinateSystem来自 `RegisterCoordinateSystem`
echarts.registerCoordinateSystem('openlayers', _getCoordinateSystem(map, {}));

// 坐标系统注册完成后需要将传入的echarts配置项的series类型的每一项的coordinateSystem重定义为 `openlayers`, 保持和上面注册类型一致
// echarts 在绘制时会根据相应的坐标系统去将传入的空间数据转换为屏幕坐标
for (let i = series.length - 1; i >= 0; i--) {
  series[i]['coordinateSystem'] = 'openlayers';
  // series[i]['animation'] = false;
}
```
  

3、需要拿到已经创建的容器去实例化 `ECharts` 主要代码如下：

```ecmascript 6
// 创建echarts实例，并且设置配置项
this.$chart = echarts.init(this.$container);
this.$chart.setOption(options)
```

4、绑定openlayers地图的重绘事件，在`ol.view`视图变化时同步更新 `echarts` 图表内容。

```ecmascript 6
const Map = this.$Map;
const view = Map.getView();
if (this.$options.forcedPrecomposeRerender) {
  // 强制实时更新，能够避免地图和 `echarts` 视图不同步问题
  this.precomposeListener_ = Map.on('precompose', this.reRender.bind(this));
}

// openlayers地图尺寸变化事件
this.sizeChangeListener_ = Map.on('change:size', this.onResize.bind(this));
// openlayers地图分辨率变化事件
this.resolutionListener_ = view.on('change:resolution', this.onZoomEnd.bind(this));
// openlayers地图中心点变化事件
this.centerChangeListener_ = view.on('change:center', this.onCenterChange.bind(this));
// openlayers地图视图旋转角度变化事件
this.rotationListener_ = view.on('change:rotation', this.onDragRotateEnd.bind(this));
// openlayers地图视图平移开始事件
this.movestartListener_ = Map.on('movestart', this.onMoveStart.bind(this));
// openlayers地图视图平移结束事件
this.moveendListener_ = Map.on('moveend', this.onMoveEnd.bind(this));
```

5、地图事件触发echarts重绘

```ecmascript 6
// 如果强制重绘会手动调用一次 `echarts` 实例的 `clear` 方法
if (this.$options.forcedRerender) {
  this.$chart.clear();
}

// 更新 echarts 容器尺寸
this.$chart.resize();

// 重新设置配置项，会触发图表的重绘和像素坐标的重新计算。
this.$chart.setOption(options)
```

基本思路可以简单理解为：
  - 绑定 `openlayers` 地图视图重绘事件同步重绘 `echarts`视图。
  - 地理坐标到真实屏幕坐标的实时转换。
  
以上就是本章节的基本内容。
