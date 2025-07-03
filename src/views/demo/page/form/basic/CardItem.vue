<template>
  <div class="card" @click="showDetails">
    <!-- 删除按钮 -->
    <button class="delete-button" @click.stop="handleDelete">删除</button>
    <div class="card-images">
      <img v-for="(imageId, index) in dataset.randImageIds" :key="index"
        :src="`http://10.3.1.128:8084/category/thumbnail/${imageId}`" alt="dataset image" class="card-image" />
    </div>
    <div class="card-body">
      <h3>{{ dataset.datasetName }}</h3>
      <p>{{ dataset.description }}</p>
    </div>

    <!-- 弹出详细信息的对话框 -->
    <div v-if="isDialogVisible" class="dialog-overlay" @click="hideDetails">
      <div class="dialog-content" @click.stop>
        <h2>{{ dataset.datasetName }}</h2>
        <div class="dialog-images">
          <img v-for="(imageId, index) in dataset.randImageIds" :key="index"
            :src="`http://10.3.1.128:8084/category/thumbnail/${imageId}`" alt="dataset image" />
        </div>
        <p>{{ dataset.description }}</p>
        <button @click="hideDetails">关闭</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';

// props：从父组件传入的单个数据集信息
const props = defineProps({
  dataset: {
    type: Object,
    required: true
  },
  onDelete: Function, 
});

// 控制对话框的显示状态
const isDialogVisible = ref(false);

// 显示详细信息的对话框
const showDetails = () => {
  isDialogVisible.value = true;
};

// 隐藏对话框
const hideDetails = () => {
  isDialogVisible.value = false;
};

const handleDelete = () => {
  if (props.onDelete) {
    props.onDelete(props.dataset.datasetName);
  }
};
</script>

<style scoped>
.card {
  width: 300px;
  border: 1px solid #ccc;
  border-radius: 10px;
  overflow: hidden;
  cursor: pointer;
  margin: 10px;
  transition: box-shadow 0.3s ease;
  position: relative; /* 确保删除按钮能相对于卡片定位 */
}

.card:hover {
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.card-images {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-gap: 2px;
}

.card-image {
  width: 100%;
  height: 100px;
  object-fit: cover;
}

.card-body {
  padding: 15px;
}

.dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
}

.dialog-content {
  background: white;
  padding: 20px;
  border-radius: 10px;
  max-width: 600px;
  width: 100%;
}

.dialog-images {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-gap: 2px;
  margin-bottom: 15px;
}

.dialog-content img {
  width: 100%;
  border-radius: 5px;
}

.dialog-content button {
  margin-top: 10px;
}

.delete-button {
  position: absolute;
  right: 10px;
  top: 10px;
  padding: 5px 10px;
  background-color: #f5f5f5;
  color: #333;
  border: 1px solid #ccc;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s, color 0.3s;
  z-index: 10; /* 确保按钮浮在图片之上 */
}

.delete-button:hover {
  background-color: #ff4d4f;
  color: white;
  border-color: #ff4d4f;
}
</style>