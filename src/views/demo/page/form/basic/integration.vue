<template>
  <a-card title="任务管理" :style="{ width: '100%' }">
    <!-- 创建新任务按钮 -->
    <div class="task-info">
      <a-button type="primary" @click="showCreateTaskModal">创建新任务</a-button>
      <a-divider />
      <a-row>
        <a-col :span="12">最大任务数量: <a-input-number v-model:value="maxTasks" :min="1" :max="100" /></a-col>
        <a-col :span="12">同时运行任务数: <a-input-number v-model:value="concurrentTasks" :min="1" :max="10" /></a-col>
      </a-row>
    </div>

    <!-- 任务列表 -->
    <a-list :dataSource="taskList" bordered style="margin-top: 20px">
      <template v-slot:renderItem="task">
        <a-list-item>
          <a-list-item-meta :description="task.item.status" class="task-meta">
            <template #title>
              <a href="https://www.antdv.com/">{{ task.item.datasetName }}</a>
            </template>
            <template #actions>
              <a-button :type="task.item.status === 'paused' ? 'primary' : 'default'" @click="toggleTaskStatus(task)">
                {{ task.item.status === 'paused' ? '恢复' : '暂停' }}
              </a-button>
              <a-button type="danger" @click="deleteTask(task)">删除</a-button>
            </template>
          </a-list-item-meta>
          <a-progress :percent="task.item.progress" status="active" showInfo style="width: calc(90% - 20px);" />
        </a-list-item>
      </template>
    </a-list>

    <!-- 创建任务模态框 -->
    <a-modal title="创建任务" v-model:visible="isCreateTaskModalVisible" @ok="createTask" @cancel="closeCreateTaskModal"
      :confirmLoading="confirmLoading">
      <a-form :model="task" :form="form" ref="form" :label-col="{ span: 6 }" :wrapper-col="{ span: 16 }">
        <!-- 动态渲染表单项 -->
        <a-form-item v-for="(item, index) in searchFormSchema" :key="index" :label="item.label" :rules="item.rules">
          <component :is="item.component" v-model:value="task[item.field]" v-bind="item.componentProps"
            v-if="item.component !== 'a-upload'" />
          <!-- 专门处理 JUpload -->
          <a-upload v-else v-model:file-list="task[item.field]" :action="item.componentProps?.action"
            :show-upload-list="false" :trigger="item.componentProps?.trigger">
            <a-button>上传文件</a-button>
          </a-upload>
        </a-form-item>
      </a-form>
    </a-modal>
  </a-card>
</template>

<script lang="ts" setup>
import { ref, onMounted, onUnmounted } from 'vue';
import { Modal } from 'ant-design-vue';
import { RefSymbol } from '@vue/reactivity';
import { getToken } from '@/utils/auth'
import { cloneDeep } from 'lodash-es';
import { defHttp } from '/@/utils/http/axios';

// 定义任务类型
interface Task {
  name: string;
  description: string;
  status: 'paused' | 'running';
  progress: number; // 任务进度，取值范围 0-100
  speed?: number;
  datasetName?: string;
  datasetUrl?: string;
  imgExt?: string;
  ufs?: string;
  imgFolder?: string;
  maxCategoryLevel?: number;
  metaPath?: string;
}

// 任务列表
// const taskList = ref<Task[]>([]);
const generateRandomProgress = () => parseFloat((Math.random() * 100).toFixed(2)); // 生成两位小数随机浮点数
const taskList = ref<Task[]>([]);
const maxTasks = ref(5); // 最大任务数
const concurrentTasks = ref(2); // 同时运行任务数
const isCreateTaskModalVisible = ref(false); // 创建任务模态框显示控制
const confirmLoading = ref(false); // 创建任务时的加载状态

// 新任务的数据
const form = ref(); // 表单的 ref
const task = ref<Task>({
  name: '',
  description: '',
  status: 'paused', // 默认为暂停状态
  progress: 0, // 初始进度为 0
});

// 定义表单结构
const searchFormSchema = ref([
  {
    label: '数据集名',
    field: 'datasetName',
    component: 'a-input',
    componentProps: { type: '' },
  },
  {
    label: '数据集路径',
    field: 'datasetUrl',
    component: 'a-input',
    componentProps: { type: '' },
  },
  {
    label: '图像类型',
    field: 'imgExt',
    component: 'a-select',
    componentProps: {
      options: [
        { value: 'tif', label: 'GeoTiff格式（tif）' },
        { value: 'jpg', label: 'JPEG格式（jpg）' },
        { value: 'png', label: 'PNG格式（png）' },
      ],
      mode: 'tags',
      showSearch: true,
    },
  },
  {
    label: '底层存储',
    field: 'ufs',
    component: 'a-select',
    componentProps: {
      options: [
        { value: 'alluxio', label: 'Alluxio' },
        { value: 's3', label: 'S3存储桶' },
        { value: 'hdfs', label: 'Hadoop File System' },
        { value: 'file', label: '本地存储' },
      ],
      mode: 'tags',
      showSearch: true,
    },
  },
  {
    label: '样本图像根目录',
    field: 'imgFolder',
    component: 'a-input',
    componentProps: { type: '' },
  },
  {
    label: '标签类别层数',
    field: 'maxCategoryLevel',
    component: 'a-input-number',
    componentProps: { min: 1 },
  },
  {
    label: '元数据文件',
    field: 'metaPath',
    component: 'a-upload',
    componentProps: { action: '/upload' },
  },
]);

// WebSocket 连接
let progressSocket: WebSocket | null = null;

// 初始化 WebSocket 连接
const initializeWebSocket = () => {
  // if (progressSocket) {
  //   console.log("WebSocket 已经初始化");
  //   return;
  // }

  const token = getToken(); // 获取 Token
  progressSocket = new WebSocket("ws://10.3.1.155:7011/websocket/progress/" + token, token);

  progressSocket.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log("收到推送:", data);
    const taskId = data.left;
    const progress = Math.min(Math.round(data.right * 100),100);

    console.log(`任务 ${taskId} 进度: ${progress}%`);

    // 更新任务进度
    const task = taskList.value.find(t => t.datasetName === taskId);
    console.log(`任务 ${task}`);

    if (task) {
      console.log("找到task",task)
      task.progress = progress;

      if (progress >= 100) {
        console.log(`任务 ${taskId} 已完成`);
      }
    }
  };

  progressSocket.onclose = () => {
    console.log("WebSocket 连接已关闭");
    setTimeout(initializeWebSocket, 2000);
  };

  progressSocket.onerror = (error) => {
    console.error("WebSocket 发生错误:", error);
    progressSocket = null;
  };

  console.log("WebSocket 连接已建立");
};

// 更新任务进度
const updateTaskProgress = (taskId: string, progress: number) => {
  const task = taskList.value.find(t => t.datasetName === taskId);
  if (task) {
    task.progress = progress;
  }
};


// 创建任务
const createTask = () => {
  form.value?.validateFields().then(() => {
    confirmLoading.value = true;

    // 构造 DatasetDTO 对象
    const datasetDTO = {
      datasetName: task.value.datasetName,
      datasetUrl: task.value.datasetUrl,
      labelPath: task.value.metaPath,
      imgExt: task.value.imgExt[0],
      imgFolders: task.value.imgFolder === undefined || task.value.imgFolder === ""
        ? [""]
        : task.value.imgFolder.split(','),
      maxCategoryLevel: task.value.maxCategoryLevel,
      sensor: "", // 根据需求添加值
      platForm: "", // 根据需求添加值
      bandInfo: [], // 根据需求添加值
      taskType: "Scene Classification", // 根据需求添加值
      metaPath: task.value.metaPath,
    };

    console.log("参数如下：", datasetDTO);
    // 调用 resolveDataSet 接口
    defHttp
      .put({
        url: '/jeecg-sample/sample/resolve',
        data: datasetDTO,
      })
      .then((response) => {
        console.log('接口调用成功:', response);
        task.value.progress = 0; // 设置初始进度为 0

        // 使用 cloneDeep 进行深拷贝，确保每个任务有独立副本
        const newTask = cloneDeep(task.value);

        addTaskToList(newTask); // 将任务添加到任务列表
        confirmLoading.value = false;
        closeCreateTaskModal();
      })
      .catch((error) => {
        console.error('接口调用失败:', error);
        Modal.error({
          title: '任务创建失败',
          content: '无法解析数据集，请检查输入并重试。',
        });
        confirmLoading.value = false;
      });
  });
};


// 展示创建任务模态框
const showCreateTaskModal = () => {
  isCreateTaskModalVisible.value = true;
};

// 关闭创建任务模态框
const closeCreateTaskModal = () => {
  isCreateTaskModalVisible.value = false;
};

// const startTaskProgress = (task: Task) => {
//   if (task.progress >= 100) return; // 如果任务已经完成，跳过更新

//   const interval = setInterval(() => {
//     if (task.progress < 100) {
//       task.progress += task.speed || 1; // 按指定速度增加进度
//       if (task.progress > 100) task.progress = 100; // 防止超出100%
//     } else {
//       clearInterval(interval); // 进度达到100%时停止更新
//       console.log(`任务 ${task.datasetName} 已完成`);
//     }
//   }, 1000); // 每秒更新一次
// };

// 添加任务到任务列表
const addTaskToList = (newTask: Task) => {
  if (taskList.value.length < maxTasks.value) {
    const taskWithSpeed = { ...newTask, speed: 2 }; // 设置默认速度为 2%/秒
    taskList.value.push(taskWithSpeed); // 深拷贝，避免引用同一对象
    // startTaskProgress(taskWithSpeed); // 启动进度条更新
    console.log(taskList.value);
  } else {
    Modal.error({
      title: '任务队列已满',
      content: `最多只能有 ${maxTasks.value} 个任务。`,
    });
  }
};



// 切换任务状态（暂停 / 恢复）
// 切换任务状态
const toggleTaskStatus = (task: { item: Task }) => {
  const taskId = task.item.datasetName;  // 获取任务ID
  const status = task.item.status;

  // 根据当前状态调用不同的接口
  const apiUrl = status === 'paused'
    ? `/jeecg-sample/sample/resume/${taskId}` // 恢复任务接口
    : `/jeecg-sample/sample/pause/${taskId}`; // 暂停任务接口

  // 调用API
  defHttp.get({ url: apiUrl })
    .then(() => {
      // 成功切换状态，更新任务状态
      task.item.status = status === 'paused' ? 'running' : 'paused';
    })
    .catch((error) => {
      console.error('任务状态切换失败:', error);
      Modal.error({
        title: '操作失败',
        content: '任务状态切换失败，请稍后再试。',
      });
    });
};


// 删除任务
const deleteTask = (task: { item: Task }) => {
  const taskId = task.item.taskId;  // 获取任务ID

  // 调用删除任务的接口
  defHttp.get({ url: `/jeecg-sample/sample/delete/${taskId}` })
    .then(() => {
      // 成功删除任务，从任务列表中移除该任务
      taskList.value = taskList.value.filter((t) => t.taskId !== taskId);
    })
    .catch((error) => {
      console.error('删除任务失败:', error);
      Modal.error({
        title: '删除失败',
        content: '任务删除失败，请稍后再试。',
      });
    });
};
// 在组件挂载时初始化 WebSocket
onMounted(() => {
  initializeWebSocket();
});

// 在组件卸载时关闭 WebSocket
onUnmounted(() => {
  if (progressSocket) {
    console.log("关闭 WebSocket 连接");
    progressSocket.close();
    progressSocket = null;
  }
});
</script>

<style scoped>
.task-info {
  margin-bottom: 20px;
}

a-list-item-meta {
  display: flex;
  flex-direction: column;
}

a-list-item-meta>.ant-list-item-meta-title {
  font-weight: bold;
  font-size: 16px;
}

a-list-item-meta>.ant-list-item-meta-description {
  font-size: 14px;
}

a-progress {
  margin-top: 10px;
  margin-bottom: 10px;
}

.task-meta {
  margin-left: 10px;
}
</style>
