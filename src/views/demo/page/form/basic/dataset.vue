<template>
    <!-- 自定义表单 -->
    <!-- <BasicForm @register="registerForm" style="margin: 50px 50px 0 50px" @submit="demoListApi" /> -->
    <div v-for="record in progressList" style="width: 420px; margin: 50px 300px 0;">
        <Typography.Title level={8}>{{ record.name }}</Typography.Title>
        <a-progress :key="record.name" :percent="record.data" size="big" :status="record.status" />
    </div>
    <div>
        <div class="card-list">
            <!-- 使用分页，展示当前页的卡片 -->
            <CardItem v-for="(dataset, index) in datasets" :key="index" :dataset="dataset" :onDelete="() => handleDeleteDataset(dataset.id)"/>
        </div>

        <!-- 分页按钮 -->
        <div class="pagination">
            <button @click="prevPage" :disabled="currentPage === 1">上一页</button>
            <span>页码 {{ currentPage }} / {{ totalPages }}</span>
            <button @click="nextPage" :disabled="currentPage === totalPages">下一页</button>
        </div>
    </div>
</template>

<script lang="ts" setup>
import { Input } from 'ant-design-vue';
import { useForm, BasicForm, FormSchema } from '/@/components/Form';
import { useListPage } from '/@/hooks/system/useListPage';
import { defHttp } from '/@/utils/http/axios';
import { reactive, onMounted, onUnmounted, ref, computed } from 'vue';
import { getToken } from '@/utils/auth';
import axios from 'axios';

import CardItem from './CardItem.vue';

// 假设这些数据是从后端接收的，这里我们使用 ref() 并假定初始数据如下
const progressList = ref([]);
const progressStatus: Ref<String | null> = ref(null)
const demoListApi = (params) => {
    console.log("处理前参数", params);
    if (params.datasetUrl != null) {
        params.datasetUrl = params.ufs + ":" + params.datasetUrl;
        if (params.imgFolder == null) params.imgFolder = "";
        params.imgFolder = params.imgFolder.split(",");
        if (params.bandInfo == null) params.bandInfo = [];
        params.labelPath = null;
        params.metaPath = null;
        params.platForm = "";
        params.sensor = "";
        params.taskType = "Scene Classification";

        console.log("处理后参数", params);
        defHttp.put({ url: '/jeecg-sample/sample/resolve', params });
        if (params.datasetName != null) {
            const newItem: ProgressItem = {
                name: params.datasetName,
                currentProgress: 0,
                totalProgress: 100,
                data: 0.0,
                status: "active",
                intervalId: null
            };
            newItem.intervalId = window.setInterval(() => { fetchProgress(newItem); }, 30000);
        }
    }
}

//定义表格查询字段
//表单搜索字段
// todo>>分辨率参数修正
const searchFormSchema: FormSchema[] = [
    {
        label: '数据集名', //显示label
        field: 'datasetName', //查询字段
        component: 'JInput', //渲染的组件
        componentProps: {
            type: ""
        },
    },
    {
        label: '数据集路径', //显示label
        field: 'datasetUrl', //查询字段
        component: 'JInput', //渲染的组件
        componentProps: {
            type: ""
        },
    },
    {
        label: '图像类型',
        field: 'imgExt',
        component: 'Select',
        componentProps: {
            options: [
                { value: 'tif', label: 'GeoTiff格式（tif）' },
                { value: 'jpg', label: 'JPEG格式（jpg）' },
                { value: 'png', label: 'PNG格式（png）' },
            ],
            //下拉多选
            mode: 'tags',
            //配置是否可搜索
            showSearch: true
        },
    },
    {
        label: '底层存储',
        field: 'ufs',
        component: 'Select',
        componentProps: {
            options: [
                { value: 'alluxio', label: 'Alluxio' },
                { value: 's3', label: 'S3存储桶' },
                { value: 'hdfs', label: 'Hadoop File System' },
                { value: 'file', label: '本地存储' },
            ],
            //下拉多选
            mode: 'tags',
            //配置是否可搜索
            showSearch: true
        },
    },
    {
        label: '样本图像根目录',
        field: 'imgFolder',
        component: 'JInput',
        componentProps: {
            type: ""
        },
    },
    {
        label: '标签类别层数',
        field: 'maxCategoryLevel',
        component: 'InputNumber',
    },
    {
        label: '元数据文件',
        field: 'metaPath',
        component: 'JUpload',
    },
];

const [registerForm] = useForm({
    //注册表单列
    schemas: searchFormSchema,
    showResetButton: false,
    submitButtonOptions: { text: '提交', preIcon: '' },
    actionColOptions: { span: 17 },
});

/**
 * 操作栏
 */
function getTableAction(record): ActionItem[] {
    return [
        {
            label: '编辑',
            onClick: handleEdit.bind(null, record),
        },
    ];
}

const fetchProgress = async (item: ProgressItem): Promise<void> => {
    try {
        await axios.get<{ data: [number, number] }>('http://10.3.1.155:7011/sample/resolve/progress/' + item.name).then((res) => {
            console.log("Res", res)
            if (res.data.length != 0) {
                const [currentProgress, totalProgress] = res.data; // 确保响应结构匹配
                item.currentProgress = currentProgress;
                item.totalProgress = totalProgress;
                item.data = (currentProgress / totalProgress) * 100;
            }
            else {
                item.currentProgress = 0;
                item.totalProgress = 9999;
                item.data = 0;
            }
            console.log("进度值", item.data)
            // 检查是否完成进度
            if (item.currentProgress >= item.totalProgress) {
                if (item.intervalId !== null) {
                    clearInterval(item.intervalId);
                }
            }
            else {
                if (!progressList.value.some(it => it.name === item.name))
                    progressList.value.push(item);
            }
        })

    } catch (error) {
        item.status = "exception";
        if (item.intervalId !== null) {
            clearInterval(item.intervalId);
        }
        console.error('Failed to fetch progress:', error);
    }
};

// 分页相关
const itemsPerPage = 10;  // 每页展示 10 个卡片（即 2 行，每行 5 个卡片）
const currentPage = ref(1);  // 当前页码
const totalPages = ref(0);   // 总页数
const datasets = ref([]);    // 当前页的数据

// 从后端加载分页数据
const loadData = async () => {
    console.log("加载样本集概览")
    defHttp.get(
        {
            url: '/jeecg-sample/sample/dataset/list',
            params: {
                page: currentPage.value,
                pageSize: itemsPerPage,
            }
        }
    ).then(paginationResult => {
        datasets.value = paginationResult.records; // 获取记录数组
        totalPages.value = paginationResult.pages; // 获取总页数

        console.log('Records:', datasets.value);
        console.log('Total Pages:', totalPages.value);
    })
    .catch(error => {
        console.error('Error fetching pagination data:', error);
    });
};
// 删除操作
const handleDeleteDataset = async (datasetId: number) => {
  try {
    await defHttp.delete({ 
        url: '/jeecg-sample/sample/dataset/delete',
        params: {
            datasetId: datasetId
        }
    });
    console.log('删除成功');
    loadData(); // 重新加载数据
  } catch (error) {
    console.error('删除失败', error);
  }
};
// 分页操作
const nextPage = () => {
    if (currentPage.value < totalPages.value) {
        currentPage.value++;
        loadData();  // 切换下一页时重新加载数据
    }
};

const prevPage = () => {
    if (currentPage.value > 1) {
        currentPage.value--;
        loadData();  // 切换上一页时重新加载数据
    }
};
onMounted(() => {
    loadData();
    progressList.value.forEach(item => {
        if (item.intervalId !== null) {
            clearInterval(item.intervalId);
        }
    });

});

</script>

<style scoped>
.card-list {
    display: grid;
    grid-template-columns: repeat(5, 1fr);
    /* 每行展示 5 个卡片 */
    grid-gap: 10px;
    /* 缩短卡片间距 */
    justify-items: center;
    margin-bottom: 20px;
}

.pagination {
    display: flex;
    justify-content: center;
    align-items: center;
    margin-top: 20px;
}

.pagination button {
    padding: 10px;
    margin: 0 5px;
    background-color: #007bff;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
}

.pagination button:disabled {
    background-color: #ccc;
}

.pagination span {
    margin: 0 10px;
}

.delete-button {
  margin-top: 10px;
  padding: 5px 10px;
  background-color: #f5f5f5;
  color: #333;
  border: 1px solid #ccc;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.3s, color 0.3s;
}

.delete-button:hover {
  background-color: #ff4d4f;
  color: white;
  border-color: #ff4d4f;
}
</style>