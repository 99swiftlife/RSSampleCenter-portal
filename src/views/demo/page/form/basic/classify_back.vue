<template>
    <!-- <super-query :config="superQueryConfig" @search="handleSuperQuery" /> -->
    <div ref="cyContainer"
        style="width: 40%; height: 300px; float: right; background-color: 
        blanchedalmond; margin: 20px; border: 3px solid #decba4; box-shadow: 0 4px 8px rgba(0,0,0,0.1); border-radius: 10px;">
    </div>
    <BasicTable @register="registerTable">
        <template #add="{ field }">
            <Button v-if="Number(field) === 0" @click="add">+</Button>
            <Button v-if="field > 0" @click="del(field)">-</Button>
        </template>
        <!--操作栏-->
        <template #action="{ record }">
            <TableAction :actions="getTableAction(record)" />
        </template>
        <template #img="{ record }">
            <TableImg :size="60" :simpleShow="true"
                :imgList="['http://10.3.1.128:8084/category/thumbnail/' + record.id]" />
        </template>
    </BasicTable>
    <BasicTable @register="registerTable2" :key="tableKey">
        <template #add="{ field }">
            <Button v-if="Number(field) === 0" @click="add">+</Button>
            <Button v-if="field > 0" @click="del(field)">-</Button>
        </template>
        <!--操作栏-->
        <template #action="{ record }">
            <TableAction :actions="getTableAction(record)" />
        </template>
        <template #img="{ record }">
            <TableImg :size="60" :simpleShow="true"
                :imgList="['http://10.3.1.128:8084/category/thumbnail/' + record.id]" />
        </template>
    </BasicTable>
</template>record

<script lang="ts" setup>
import cytoscape from 'cytoscape';
import cyqtip from 'cytoscape-qtip';
import { Input } from 'ant-design-vue';
import { ActionItem, BasicColumn, BasicTable, TableAction, TableImg } from '/@/components/Table';
import { useListPage } from '/@/hooks/system/useListPage';
import { defHttp } from '/@/utils/http/axios';
import { reactive, onMounted, ref } from 'vue';
import { getToken } from '@/utils/auth'
import cola from 'cytoscape-cola';
import { e } from 'unocss';

cytoscape.use(cola);
const tableKey = ref(0);
//定义表格列
const columns: BasicColumn[] = [
    {
        title: '样本号',
        dataIndex: 'id',
        width: 80,
        align: 'left',
        resizable: true,
        sorter: {
            multiple: 1,
        },
    },
    {
        title: '样本预览',
        dataIndex: 'labelPath',
        width: 80,
        slots: { customRender: 'img' },
    },
    {
        title: '数据集',
        dataIndex: 'datasetName',
        width: 80,
        resizable: true,
    },
    {
        title: '样本标签',
        dataIndex: 'labelName',
        width: 120,
        resizable: true,
    },
    {
        title: '分辨率',
        dataIndex: 'resolution',
        width: 70,
        resizable: true,
    },
    {
        title: '样本尺寸',
        dataIndex: 'sampleSize',
        width: 140,
        resizable: true,
    },
    {
        title: '样本图像类型',
        dataIndex: 'imgType',
        width: 140,
        resizable: true,
        sorter: {
            multiple: 2,
        },
    },
    // {
    //     title: '标签文件路径',
    //     dataIndex: 'labelPath',
    //     sorter: {
    //         multiple: 3,
    //     },
    //     width: 120,
    //     resizable: true,
    // },
    // {
    //     title: '传感器类型',
    //     dataIndex: 'sensor',
    //     width: 100,
    //     resizable: true,
    // },
    // {
    //     title: '空间范围',
    //     dataIndex: 'bbox',
    //     width: 120,
    //     resizable: true,
    // },
    // {
    //     title: '时间范围',
    //     dataIndex: 'time',
    //     width: 120,
    //     resizable: true,
    // },
];
//定义表格查询字段
//表单搜索字段
// todo>>分辨率参数修正
const searchFormSchema: FormSchema[] = [
    {
        label: '样本编号', //显示label
        field: 'id', //查询字段
        component: 'JInput', //渲染的组件
        componentProps: {
            type: ""
        },
    },
    {
        label: '分辨率',
        field: 'resolution',
        component: 'JRangeNumber',
    },
    {
        label: '样本尺寸',
        field: 'sampleSize',
        component: 'JInput',
        defaultValue: '256', //设置默认值
        componentProps: {
            type: ""
        },
    },
    {
        label: '样本图像类型',
        field: 'imgType',
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
    // {
    //     label: '样本图像路径',
    //     field: 'imgPath',
    //     component: 'JInput',
    // },
    // {
    //     label: '标签路径',
    //     field: 'labelPath',
    //     component: 'JInput',
    // },
    {
        label: '空间范围',
        field: 'bbox',
        component: 'RectangleSelect',
        colProps: {
            span: 12,
        },

    },
    {
        label: '时间范围',
        field: 'time',
        component: 'RangePicker',
        componentProps: {
            //是否显示时间
            showTime: true,
            //日期格式化
            format: 'YYYY-MM-DD HH:mm:ss',
            //范围文本描述用集合
            placeholder: ['请选择开始日期时间', '请选择结束日期时间']
        },
        colProps: {
            span: 12,
        },
    },
    {
        label: '标签类名',
        field: 'labelName',
        component: 'InputSelect',
        colProps: {
            span: 20,
            offset: 100,
        },
    },
];
// 填写labelId后出现选项 general / exect
// 选择general后先查找得到关联的标签类别：展示为伸缩列表
// 用户从伸缩列表中选择所需的类别
// 根据所选的类别调用 general接口获取查询的样本元数据
// 展示样本元数据

let cy = null;
let general_query_param = {};
//ajax请求api接口
// TODO:修改检索逻辑，区分精确检索和补充样本检索
const exactListApi = (params) => {
    console.log("参数", params);
    // 处理参数
    if (params.bbox) {
        let array = params.bbox.split(',').map(part => parseFloat(part.trim()));
        params.bbox = {
            "ll": [array[1], array[0]],
            "lr": [array[1], array[2]],
            "ul": [array[3], array[0]],
            "ur": [array[3], array[2]]
        }
    }
    if (params.time) {
        let array = params.time.split(/GMT,?/).map(s => s.trim() + ' GMT').slice(0, -1).map(part => new Date(part.trim()).getTime());
        params.time_begin = array[0];
        params.time_end = array[1];
        delete params.time;
    }
    if (params.resolution) {
        let array = params.resolution.split(',');
        params.resolutione_begin = array[0];
        params.resolutione_end = array[1];
        delete params.resolution;
    }
    let true_labels = [];
    let associate_labels = [];
    console.log("处理前参数", params)
    if (params.labelName) {
        let labels = params.labelName.split(',');

        labels.forEach(label => {
            console.log(label)
            if (label.startsWith("k-")) {
                true_labels.push(label.substring(2)); // 去掉 "k-" 前缀并添加到 true_labels
            } else {
                associate_labels.push(label); // 直接添加到 associate_labels
            }
            console.log("true-label: ",true_labels)
        });

        let allLabels = true_labels.concat(associate_labels);

        highlightAndFocus(allLabels)
        params.labelId_MultiString = allLabels.join(',');
        // params.labelId = true_labels
        // 过滤掉准确样本
        params.labelId_Filter = true_labels.join(',');
        console.log(params.labelId_Filter)
        delete params.labelName;
    }
    // 浅拷贝
    general_query_param = { ...params }
    console.log("处理后参数", general_query_param)
    // generalListApi(params)
    tableKey.value = (tableKey.value+1)%10
    params.labelId_MultiString = true_labels.join(',');
    delete params.labelId_Filter;
    console.log("exact params:",params);
    return defHttp.get({ url: '/jeecg-sample/sample/query/exact', params })
};
const generalListApi = async (params) => {
    general_query_param.pageNo = params.pageNo
    general_query_param.pageSize = params.pageSize
    general_query_param.order = params.order
    params = general_query_param
    console.log("general 查询的参数",general_query_param)
    return defHttp.get({ url: '/jeecg-sample/sample/query/general', params })
};

const loadThumbnail = async (id) => {
    console.log(id);
    return defHttp.get({ url: 'http://10.3.1.128:8084/category/thumbnail/' + 422643 })
};

const loadData = async () => {
    try {
        let nodes = []
        let edges = []
        await defHttp.get({ url: '/jeecg-classify/classify/query/all' }).then((res) => {
            nodes = Object.entries(res.left).map(([key, value]) => ({
                data: {
                    id: key,
                    label: value,
                }
            }));
            res.right.forEach(ele => {
                edges.push({
                    data: {
                        source: ele.start,
                        target: ele.end,
                        label: ele.type,
                        weight: ele.weight,
                    }
                })
            });
        })
        let elements = {
            nodes: nodes,
            edges: edges,
        }
        // 如果 cy 实例已存在，仅更新其 elements
        if (cy) {
            cy.elements().remove();  // 清除当前图形的所有元素
            cy.add(elements);  // 添加新的元素
            cy.layout({
                name: 'cola',
                nodeSpacing: function (node) { return 10; },
                edgeLengthVal: function (edge) { return 50; },
                animate: true,
                fit: false,
                padding: 30
            }).run();  // 重新应用布局layout: { name: 'cose', fit: false },
        }
    } catch (error) {
        console.error('Failed to load graph data:', error);
    }
};


// 列表页面公共参数、方法
const { tableContext } = useListPage({
    designScope: 'basic-table-demo-ajax',
    tableProps: {
        title: '准确样本列表',
        api: exactListApi,
        columns: columns,
        formConfig: {
            schemas: searchFormSchema,
            labelAlign: 'left',
            //控制标题宽度占比
            labelCol: {
                xs: 4,
                sm: 4,
                md: 4,
                lg: 6,
                xl: 8,
                xxl: 8,
            },
            //控制组件宽度占比
            wrapperCol: {
                xs: 13,
                sm: 13,
                md: 14,
                lg: 20,
                xl: 30,
                xxl: 35,
            },
        },
        actionColumn: {
            width: 120,
        },
    },
});
// 列表页面公共参数、方法
const { tableContext:tableContext2 } = useListPage({
    designScope: 'basic-table-demo-ajax',
    tableProps: {
        title: '补充样本列表',
        api: generalListApi,
        columns: columns,
        useSearchForm: false,
        actionColumn: {
            width: 120,
        },
    },
});

//BasicTable绑定注册
const [registerTable, { getForm }] = tableContext;
const [registerTable2, {reload}] = tableContext2;

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

function handleEdit(record) {
    let { getFieldsValue } = getForm();
    console.log("查询的数据", getFieldsValue)
    console.log(record);
}



const cyContainer = ref(null);

onMounted(() => {
    cy = cytoscape({
        container: cyContainer.value,
        elements: {
        },
        style: [
            {
                selector: 'node',
                style: {
                    'content': 'data(label)',
                    'text-valign': 'center',
                    'color': '#fff',
                    'background-color': '#666',
                    'font-family': 'Arial, sans-serif',
                    'font-size': '6px'
                }
            },
            {
                selector: ':selected',
                style: {
                    'background-color': 'red',
                    'line-color': 'red',
                    'target-arrow-color': 'black',
                    'source-arrow-color': 'black',
                    'opacity': 1
                }
            },
            {
                selector: 'edge',
                style: {
                    'curve-style': 'bezier',
                    'label': 'data(label)',
                    'target-arrow-shape': 'triangle',
                    'line-color': '#ccc',
                    'target-arrow-color': '#ccc',
                    'width': 2,
                    'font-family': 'Arial, sans-serif',
                    'font-size': '6px'
                }
            },
            {
                selector: '[lineStyle="dotted"]',
                style: {
                    'line-style': 'dotted'
                }
            },
            {
                selector: '.highlighted',
                style: {
                    'background-color': 'red',
                    'line-color': 'red',
                    'target-arrow-color': 'red',
                    'border-width': 3,
                    'border-color': 'red'
                }
            }
        ],
        layout: { name: 'cose', fit: false },
        wheelSensitivity: 0.1,
    });
    cy.on('mouseover', 'node', (event) => {
        let node = event.target;
        console.log("触发鼠标悬浮事件");
        console.log({
            title: node.data('label'),
            text: `ID: ${node.id()}`,
            event: event.type
        })
        node.qtip({
            content: {
                title: node.data('label'),
                text: `ID: ${node.id()}`
            },
            position: {
                my: 'top left',
                at: 'top center'
            },
            show: {
                event: 'mouseover'
            },
            hide: {
                event: 'mouseout'
            },
            style: {
                classes: 'qtip-bootstrap',
                tip: {
                    width: 8,
                    height: 8
                }
            }
        });
    });
    loadData();
});
const highlightAndFocus = (ids) => {
    var eles = cy.elements();

    // Remove all highlights
    eles.removeClass('highlighted');

    // Filter nodes to highlight
    var toHighlight = cy.$(ids.map(id => `#${id}`).join(','));
    console.log("HighLight", toHighlight);
    toHighlight.addClass('highlighted');

    // Fit view to highlighted elements
    cy.fit(toHighlight, 50); // 50px padding
}

</script>

<style scoped>
.cytoscape {
    height: 100vh;
    width: 100vw;
    position: absolute;
    top: 0;
    left: 0;
}
</style>