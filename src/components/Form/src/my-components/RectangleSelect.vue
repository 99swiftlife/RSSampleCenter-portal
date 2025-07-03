<template>
  <div>
    <a-input v-model:value="selectedArea" style="width: 14em;" placeholder="ctrl+鼠标拖动选择区域" @click="showMap = !showMap"  />
    <div v-show="showMap" ref="mapContainer" class="map-container"></div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted } from 'vue';
import { Input } from 'ant-design-vue';
import Map from 'ol/Map';
import View from 'ol/View';
import TileLayer from 'ol/layer/Tile';
import OSM from 'ol/source/OSM';
import { DragBox } from 'ol/interaction';
import { platformModifierKeyOnly } from 'ol/events/condition';
import { transformExtent } from 'ol/proj';
import 'ol/ol.css';

const showMap = ref(false); // 由父组件控制
const mapContainer = ref(null);
const selectedArea = ref('');
let map = null;

import { defineEmits } from 'vue';

const emit = defineEmits(['change']);

function sendEvent() {
  console.log(selectedArea.value)
  emit('change', selectedArea.value); // 发送当前日期
}

// 初始化地图的逻辑，原本在 onMounted 中
const initMap = () => {
  console.log(mapContainer.value, map);
  if (mapContainer.value && !map) {
    console.log("初始化地图!")
    map = new Map({
      target: mapContainer.value,
      layers: [
        new TileLayer({
          source: new OSM()
        })
      ],
      view: new View({
        center: [0, 0],
        zoom: 2
      })
    });

    const dragBox = new DragBox({
      condition: platformModifierKeyOnly
    });

    map.addInteraction(dragBox);
    dragBox.on('boxend', () => {
      console.log("触发矩形绘制事件");
      const extent = dragBox.getGeometry().getExtent();
      const transformedExtent = transformExtent(extent, 'EPSG:3857', 'EPSG:4326');

      // 格式化输出，保留两位小数
      const formattedExtent = transformedExtent.map(coord => coord.toFixed(2));
      selectedArea.value = `${formattedExtent.join(', ')}`;
      showMap.value = false; // Optionally hide map after selection
      console.log("抛出Input事件！")
      sendEvent();
    });
  }
};

watch(showMap, (newVal) => {
  if (newVal) {
    initMap(); // 当 showMap 为 true 时，初始化地图
  }
}, { immediate: true });
</script>

<style>
.map-container {
  position: absolute;
  top: 100%;
  /* Just below the input box */
  left: 0;
  width: 400px;
  height: 300px;
  z-index: 1000;
  /* Ensure it floats above other content */
}
</style>
