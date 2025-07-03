<template>
    <div class="search-container">
        <!-- 分辨率范围选择 -->
        <div class="range-container">
            <a-input-number v-model:value="resolutionStart" placeholder="最小分辨率" class="input-range" />
            <a-input-number v-model:value="resolutionEnd" placeholder="最大分辨率" class="input-range" />
        </div>

        <div class="range-container">
            <!-- 输入样本尺寸的输入框 -->
            <a-input v-model:value="sampleSize" placeholder="输入样本尺寸" style="width: 48%" />

            <!-- 空间选区子组件，右对齐 -->
            <SpaceSelection class="left-shift" />
        </div>

        <!-- 样本图像类型选择（支持多选） -->
        <a-select v-model:value="imgType" placeholder="选择样本图像类型" mode="multiple" class="input-field" allowClear>
            <a-select-option value="jpg">JPEG格式（jpg）</a-select-option>
            <a-select-option value="tif">GeoTiff格式（tif）</a-select-option>
            <a-select-option value="png">PNG格式（png）</a-select-option>
        </a-select>

        <!-- 输入多个标签类别
        <a-input v-model:value="inputLabels" placeholder="输入多个标签类别，以逗号分隔" @pressEnter="handleAddLabels"
            class="input-field" /> -->

        <a-select
            v-model:value="selectedTagIds" mode="multiple" show-search :placeholder="'输入标签类别，支持搜索和多选'"
            style="width: 100%; margin-bottom: 15px;"
            :default-active-first-option="false"
            :show-arrow="true" :filter-option="false" :not-found-content="fetching ? undefined : '未找到相关标签'"
            @search="handleSearchForCorrection" @focus="handleFocusOnSelect" @blur="handleBlurOnSelect" @change="handleTagsChange" allowClear
        >
            <a-select-option v-for="(label, id) in correctedTagOptions" :key="id" :value="label">
                {{ label }}
            </a-select-option>
        </a-select>

        <!-- <!== 标签校正时需要确认人工确认时的弹出卡片 ==>
        <a-modal v-model:visible="correctionPopoverVisible" title="标签校正" @ok="handleTagCorrectionConfirm"
            @cancel="handleTagCorrectionCancel" class="dynamic-dataset-modal">
            <a-form layout="vertical" class="jeecg-style-form">
                <!== 当前校正的标签切换 ==>
                <a-form-item label="当前需要确认的标签" class="form-item">
                    <p>{{ inputTagForCorrectionList[currentTagIndex] }}</p> <!== 显示当前要校正的标签 ==>
                </a-form-item>

                <!== 校正后的标签选择 ==>
                <a-form-item label="校正为" class="form-item">
                    <a-select v-model="selectedCorrectedTag" placeholder="选择校正后的标签" style="width: 100%;">
                        <a-select-option v-for="(label, id) in correctedTagOptions" :key="id" :value="id">
                            {{ label }}
                        </a-select-option>
                    </a-select>
                </a-form-item>

                <!== 切换下一个标签 ==>
                <a-form-item>
                    <a-button @click="handleNextTag">下一个</a-button>
                </a-form-item>
            </a-form>
        </a-modal> -->


        <!-- 标签类别下拉列表和全选框 -->
        <div class="select-and-dropdown">
            <!-- 全选按钮 -->
            <!-- <a-checkbox v-model:checked="selectAll" @change="handleSelectAll">全选</a-checkbox> -->

            <!-- <!== 标签下拉列表，支持多选和单项删除 ==>
            <a-select v-model:value="selectedTagFromDropdown" placeholder="选择标签" class="tag-dropdown">
                <!== 遍历渲染标签选项 ==>
                <a-select-option v-for="(tag) in tagList" :key="tag.id" :value="tag.id">
                    {{ tag.label }}
                </a-select-option>
            </a-select>

            <!== 清除所有选项的按钮 ==>
            <a-button type="danger" @click="clearAllTags" class="clear-button">
                清除全部
            </a-button> -->

            <!-- 准确样本按钮 -->
            <a-button type="primary" :icon="h(SearchOutlined)" @click="handleExactSample(selectedTagFromDropdown)">
                <template #icon><SearchOutlined /></template> 准确样本
            </a-button>

            <a-popover trigger="click" placement="bottom" v-model:visible="popoverVisible" title="选择相似标签">
                <template #content>
                    <a-select mode="multiple" v-model:value="selectedSimilarTags" placeholder="选择相似标签"
                        style="width: 200px;" allowClear>
                        <a-select-option v-for="(value, key) in similarTags" :key="key" :value="key">
                            {{ value }}
                        </a-select-option>
                    </a-select>
                    <a-button :icon="h(SearchOutlined)" @click="handleSupplementSample(selectedSimilarTags)"
                        style="margin-top: 10px;">
                        检索
                    </a-button>
                </template>
                <a-button type="default" :icon="h(SearchOutlined)" class="action-button" @click="handleSimilarCategories(selectedTagFromDropdown)">
                    <template #icon><SearchOutlined /></template> 补充样本
                </a-button>
            </a-popover>
            <div class="right-aligned">
                <!-- 上传标签附件按钮 -->
                <a-button type="primary" @click="showUploadLabelModal" class="upload-label-button">上传标签附件</a-button>

                <!-- 导出动态数据集按钮 -->
                <a-button type="danger" @click="showDynamicDatasetModal" class="import-button">导出动态数据集</a-button>
            </div>

            <!-- 上传标签附件的弹出框 -->
            <a-modal v-model:visible="uploadLabelModalVisible" title="上传标签附件" @ok="handleUploadLabel"
                @cancel="handleUploadLabelModalCancel">
                <a-form layout="vertical" class="jeecg-style-form">
                    <!-- 数据集选择 -->
                    <a-form-item label="选择数据集" class="form-item">
                        <a-select v-model="selectedDatasetId" placeholder="选择数据集" style="width: 100%;">
                            <a-select-option v-for="dataset in datasetOptions" :key="dataset.id" :value="dataset.id">
                                {{ dataset.datasetName }}
                            </a-select-option>
                        </a-select>
                    </a-form-item>

                    <!-- CSV文件上传 -->
                    <a-form-item label="上传CSV文件" class="form-item">
                        <a-upload :beforeUpload="beforeUpload" :fileList="fileList" @remove="handleFileRemove">
                            <a-button icon="upload">点击上传</a-button>
                        </a-upload>
                    </a-form-item>

                    <!-- 添加标签信息按钮 -->
                    <a-form-item>
                        <a-button type="primary" @click="handleUploadLabel">添加标签信息</a-button>
                    </a-form-item>
                </a-form>
            </a-modal>

            <!-- 动态数据集输入信息的弹出卡片 -->
            <a-modal
                v-model:visible="dynamicDatasetModalVisible"
                title="导出动态数据集"
                
                class="dynamic-dataset-modal"
            >
                <a-form layout="vertical" class="jeecg-style-form">
                    <a-form-item label="数据集名称" class="form-item">
                        <a-input v-model:value="dynamicDatasetName" placeholder="请输入数据集名称" class="form-input" />
                    </a-form-item>

                    <a-form-item label="数据集描述" class="form-item">
                        <a-textarea v-model:value="dynamicDatasetDescription" placeholder="请输入数据集描述" rows="4"
                            class="form-textarea" />
                    </a-form-item>
                </a-form>

                <template #footer>
                    <a-button key="back" @click="handleModalCancel">取消</a-button>
                    <a-button key="download" type="default" @click="handleDownloadDataset">下载数据集</a-button>
                    <a-button key="submit" type="primary" :loading="confirmLoading" @click="handleDynamicDatasetSubmit">确定</a-button>
                </template>
            </a-modal>

        </div>

        <!-- 样本数据表格 -->
        <a-table :columns="columns" :dataSource="sampleData" rowKey="id" class="table-container">
            <template v-slot:action="{ record }">
                <!-- 用于勾选 -->
                <a-checkbox v-model:checked="selectedSamples[record.id]"
                    @change="updateSelectedSamples(record)"></a-checkbox>
            </template>
            <template v-slot:img="{ record }">
                <TableImg :size="60" :simpleShow="true"
                    :imgList="['http://10.3.1.128:8084/category/thumbnail/' + record.id]" />
            </template>
        </a-table>

        <!-- 表格下方的分页按钮 -->
        <div class="pagination-buttons">
            <a-button @click="goToFirstPage" :disabled="pageNo === 1">第一页</a-button>
            <a-button @click="goToPreviousPage" :disabled="pageNo === 1">上一页</a-button>
            <span>第 {{ pageNo }} 页 / 共 {{ Math.ceil(totalSamples / pageSize) }} 页</span>
            <a-button @click="goToNextPage" :disabled="pageNo === Math.ceil(totalSamples / pageSize)">下一页</a-button>
            <a-button @click="goToLastPage" :disabled="pageNo === Math.ceil(totalSamples / pageSize)">最后一页</a-button>
        </div>
    </div>
</template>
<script lang="ts" setup>
import { ref, watch, computed } from 'vue';
import { defHttp } from '/@/utils/http/axios';
import { message,Checkbox } from 'ant-design-vue';
import { TableImg } from '/@/components/Table';
import { randomInt } from 'crypto';
import SpaceSelection from './mapTangle.vue';  // 引入空间选区子组件
import { h } from 'vue';
import { SearchOutlined } from '@ant-design/icons-vue'; // <-- 添加这一行

const isExactSampleQuery = ref(true); // 用于区分当前是准确样本查询还是补充样本查询
const similarTags = ref<{ [key: number]: string }>({}); // 存储补充样本检索返回的相似标签
const selectedSimilarTags = ref<string[]>([]); // 用户从多选下拉框中选中的相似标签
const popoverVisible = ref(false); // 控制补充样本检索的弹出框可见性
const inputLabels = ref(''); // 用户输入的标签类别
const tagList = ref<{ id: number, label: string }[]>([]); // 标签列表，存储 id 和标签名称
const selectedTagFromDropdown = ref<string | null>(null); // 从下拉列表选择的标签
const sampleData = ref([]); // 样本数据
const selectedSamples = ref<{ [key: string]: boolean }>({}); // 当前页面中每个样本的选中状态

// 新增状态，用于控制弹出框的可见性和动态数据集的输入信息
const dynamicDatasetModalVisible = ref(false); // 控制弹出框是否可见
const dynamicDatasetName = ref(''); // 数据集名称
const dynamicDatasetDescription = ref(''); // 数据集描述

// 全选状态
const selectAll = ref(false);

// 搜索条件
const resolutionStart = ref<number | null>(0); // 分辨率范围开始值
const resolutionEnd = ref<number | null>(10); // 分辨率范围结束值
const sampleSize = ref(''); // 样本尺寸
const imgType = ref<string[]>([]); // 样本图像类型，支持多选

// 全局选中样本映射，键为标签，值为样本ID列表
const selectedSamplesMap = ref<{ [tag: string]: number[] }>({});

// 校正卡片显示状态
const correctionPopoverVisible = ref(false);
const inputTagForCorrection = ref(''); // 用户输入的标签
const inputTagForCorrectionList = ref<string[]>([]); // 存储所有需要确认的标签
const currentTagIndex = ref(0); // 当前处理的标签索引
const correctedTagOptionsList = ref<{ [key: string]: any }>({}); // 存储每个标签对应的校正选项

// 标签校正部分的新状态
const correctedTagOptions = ref<{ [key: string]: string }>({}); // 校正接口返回的标签列表，用于下拉选择
const selectedTagForInput = ref<string | null>(null); // 用户选择的校正标签的 ID
let correctionTimeout: ReturnType<typeof setTimeout> | null = null; // 用于防抖的定时器
// 标签列表，存储 id 和标签名称 (保持不变，但现在由selectedTagIds同步)

// 新增 ref 来绑定 a-select 的多选值
const selectedTagIds = ref<string[]>([]); // 绑定到a-select的v-model，存储已选标签的ID数组


// 分页相关
const pageNo = ref(1); // 当前页码
const pageSize = ref(10); // 每页显示的样本数
const totalSamples = ref(0); // 样本总数，用于分页

// csv上传相关
const uploadLabelModalVisible = ref(false); // 控制上传标签附件的弹出框可见性
const datasetOptions = ref([]); // 存储数据集选项
const selectedDatasetId = ref(null); // 选择的数据集ID
const fileList = ref([]); // 上传的文件列表
const confirmLoading = ref(false); // 用于控制确定按钮的加载状态

// 定义表格的列
const columns = [
    { title: '样本号', dataIndex: 'id', key: 'id', width: 80 },
    { title: '样本预览', dataIndex: 'labelPath', key: 'img', width: 80, slots: { customRender: 'img' } },
    { title: '数据集', dataIndex: 'datasetName', key: 'datasetName', width: 80 },
    { title: '样本标签', dataIndex: 'labelName', key: 'labelName', width: 120 },
    { title: '分辨率', dataIndex: 'resolution', key: 'resolution', width: 70 },
    { title: '样本尺寸', dataIndex: 'sampleSize', key: 'sampleSize', width: 140 },
    { title: '样本图像类型', dataIndex: 'imgType', key: 'imgType', width: 140 },
    {
        // 修改后的 action 列
        dataIndex: 'action',
        key: 'action',
        width: 100,
        slots: { customRender: 'action' }, // 此插槽用于行渲染
        title: () => h(
            Checkbox,
            {
                checked: selectAll.value,
                onChange: handleSelectAll,
            },
            '全选' // 复选框旁边的文本，如果需要的话
        ), // 此插槽用于表头渲染
    },
];

// 显示弹出框
const showDynamicDatasetModal = () => {
    dynamicDatasetModalVisible.value = true;
};

// 关闭弹出框并重置输入信息
const handleModalCancel = () => {
    dynamicDatasetModalVisible.value = false;
    dynamicDatasetName.value = '';
    dynamicDatasetDescription.value = '';
};

// 新增的下载数据集方法
const handleDownloadDataset = async () => {
    if (!dynamicDatasetName.value.trim()) {
        message.warning('请先填写数据集名称');
        return;
    }

    // 实际下载逻辑：根据数据集名称请求后端接口
    try {
        // 注意：这里需要根据你实际的后端接口和下载方式进行调整
        // 假设后端有一个接口 /jeecg-sample/sample/dynamic/download
        // 并且支持以文件流的形式返回数据集
        defHttp.get({ url: 'http://10.3.1.128:8084/category/thumbnail/' + 422643 })
        const response = await defHttp.get(
            {
                url: `http://10.3.1.128:8084/category/download_dataset/${dynamicDatasetName.value}`,
                responseType: 'blob', // 关键：指定响应类型为 blob，用于文件下载
            },
            { isReturnNativeResponse: true } // 确保返回完整的响应对象
        );

        if (response.data) {
            // 创建一个 Blob 对象
            const blob = new Blob([response.data], { type: response.headers['content-type'] });
            // 创建一个下载链接
            const url = window.URL.createObjectURL(blob);
            const link = document.createElement('a');
            link.href = url;
            // 获取文件名，如果后端在响应头中提供了
            const contentDisposition = response.headers['content-disposition'];
            let fileName = 'dataset.zip'; // 默认文件名
            if (contentDisposition) {
                const fileNameMatch = contentDisposition.match(/filename\*?=['"]?(?:UTF-8'')?([^"']+)/);
                if (fileNameMatch && fileNameMatch[1]) {
                    fileName = decodeURIComponent(fileNameMatch[1]);
                }
            }
            link.setAttribute('download', fileName);
            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
            window.URL.revokeObjectURL(url);
            message.success('数据集下载成功！');
        } else {
            message.error('下载数据集失败：未收到文件数据。');
        }
    } catch (error) {
        console.error('下载数据集出错:', error);
        message.error('下载数据集失败，请稍后重试。');
    }
};

// 提交动态数据集信息并调用原有的导入逻辑
const handleDynamicDatasetSubmit = async () => {
    if (!dynamicDatasetName.value.trim()) {
        message.warning('请填写数据集名称');
        return;
    }
    message.success('导出成功');
    confirmLoading.value = true;
    if (Object.keys(selectedSamplesMap.value).length === 0) {
        message.warning('没有选中的样本');
        return;
    }

    // 检查 selectedSamplesMap.value 是否为有效对象
    if (!selectedSamplesMap.value || typeof selectedSamplesMap.value !== 'object' || Array.isArray(selectedSamplesMap.value)) {
        throw new Error('选中的样本映射无效');
    }

    // 调用后端接口，并捕获返回值
    const response = await defHttp.post({
        url: '/jeecg-sample/sample/dynamic/create',
        data: {
            datasetName: dynamicDatasetName.value,
            // description: dynamicDatasetDescription.value,
            insMap: selectedSamplesMap.value
        }
    });

    // 检查返回值是否为 true 或预期的成功值
    if (response === true || response.success) {
        message.success('导出成功');

        // 清空选中样本映射
        selectedSamplesMap.value = {};

        // 关闭弹出框并重置输入信息
        dynamicDatasetModalVisible.value = false;
        dynamicDatasetName.value = '';
        dynamicDatasetDescription.value = '';
    } else {
        throw new Error('导出失败，服务器返回不正确');
    }

};

// 点击确定，将校正后的标签加入标签列表
const handleTagCorrectionConfirm = () => {
    if (selectedCorrectedTag.value) {
        const correctedTagName = correctedTagOptions.value[selectedCorrectedTag.value];
        tagList.value.push({ id: selectedCorrectedTag.value, label: correctedTagName }); // 添加选择的校正标签
    }
    correctionPopoverVisible.value = false; // 关闭校正卡片
    selectedCorrectedTag.value = ''; // 清空选择
};

// 标签校正逻辑：现在由a-select的@search事件触发
const handleSearchForCorrection = (value: string) => {
    // 如果输入内容包含分号，则不进行校正，并清空选项
    if (value.includes(';')) {
        if (correctionTimeout) {
            clearTimeout(correctionTimeout);
        }
        correctedTagOptions.value = {};
        fetching.value = false;
        return;
    }

    if (correctionTimeout) {
        clearTimeout(correctionTimeout);
    }

    fetching.value = true; // 开始加载
    correctedTagOptions.value = {}; // 每次新搜索时清空旧的选项，等待新结果

    correctionTimeout = setTimeout(async () => {
        if (value.trim() === '') {
            correctedTagOptions.value = {};
            fetching.value = false;
            return;
        }
        try {
            const res = await correctTag(value.trim());
            correctedTagOptions.value = res || {};
            fetching.value = false; // 结束加载
        } catch (error) {
            console.error('标签校正失败:', error);
            correctedTagOptions.value = {};
            fetching.value = false; // 结束加载
        }
    }, 500); // 0.5秒延迟校正
};

// 新增：处理a-select选中项变化，并同步tagList
const handleTagsChange = (values: string[]) => {
    // 清空当前的 tagList，然后根据 selectedTagIds 重新构建
    tagList.value = [];
    values.forEach(id => {
        // 尝试从 correctedTagOptions 中获取标签名称
        // 或者从一个已有的所有标签的映射中获取（如果你有这样的数据）
        const labelName = correctedTagOptions.value[id] || `未知标签(${id})`; // 提供一个默认值
        // 这里你可能需要一个更全面的标签名称查找逻辑
        // 例如，如果 tagList 中已经有这个ID的标签，就直接用 tagList 中的名称

        tagList.value.push({ id: parseInt(id), label: labelName });
        console.log("correctedTagOptions：",correctedTagOptions.value);
    });
    
    selectedTagFromDropdown.value = Object.keys(correctedTagOptions.value).join(',');
    // 每次选中后，清空当前的校正选项，避免下拉列表一直显示
    correctedTagOptions.value = {};
    fetching.value = false;
};

// 优化：当a-select失去焦点时，清空校正选项，避免下次直接打开显示旧结果
const handleBlurOnSelect = () => {
    correctedTagOptions.value = {}; // 失去焦点后清空校正选项
    fetching.value = false;
};

// 点击取消，关闭校正卡片
const handleTagCorrectionCancel = () => {
    correctionPopoverVisible.value = false;
    selectedCorrectedTag.value = '';
    inputTagForCorrectionList.value = [];
    currentTagIndex.value = 0;
};

// 加载当前需要校正的标签的校正选项
const loadCorrectedOptions = () => {
    const currentTag = inputTagForCorrectionList.value[currentTagIndex.value];
    correctedTagOptions.value = correctedTagOptionsList.value[currentTag] || {}; // 加载当前标签对应的选项
};

const fetching = ref(false); // 用于控制加载状态


// 点击下一步切换到下一个标签
const handleNextTag = () => {
    if (currentTagIndex.value < inputTagForCorrectionList.value.length - 1) {
        currentTagIndex.value++;
        loadCorrectedOptions(); // 切换标签时加载对应的校正选项
        selectedCorrectedTag.value = ''; // 清空当前选择
    } else {
        handleTagCorrectionConfirm();
    }
};


// 添加标签到标签列表，并在必要时弹出校正卡片
const handleAddLabels = async () => {
    if (!inputLabels.value.trim()) {
        message.warning('请输入标签类别');
        return;
    }

    const labels = inputLabels.value.split(',').map(label => label.trim()).filter(label => label !== '');
    for (const label of labels) {
        const correctedTags = await correctTag(label);
        // 检查返回的结果是否为空，若为空则跳过当前标签
        if (!correctedTags || Object.keys(correctedTags).length === 0) {
            continue;
        }
        const correctedTagIds = Object.keys(correctedTags);
        const correctedTagValues = Object.values(correctedTags);

        // 如果校正结果不一致或有多个标签，记录需要确认的标签
        if (correctedTagIds.length > 1 || correctedTagValues[0] !== label) {
            inputTagForCorrectionList.value.push(label);
            correctedTagOptionsList.value[label] = correctedTags; // 将每个标签的校正选项存储起来
        } else {
            tagList.value.push({ id: correctedTagIds[0], label: correctedTagValues[0] });
        }
    }

    if (inputTagForCorrectionList.value.length > 0) {
        currentTagIndex.value = 0; // 重置索引为第一个需要确认的标签
        loadCorrectedOptions(); // 加载第一个标签的校正选项
        correctionPopoverVisible.value = true; // 显示校正卡片
    }
    visible
    inputLabels.value = ''; // 清空输入框
    console.log(tagList.value)
};

// 分页逻辑
const goToFirstPage = () => {
    if (pageNo.value !== 1) {
        pageNo.value = 1;
        reloadSamples(); // 重新获取数据
    }
};

const goToPreviousPage = () => {
    if (pageNo.value > 1) {
        pageNo.value--;
        reloadSamples(); // 重新获取数据
    }
};

const goToNextPage = () => {
    if (pageNo.value < Math.ceil(totalSamples.value / pageSize.value)) {
        pageNo.value++;
        reloadSamples(); // 重新获取数据
    }
};

const goToLastPage = () => {
    const lastPage = Math.ceil(totalSamples.value / pageSize.value);
    if (pageNo.value !== lastPage) {
        pageNo.value = lastPage;
        reloadSamples(); // 重新获取数据
    }
};

// 清除所有选择的标签
const clearAllTags = () => {
    tagList.value.splice(0, tagList.value.length);  // 清空下拉框中的所有选项
    selectedTagFromDropdown.value = null;
};

// 根据当前页面加载准确或补充样本
const reloadSamples = () => {
    if (isExactSampleQuery.value) {
        handleExactSample(selectedTagFromDropdown.value); // 准确样本分页查询
    } else {
        handleSupplementSample(selectedSimilarTags.value); // 补充样本分页查询
    }
};

// 调用校正接口进行标签校正
const correctTag = async (inputTag: string) => {
    try {
        const res = await defHttp.get({ url: `/jeecg-classify/classify/query/${inputTag}` })
        return res || {};
    } catch (error) {
        // console.error('标签校正请求失败:', error); // 不再弹窗，只记录
        return {};
    }
};

// 更新选中的样本，添加到全局映射中
const updateSelectedSamples = (record: any) => {
    if (!selectedTagFromDropdown.value) return;

    const tag = selectedTagFromDropdown.value;
    if (!selectedSamplesMap.value[tag]) {
        selectedSamplesMap.value[tag] = [];
    }

    // 如果选中，添加到映射；取消选中则移除
    if (selectedSamples.value[record.id]) {
        selectedSamplesMap.value[tag].push(record.id);
    } else {
        selectedSamplesMap.value[tag] = selectedSamplesMap.value[tag].filter(id => id !== record.id);
    }
};

// 准确样本分页检索
const handleExactSample = async (tag: string | null) => {
    if (!tag) {
        message.warning('请选择一个标签');
        return;
    }
    try {
        popoverVisible.value = false; // 新增：关闭卡片
        isExactSampleQuery.value = true; // 设置为准确样本查询
        const params = { labelId_MultiString: selectedTagFromDropdown.value, resolution_begin: getResolution()?.split(",")[0], resolution_end: getResolution().split(",")[1], sampleSize: sampleSize.value, imgType: imgType.value.join(','), pageNo: pageNo.value, pageSize: pageSize.value };
        const res = await defHttp.get({ url: '/jeecg-sample/sample/query/exact', params });
        sampleData.value = res.records;
        totalSamples.value = res.total;
        console.log("先全局反选！");
        handleSelectAll({ target: { checked: true } }); // 全选
    } catch (error) {
        message.error('检索样本失败' + error.message);
    }
};


// 补充样本检索方法，弹出相似标签列表
const handleSimilarCategories = async (tag: string | null) => {
    if (!tag) {
        message.warning('请选择一个标签');
        return;
    }
    try {
        const res = await defHttp.get({ url: `/jeecg-classify/classify/query/associated/${tag}` });
        similarTags.value = res || {};
        popoverVisible.value = true; // 显示弹出框
    } catch (error) {
        message.error('获取相似标签失败');
    }
};


// 补充样本分页检索
const handleSupplementSample = async (tags: string[] | null) => {
    if (!tags) {
        message.warning('请选择一个标签');
        return;
    }
    try {
        popoverVisible.value = false; // 新增：关闭卡片
        isExactSampleQuery.value = false; // 设置为补充样本查询
        const params = {
            labelId_MultiString: tags.join(','),
            labelId_Filter: selectedTagFromDropdown.value,
            resolution_begin: getResolution()?.split(",")[0], resolution_end: getResolution().split(",")[1],sampleSize: sampleSize.value, imgType: imgType.value.join(','), pageNo: pageNo.value, pageSize: pageSize.value
        };
        const res = await defHttp.get({ url: '/jeecg-sample/sample/query/general', params });
        sampleData.value = res.records;
        totalSamples.value = res.total;
        console.log("先全局反选！");
        handleSelectAll({ target: { checked: true } }); // 全选
    } catch (error) {
        console.log(error);
        message.error('检索相似样本失败');
    }
};

// 获取分辨率范围
const getResolution = () => {
    return (resolutionStart.value !== null && resolutionEnd.value !== null)
        ? `${resolutionStart.value},${resolutionEnd.value}`
        : undefined;
};

// 全选/取消全选
const handleSelectAll = (e) => {
    selectAll.value = e.target.checked; // 根据复选框的选中状态更新 selectAll
    console.log("selectAll ", selectAll.value);
    if (selectAll.value) {
        // 全选
        sampleData.value.forEach(sample => {
            selectedSamples.value[sample.id] = true;
            updateSelectedSamples(sample);
        });
    } else {
        // 取消全选
        sampleData.value.forEach(sample => {
            selectedSamples.value[sample.id] = false;
            updateSelectedSamples(sample);
        });
    }
    console.log(selectedSamples.value);
};

// 监听样本数据变化，自动调整选中状态
watch(sampleData, (newSampleData) => {
    newSampleData.forEach(sample => {
        if (selectedSamples.value[sample.id] === undefined) {
            selectedSamples.value[sample.id] = false;
        }
    });
});

// 显示上传标签附件的弹出框
const showUploadLabelModal = () => {
    uploadLabelModalVisible.value = true;
    fetchDatasetOptions(); // 加载数据集选项
};

// 取消并重置上传标签附件的弹出框
const handleUploadLabelModalCancel = () => {
    uploadLabelModalVisible.value = false;
    selectedDatasetId.value = null;
    fileList.value = [];
};

// 获取数据集列表
const fetchDatasetOptions = async () => {
    defHttp.get(
        {
            url: '/jeecg-sample/sample/dataset/list',
            params: {
                page: 0,
                pageSize: -1,
            }
        }
    ).then(paginationResult => {
        datasetOptions.value = paginationResult.records;
    }).catch(error => {
        console.error('Error fetching pagination data:', error);
    });
};

// 上传CSV文件之前的处理
const beforeUpload = (file) => {
    fileList.value = [file]; // 保持最新的上传文件
    return false; // 阻止默认的上传行为
};

// 移除上传的文件
const handleFileRemove = () => {
    fileList.value = [];
};

// 提交上传的CSV文件和数据集ID
const handleUploadLabel = async () => {
    if (!selectedDatasetId.value) {
        message.warning('请选择数据集');
        return;
    }
    if (!fileList.value.length) {
        message.warning('请上传CSV文件');
        return;
    }

    const formData = new FormData();
    formData.append('file', fileList.value[0]);
    formData.append('id', selectedDatasetId.value);

    try {
        const response = await defHttp.post({
            url: '/jeecg-sample/sample/csv',
            data: formData,
            headers: { 'Content-Type': 'multipart/form-data' },
        });
        if (response.success) {
            message.success('标签信息上传成功');
            uploadLabelModalVisible.value = false; // 关闭模态框
            selectedDatasetId.value = null;
            fileList.value = [];
        } else {
            message.error('标签信息上传失败');
        }
    } catch (error) {
        message.error('标签信息上传失败');
    }
};
</script>
<style scoped>
.clear-button {
    background-color: #ff4d4f;
    color: #fff;
    border-radius: 5px;
    padding: 5px 10px;
    margin-right: 10px;
}

.clear-button:hover {
    background-color: #d9363e;
    border-color: #d9363e;
}

.pagination-buttons {
    margin-top: 20px;
    display: flex;
    justify-content: center;
    align-items: center;
}

.pagination-buttons a-button {
    margin: 0 5px;
}

/* 对按钮的样式进行调整 */
.action-button {
    margin-left: 10px;
}

/* 右对齐导入按钮 */
.right-aligned {
    margin-left: auto;
}

.import-button {
    background-color: #f5222d;
    border-color: #f5222d;
    color: #fff;
    margin-left: 20px;
    padding: 6px 16px;
    font-weight: bold;
    border-radius: 5px;
}

/* 按钮悬停时的样式 */
.import-button:hover {
    background-color: #1313cf;
    border-color: #13cfcf;
}

a-modal {
    max-width: 400px;
}

.search-container {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
    background-color: #f0f2f5;
    border: 1px solid #d9d9d9;
    border-radius: 10px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.input-field {
    width: 100%;
    margin-bottom: 15px;
    font-size: 14px;
}

.range-container {
    display: flex;
    justify-content: space-between;
    margin-bottom: 20px;
}

.input-range {
    width: 48%;
    font-size: 14px;
}

.left-shift {
    position: absolute;
    left: calc(52.3% - 10px);
    /* 将组件从右侧稍微移向左侧 */
    width: 36%;
    /* 增大 SpaceSelection 的宽度 */
}

.select-and-dropdown {
    display: flex;
    align-items: center;
    margin-bottom: 15px;
}

.tag-dropdown {
    margin-left: 15px;
    width: 200px;
}

a-button {
    margin-left: 10px;
}

.table-container {
    margin-top: 20px;
    background-color: #fff;
    border-radius: 10px;
}

.sample-image {
    width: 60px;
    height: auto;
}

.select-all-container {
    margin-bottom: 10px;
    font-size: 14px;
}

.sample-image {
    width: 50px;
    height: auto;
}

/* JeecgVue 风格的表单整体样式 */
.jeecg-style-form {
    padding: 20px;
    /* 表单的内边距 */
}

/* 表单项样式 */
.form-item {
    margin-bottom: 20px;
    /* 表单项的外边距 */
    font-family: 'Microsoft YaHei', sans-serif;
    /* JeecgVue 常用字体 */
    font-weight: 500;
    font-size: 15px;
    /* 稍大一点的字体 */
}

/* 输入框样式 */
.form-input {
    width: 100%;
    padding: 10px 15px;
    /* 输入框的内边距 */
    border-radius: 4px;
    /* 圆角 */
    border: 1px solid #d9d9d9;
    font-size: 14px;
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
    /* 添加轻微阴影 */
}

/* 多行文本框样式 */
.form-textarea {
    width: 100%;
    padding: 12px 15px;
    border-radius: 4px;
    border: 1px solid #d9d9d9;
    font-size: 14px;
    box-shadow: 0 1px 4px rgba(0, 0, 0, 0.1);
}

/* 优化弹出框的整体样式 */
.dynamic-dataset-modal {
    max-width: 500px;
    /* 弹出框的宽度 */
    padding: 20px;
    border-radius: 6px;
    font-family: 'Microsoft YaHei', sans-serif;
    /* JeecgVue 常用字体 */
}

/* 按钮的样式（可以根据 JeecgVue 的风格进一步优化） */
a-button {
    padding: 10px 20px;
    font-size: 14px;
    font-weight: 600;
}

.upload-label-button {
    margin-right: 10px;
    background-color: #52c41a;
    color: #fff;
    font-weight: bold;
    border-radius: 5px;
}

.upload-label-button:hover {
    background-color: #389e0d;
    border-color: #389e0d;
}
</style>
