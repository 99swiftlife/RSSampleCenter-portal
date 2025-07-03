<template>
    <a-row gutter={16} align="middle">
      <!-- 输入框和按钮同一行 -->
      <a-col span="18">
        <a-input v-model:value="inputValue" placeholder="请输入经纬度范围" readonly style="width: 100%" />
      </a-col>
      <a-col span="6">
        <a-button type="primary" @click="loadMap" style="width: 100%">选区</a-button>
      </a-col>
  
      <!-- 地图容器 -->
      <a-col span="24" style="position: relative;">
        <!-- 使用 v-show 来避免销毁和重建地图 -->
        <div v-show="mapVisible" ref="mapContainer" class="map-container" />
      </a-col>
    </a-row>
  </template>
  
  <script lang="ts" setup>
  import { ref, nextTick, watch } from 'vue';  // 导入 nextTick 和 watch
  import 'ol/ol.css';
  import { Map } from 'ol';
  import View from 'ol/View';
  import TileLayer from 'ol/layer/Tile';
  import { OSM } from 'ol/source';
  import { fromLonLat, toLonLat } from 'ol/proj';
  import { Polygon } from 'ol/geom';
  import { Fill, Stroke, Style } from 'ol/style';
  import { Vector as VectorSource } from 'ol/source';
  import { Vector as VectorLayer } from 'ol/layer';
  import Feature from 'ol/Feature'; // 正确导入 Feature
  import { Event as OlEvent } from 'ol/events';
  
  const inputValue = ref('');
  const mapVisible = ref(false);  // 控制地图是否可见
  
  // 地图容器的引用
  const mapContainer = ref<HTMLElement | null>(null);
  let map: Map | null = null;
  let vectorLayer: VectorLayer | null = null;
  let vectorSource: VectorSource | null = null;
  let startCoordinate: number[] | null = null;  // 矩形起点
  let rectangleFeature: Feature | null = null;  // 用于保存绘制的矩形
  
  // 加载地图
  const loadMap = () => {
    mapVisible.value = !mapVisible.value;  // 切换地图显示状态
  
    if (mapVisible.value && mapContainer.value && !map) {
      vectorSource = new VectorSource();
      vectorLayer = new VectorLayer({
        source: vectorSource,
      });
  
      map = new Map({
        target: mapContainer.value,
        layers: [
          new TileLayer({
            source: new OSM(), // OpenStreetMap 图层
          }),
          vectorLayer,
        ],
        view: new View({
          center: fromLonLat([0, 0]), // 设置地图中心为经纬度[0, 0]
          zoom: 2, // 初始缩放级别
        }),
      });
  
      // 添加地图点击事件，记录起点和终点
      map.on('click', handleMapClick);
  
      // 强制更新地图的大小
      nextTick(() => {
        map?.updateSize();
      });
    }
  };
  
  // 处理地图点击事件
  const handleMapClick = (event: OlEvent) => {
    const coordinates = event.coordinate;  // 获取点击坐标 (地图投影坐标)
    if (!startCoordinate) {
      // 第一次点击，记录起点
      startCoordinate = coordinates;
    } else {
      // 第二次点击，确定终点并绘制矩形
      drawRectangle(startCoordinate, coordinates);
      startCoordinate = null;  // 清除起点，准备下一次绘制
    }
  };
  
  // 绘制矩形
  const drawRectangle = (start: number[], end: number[]) => {
    const minX = Math.min(start[0], end[0]);
    const minY = Math.min(start[1], end[1]);
    const maxX = Math.max(start[0], end[0]);
    const maxY = Math.max(start[1], end[1]);
  
    // 使用 Polygon 表示矩形
    const polygon = new Polygon([
      [
        [minX, minY],
        [minX, maxY],
        [maxX, maxY],
        [maxX, minY],
        [minX, minY],
      ],
    ]);
  
    // 创建矩形特性并添加到地图
    if (rectangleFeature) {
      vectorSource?.removeFeature(rectangleFeature); // 移除之前的矩形
    }
  
    rectangleFeature = new Feature({
      geometry: polygon,
    });
  
    // 给矩形设置样式
    rectangleFeature.setStyle(
      new Style({
        fill: new Fill({
          color: 'rgba(255, 255, 255, 0.2)', // 半透明白色填充
        }),
        stroke: new Stroke({
          color: '#ffcc33', // 边框颜色
          width: 2, // 边框宽度
        }),
      })
    );
  
    vectorSource?.addFeature(rectangleFeature);
  
    // 更新输入框中的经纬度范围，保留两位小数
    const [minLonLat, maxLonLat] = [start, end].map((coord) => toLonLat(coord));
  
    // 使用 nextTick 来确保 DOM 更新完成后再更新 inputValue
    nextTick(() => {
      inputValue.value = `Min Lon: ${minLonLat[0].toFixed(2)}, Min Lat: ${minLonLat[1].toFixed(2)} - Max Lon: ${maxLonLat[0].toFixed(2)}, Max Lat: ${maxLonLat[1].toFixed(2)}`;
    });
  };
  </script>
  
  <style scoped>
  .map-container {
    position: absolute; /* 绝对定位 */
    top: 10;
    left: 0;
    width: 100%;  /* 填满父容器宽度 */
    height: 300px; /* 设置地图高度 */
    z-index: 10;  /* 确保地图在输入框和按钮之上 */
    background-color: white; /* 可以设置背景颜色以便于查看 */
  }
  
  .a-col {
    position: relative; /* 让地图容器能绝对定位在 a-col 内 */
  }
  </style>
  