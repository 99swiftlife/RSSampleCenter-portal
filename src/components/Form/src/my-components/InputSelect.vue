<template>
    <div class="select-with-button">
        <a-tree-select
            mode="combobox"
            v-model:value="inputValue"
            :tree-data="options"
            :show-search="true"
            :treeCheckStrictly="true"
            :dropdown-style="{ maxHeight: '400px', overflow: 'auto' }"
            style="flex-grow: 1;"
            tree-checkable
            allow-clear
            multiple
            tree-node-filter-prop="label"
            placeholder="请输入标签类名"
            @search="handleChange"
            @change="sendEvent"
        >
        </a-tree-select>
        <!-- <a-button style="width: 3em;" @click="handleSearch" type="primary">
            <SearchOutlined />
        </a-button> -->
    </div>
</template>

<script setup>
import { ref } from 'vue';;
import { defHttp } from '/@/utils/http/axios';
import { SearchOutlined } from '@ant-design/icons-vue';
import { defineEmits } from 'vue';

const inputValue = ref('');
const options = ref([]);
let no_null_value = undefined;
let searchTimeout = null; // 保存定时器的变量

const emit = defineEmits(['change']);

// 发出 change 事件
function sendEvent() {
    let ids = [];
    inputValue.value.forEach(element => {
        ids.push(element.value);
    });
    console.log("IDS", ids);
    emit('change', ids.join(','));
}

// 获取关联数据
const getAsociate = async (id) => {
    try {
        const sub_res = await defHttp.get({ url: `/jeecg-classify/classify/query/associated/${id}` });
        const res = Object.entries(sub_res).map(([key, value]) => ({
            title: value,
            value: key,
            key: key,
        }));
        return res;
    } catch (error) {
        console.error('Failed to fetch data:', error);
        return [];
    }
};

// 调用 API 获取数据并更新 options
const handleSearch = async () => {
    console.log("触发查找事件！");
    if (!no_null_value) {
        options.value = [];
        return;
    }
    try {
        console.log("查找标签", no_null_value);
        defHttp.get({ url: `/jeecg-classify/classify/query/${no_null_value}` }).then((res) => {
            console.log(res);

            async function processEntries(entries) {
                const results = [];
                for (const [key, value] of entries) {
                    const children = await getAsociate(key);
                    results.push({
                        value: "k-" + key,
                        title: value,
                        key: key,
                        children: children
                    });
                }
                return results;
            }

            const entries = Object.entries(res);
            processEntries(entries).then(re => {
                options.value = re;
            });
        });
        no_null_value = undefined;
    } catch (error) {
        console.error('Failed to fetch data:', error);
        no_null_value = undefined;
    }
};

// 输入变化后触发防抖搜索
const handleChange = (value) => {
    console.log("current value",value)
    if (value) {
        no_null_value = value;
    }

    // 清除之前的定时器，防止频繁调用 handleSearch
    if (searchTimeout) {
        clearTimeout(searchTimeout);
    }

    // 设置新的定时器，如果 1 秒内没有新的输入变化，触发 handleSearch
    searchTimeout = setTimeout(() => {
        handleSearch();
    }, 1000);
};

const handleBlur = (value) => {
    inputValue.value = value;
};
</script>

<style>
.select-with-button {
    display: flex;
    align-items: center;
    width: 100%;
    gap: 8px;
}
</style>
