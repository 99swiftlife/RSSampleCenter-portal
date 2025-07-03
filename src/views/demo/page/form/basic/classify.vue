<template>
    <!-- 标签查询输入框和按钮 -->
    <div style="text-align: center; margin-bottom: 20px;">
        <a-input v-model:value="inputTag" placeholder="输入标签名称" style="width: 300px;" />
        <a-button type="primary" @click="handleTagSearch">查询关联标签</a-button>
    </div>

    <!-- 最短路查询输入框和按钮 -->
    <div style="text-align: center; margin-bottom: 20px;">
        <a-input v-model:value="startNode" placeholder="输入起始节点" style="width: 150px; margin-right: 10px;" />
        <a-input v-model:value="endNode" placeholder="输入终止节点" style="width: 150px; margin-right: 10px;" />
        <a-button type="primary" @click="handleShortestPathSearch">检索最短路径</a-button>
    </div>

    <div style="text-align: center; margin-bottom: 20px;">
        <a-input v-model:value="subgraphInput" placeholder="输入节点名称，使用逗号分隔" style="width: 300px; margin-right: 10px;" />
        <a-button type="primary" @click="handleSubgraphLoad">加载子图</a-button>
        <a-button type="primary" @click="addTagMapping">添加标签映射</a-button>
        <a-button type="danger" @click="clearHistory">清除</a-button>
    </div>

    <!-- Cytoscape 容器 -->
    <div ref="cyContainer"
        style="width: 80%; height: 80vh; margin: 0 auto; background-color: white; border: 3px solid #decba4; box-shadow: 0 4px 8px rgba(0,0,0,0.1); border-radius: 10px;">
    </div>


    <!-- 弹出的选择框，用于当有多个校正标签时让用户选择 -->
    <a-modal v-model:visible="isModalVisible" title="选择一个标签" @ok="handleSelectTag">
        <a-select v-model="selectedTag" placeholder="选择标签">
            <a-select-option v-for="(name, id) in tagOptions" :key="id" :value="id">
                {{ name }}
            </a-select-option>
        </a-select>
    </a-modal>
</template>

<script lang="ts" setup>
import cytoscape from 'cytoscape';
import cyqtip from 'cytoscape-qtip';
import cola from 'cytoscape-cola';
import { defHttp } from '/@/utils/http/axios';
import { ref, onMounted } from 'vue';
import cytoscapeCoseBilkent from 'cytoscape-cose-bilkent';

cytoscape.use(cytoscapeCoseBilkent);

// 变量定义
const inputTag = ref('');           // 输入的标签名称
const isModalVisible = ref(false);  // 是否显示选择框
const selectedTag = ref('');        // 用户选择的校正标签 ID
const tagOptions = ref({});         // 校正返回的标签选项
const cyContainer = ref(null);      // Cytoscape 容器引用
let cy = null;                      // Cytoscape 实例
const startNode = ref('');  // 存储起始节点
const endNode = ref('');    // 存储终止节点
let originalNodeStyles = {};  // 用于保存节点原始样式
const subgraphInput = ref('');  // 用于存储子图加载的输入框内容
const historyNodeList = ref([]);  // 全局历史输入节点列表


// 异步加载数据
const loadData = async () => {
    try {
        let nodes = [];
        let edges = [];
        const res = await defHttp.get({ url: '/jeecg-classify/classify/query/all' });

        // 处理节点数据
        nodes = Object.entries(res.left).map(([key, value]) => ({
            data: { id: key, label: value }
        }));

        // 处理边数据
        edges = res.right.map(ele => ({
            data: {
                source: ele.start,
                target: ele.end,
                label: ele.type,
                weight: ele.weight,
            }
        }));

        // 更新 Cytoscape 元素
        if (cy) {
            cy.elements().remove();  // 移除所有现有元素
            cy.add({ nodes, edges });  // 添加新元素
            cy.fit(cy.elements(), 50);  // 适应视图

            // 执行知识图谱的布局
            cy.layout({
                name: 'cose-bilkent',
                idealEdgeLength: 100,   // 边的理想长度，关联越紧密的边越短
                nodeRepulsion: 4500,    // 节点之间的排斥力，减少节点重叠
                edgeElasticity: 0.45,   // 边的弹性，较低的值会让关联紧密的节点更靠近
                nestingFactor: 0.1,     // 控制节点分布的松紧程度
                gravity: 0.25,          // 全局引力，数值较低时图形更发散
                numIter: 2500,          // 迭代次数，保证布局收敛
                animate: 'end',         // 布局结束时动画显示
            }).run();

        }
    } catch (error) {
        console.error('Failed to load graph data:', error);
    }
};

// 调用校正接口进行标签校正
const correctTag = async (inputTag: string) => {
    try {
        const res = await defHttp.get({ url: `/jeecg-classify/classify/query/${inputTag}` })
        return res || {};
    } catch (error) {
        message.error('标签校正失败');
        return {};
    }
};
// 处理查询关联标签
const handleTagSearch = async () => {
    const correctedResult = await correctTag(inputTag.value);

    if (Object.keys(correctedResult).length === 0) {
        message.error('未找到匹配的标签');
        return;
    }

    // 如果有多个校正结果，弹出选择框
    if (Object.keys(correctedResult).length > 1) {
        tagOptions.value = correctedResult; // 填充选择框的选项
        isModalVisible.value = true;        // 显示选择框
    } else {
        const correctedId = Object.keys(correctedResult)[0];
        fetchAndHighlightAssociatedTags(correctedId);
    }
};

// 处理选择标签后的操作
const handleSelectTag = () => {
    if (selectedTag.value) {
        fetchAndHighlightAssociatedTags(selectedTag.value);
    }
    isModalVisible.value = false;
};

// 调用查询关联标签接口并高亮显示
const fetchAndHighlightAssociatedTags = async (tagId) => {
    try {
        const associatedResult = await defHttp.get({ url: `/jeecg-classify/classify/query/associated/${tagId}` });
        const associatedTags = Object.entries(associatedResult).map(([id, name]) => ({ id, name }));
        highlightTags(tagId, associatedTags);
    } catch (error) {
        message.error('查询关联标签失败');
    }
};

// 高亮标签，分为校正结果和关联标签
const highlightTags = (correctedId, associatedTags) => {
    cy.elements().removeClass('highlighted corrected'); // 移除所有高亮

    // 高亮校正结果
    const correctedNode = cy.getElementById(correctedId);
    correctedNode.addClass('corrected');

    // 高亮关联标签
    associatedTags.forEach(tag => {
        const associatedNode = cy.getElementById(tag.id);
        associatedNode.addClass('highlighted');
    });

    cy.fit(cy.elements('.highlighted-node, .corrected'), 50); // 聚焦高亮元素
};

// 调用最短路径接口并高亮显示路径
const handleShortestPathSearch = async () => {

    const res = await defHttp.get({
        url: `/jeecg-classify/classify/shortest-path`,
        params: { startNodeName: startNode.value, endNodeName: endNode.value }
    });

    const relationships = res || [];

    if (relationships.length === 0) {
        message.error('未找到最短路径');
        return;
    }

    // 提取路径上的节点和边
    const nodesToHighlight = new Set();
    relationships.forEach(rel => {
        nodesToHighlight.add(rel.start);
        nodesToHighlight.add(rel.end);
    });

    // 高亮路径上的节点和关系
    highlightAndFocus(Array.from(nodesToHighlight), relationships);

};

// 调用加载子图接口并更新Cytoscape中的图形
const handleSubgraphLoad = async (historyNodeList, newNodeList) => {
    try {
        // 合并 historyNodeList 和 newNodeList，确保去重
        const allNodesSet = new Set([...historyNodeList, ...newNodeList]);

        // 使用 Map 来存储节点，避免重复
        const nodeMap = new Map();
        for (const name of allNodesSet) {
            const nodeRes = await defHttp.get({
                url: `/jeecg-classify/classify/query/${name}`,
            });

            // 假设 nodeRes 返回格式 { id: 节点ID, name: 节点名称 }
            Object.entries(nodeRes).forEach(([id, label]) => {
                nodeMap.set(id, { id, label });
            });
        }

        // 获取边数据，调用子图接口获取与节点相关的边信息
        const res = await defHttp.get({
            url: '/jeecg-classify/classify/query/sub',
            params: { nameList: Array.from(allNodesSet).join(',') }
        });

        const edges = res.map(rel => ({
            data: {
                source: rel.start,
                target: rel.end,
                label: rel.type,
                weight: rel.weight,
                startName: rel.startName,
                endName: rel.endName,
            }
        }));

        // 提取出涉及到的节点 ID 集合，并确保其类型为字符串
        const involvedNodeIds = new Set();
        edges.forEach(edge => {
            involvedNodeIds.add(String(edge.data.source));  // 将 source 转换为字符串
            involvedNodeIds.add(String(edge.data.target));  // 将 target 转换为字符串
        });

        // 过滤孤立节点，只保留与边关联的节点
        const nodes = Array.from(nodeMap.values())
            .filter(node => involvedNodeIds.has(String(node.id)))  // 确保 node.id 和边的 source/target 一致
            .map(node => ({
                data: { id: String(node.id), label: node.label }
            }));

        // 更新 Cytoscape 图
        if (cy) {
            cy.elements().remove();  // 移除所有现有元素
            cy.add({ nodes, edges });  // 添加新元素
            cy.fit(cy.elements(), 50);  // 适应视图

            // 高亮逻辑：仅当 historyNodeList 不为空时，高亮 newNodeList 中的节点
       

            // 清除 historyNodeList 中的节点高亮显示
            historyNodeList.forEach(label => {
                cy.elements(`node[label="${label}"]`).removeClass('highlighted-node');
            });

            console.log("his",historyNodeList)
            if (historyNodeList.length > 0) {
                newNodeList.forEach(label => {
                    cy.elements(`node[label="${label}"]`).addClass('highlighted-node');
                });

                // 高亮新增节点相关的边（source 或 target 为新增节点的边）
                edges.forEach(edge => {
                    const sourceInNewNodes = newNodeList.includes(String(edge.data.startName));
                    const targetInNewNodes = newNodeList.includes(String(edge.data.endName));

                    // 如果边的 source 或 target 在 newNodeList 中，则高亮该边
                    if (sourceInNewNodes || targetInNewNodes) {
                        cy.elements(`edge[source="${edge.data.source}"][target="${edge.data.target}"]`).addClass('highlighted-edge');
                    }
                });
            }

            // 应用布局
            cy.layout({
                name: 'cose-bilkent',
                idealEdgeLength: 100,
                nodeRepulsion: 4500,
                edgeElasticity: 0.45,
                nestingFactor: 0.1,
                gravity: 0.25,
                numIter: 2500,
                animate: 'end',
            }).run();
        }
    } catch (error) {
        message.error('加载子图失败');
    }
};


// 清除历史列表
const clearHistory = () => {
    historyNodeList.value = [];  // 清空列表内容
};

// 添加标签映射按钮点击事件处理
const addTagMapping = async () => {
    let currentNodeList = subgraphInput.value.split(',').map(item => item.trim());

    // 当输入为 "ALL" 时，调用 /query/all 接口获取所有节点
    if (subgraphInput.value.toUpperCase() === 'ALL') {
        const allNodesRes = await defHttp.get({ url: '/jeecg-classify/classify/query/all' });
        currentNodeList = Object.values(allNodesRes.left);  // 获取所有节点的名称列表
    }

    // 比较去重，找到新增节点 newNodeList
    const newNodeList = currentNodeList.filter(node => !historyNodeList.value.includes(node));

    // 调用 handleSubgraphLoad，传递 historyNodeList 和 newNodeList
    handleSubgraphLoad(historyNodeList.value, newNodeList);

    // 将新节点添加到历史列表
    historyNodeList.value.push(...newNodeList);
};


// 初始化 Cytoscape
onMounted(() => {
    cy = cytoscape({
        container: cyContainer.value,
        elements: {},  // 初始化空元素
        style: [       // 配置节点和边样式
            {
                selector: 'edge',  // 针对边的样式
                style: {
                    'label': 'data(label)',  // 显示边的标签
                    'font-size': '10px',  // 调整边标签的字体大小，减少重叠
                    'color': '#333',  // 边标签的文字颜色
                    'text-background-opacity': 0,           // 标签背景透明
                    'text-background-color': '#ffffff',     // 标签背景颜色（无效，因为透明度为0）
                    'text-background-padding': '3px',
                    'text-background-shape': 'roundrectangle',  // 文字背景为圆角矩形
                    'text-rotation': 'autorotate',        // 标签沿边旋转
                    'text-margin-y': -10,  // 将文字稍微向上移动，避免与边重叠
                    'curve-style': 'bezier',  // 使用贝塞尔曲线减少边的重叠
                    'width': 2,  // 边的宽度
                    'line-color': '#aaa',  // 边的颜色
                    'target-arrow-shape': 'triangle',  // 设置箭头形状
                    'target-arrow-color': '#aaa',  // 箭头颜色
                }
            },
            {
                selector: 'node',  // 针对节点的样式
                style: {
                    'content': 'data(label)',  // 节点标签
                    'font-size': '12px',  // 调整节点标签字体大小
                    'text-valign': 'center',  // 文字垂直对齐
                    'text-halign': 'center',  // 文字水平对齐
                    'background-color': '#88c0d0',  // 节点的背景颜色
                    'border-width': 2,  // 节点的边框宽度
                    'border-color': '#4c566a',  // 节点边框的颜色
                    'width': 60,                     // 设置节点的宽度
                    'height': 60,                    // 设置节点的高度
                    'text-outline-width': 2,  // 文本描边宽度
                    'text-outline-color': '#fff',  // 文本描边颜色
                }
            },
            {
                selector: ':selected',
                style: {
                    'background-color': '#bf616a',  // 选中节点背景色
                    'line-color': '#bf616a',        // 选中边颜色
                    'target-arrow-color': '#bf616a',
                    'source-arrow-color': '#bf616a',
                }
            },
            {
                selector: '.corrected',
                style: {
                    'background-color': '#FFD700', // 校正结果的高亮颜色为金色
                    'line-color': '#FFD700',
                    'border-width': 3,
                }
            },
            {
                selector: '.highlighted-node',
                style: {
                    'background-color': '#CC908A',  // 节点高亮颜色
                    'border-width': 3,
                    'border-color': '#CC908A',      // 节点边框颜色
                }
            },
            {
                selector: '.highlighted-edge',
                style: {
                    'line-color': '#CC908A',        // 边的高亮颜色
                    'width': 4,                     // 高亮边的宽度
                    'target-arrow-color': '#CC908A', // 边箭头颜色
                    'target-arrow-shape': 'triangle', // 边箭头形状
                }
            }

        ],
        layout: { name: 'cose', fit: false },  // 初始布局
        wheelSensitivity: 0.1,  // 缩放灵敏度
    });

    // 添加鼠标悬浮事件，显示节点信息并改变样式
    cy.on('mouseover', 'node', (event) => {
        const node = event.target;

        // 保存当前样式
        originalNodeStyles[node.id()] = {
            'background-color': node.style('background-color'),
            'border-color': node.style('border-color'),
            'border-width': node.style('border-width')
        };

        // 鼠标悬停时修改样式
        node.style({
            'background-color': '#ebcb8b',  // 悬浮时背景色
            'border-color': '#ebcb8b',      // 悬浮时边框颜色
            'border-width': 4               // 悬浮时边框宽度
        });
    });

    // 鼠标移出时恢复之前的样式
    cy.on('mouseout', 'node', (event) => {
        const node = event.target;

        // 恢复之前保存的样式
        if (originalNodeStyles[node.id()]) {
            node.style({
                'background-color': originalNodeStyles[node.id()]['background-color'],
                'border-color': originalNodeStyles[node.id()]['border-color'],
                'border-width': originalNodeStyles[node.id()]['border-width']
            });
        }
    });

    // 加载数据并展示
    loadData();
});

// 高亮并聚焦指定节点和关系
const highlightAndFocus = (nodeIds, relationships) => {
    // 移除所有节点和边的高亮
    cy.elements().removeClass('highlighted-node highlighted-edge');

    // 高亮路径上的节点
    const nodes = cy.$(nodeIds.map(id => `#${id}`).join(','));
    nodes.addClass('highlighted-node');  // 应用节点高亮样式

    // 高亮路径上的边
    relationships.forEach(rel => {
        const edge = cy.edges(`[source = "${rel.start}"][target = "${rel.end}"]`);
        if (edge) {
            edge.addClass('highlighted-edge');  // 应用边高亮样式
        }
    });

    // 聚焦高亮元素
    cy.fit(cy.elements('.highlighted-node, .highlighted-edge'), 50);
};

</script>

<style scoped>
/* 适应整个窗口的 Cytoscape 样式 */
.cytoscape {
    height: 80vh;
    width: 80vw;
    margin: 0 auto;
    border-radius: 15px;
    background-color: #eceff4;
    /* 淡色背景 */
    box-shadow: 0px 4px 12px rgba(0, 0, 0, 0.1);
}
</style>