<!--
 * @Descripttion: 
 * @Author: junshi junshi@ssc-hn.com
 * @Date: 2022-11-17
 * @LastEditors: junshi junshi@ssc-hn.com
 * @LastEditTime: 2022-11-29
-->

<script lang="ts" setup>
import { onMounted, reactive, ref, unref, watch } from "vue";
import type { UnwrapRef } from "vue";
import { cloneDeep, isEmpty } from "lodash-es";
import useStorage from "./hooks/useStorage";
import { ICookieTableDataSource, ICookieTableColumn, LIST_KEY } from "./type";
import { DEFAULT_LIST } from "./config";

const isOpenSync = ref(true);
const isHTJProject = ref(true);
const isDevelop = ref(false);

const columns = ref<ICookieTableColumn[]>([
  {
    title: "id",
    dataIndex: "id",
    key: "id",
    align: "center",
  },
  {
    title: "from",
    dataIndex: "from",
    key: "from",
    align: "center",
  },
  {
    title: "cookie name",
    dataIndex: "cookieName",
    key: "cookieName",
    align: "center",
  },
  {
    title: "to",
    dataIndex: "to",
    key: "to",
    align: "center",
  },
  {
    title: "操作",
    dataIndex: "action",
    key: "action",
    align: "center",
  },
]);

const dataSource = ref<ICookieTableDataSource[]>(DEFAULT_LIST);

const editableData: UnwrapRef<Record<string, ICookieTableDataSource>> = reactive({});

const { updateStorage, getStorage, updateCookie } = useStorage();

onMounted(async () => {
  // 初始化开启同步状态
  const openSyncLocal = await getStorage("isOpenSync");

  console.log(openSyncLocal, "初始化开启同步状态");

  if (!isEmpty(openSyncLocal)) {
    isOpenSync.value = openSyncLocal.isOpenSync;
  }

  // 从 localStorage 初始化数据
  const storage = await getStorage();
  console.log(storage, " 从 localStorage 初始化数据");
  const domainList = !isEmpty(storage) ? (Object.values(storage[LIST_KEY]) as ICookieTableDataSource[]) : [];

  if (!isEmpty(domainList)) {
    dataSource.value = domainList;
  }

  // 更新 localStorage 和 cookie
  if (!isEmpty(unref(dataSource))) {
    updateStorage(dataSource.value);

    dataSource.value.forEach((item) => {
      updateCookie({
        from: item.from,
        to: item.to,
        cookieName: item.cookieName,
      });
    });
  }
});

watch(isOpenSync, async () => {
  await chrome.storage.local.set({ isOpenSync: isOpenSync.value });
});

function handleEdit(rowId: string) {
  editableData[rowId] = cloneDeep(dataSource.value.filter((item) => item.id === rowId)[0]);
}

async function handleSave(rowId: string) {
  Object.assign(dataSource.value.filter((item) => item.id === rowId)[0], editableData[rowId]);
  delete editableData[rowId];
  // 更新 localStorage
  updateStorage(dataSource.value);
}

function handleDelete(item) {
  console.log(item, "item");
  dataSource.value = dataSource.value.filter((i) => i !== item);
}

function handleCancel(rowId: string) {
  delete editableData[rowId];
}

function handleAdd() {
  const maxId = dataSource.value.length > 0 ? Math.max(...dataSource.value.map((item) => Number(item.id))) : 0;

  const addDataId = (maxId + 1).toString();

  dataSource.value.push({
    id: addDataId,
    from: "",
    cookieName: "",
    to: "localhost",
  });

  // 新增后空数据可以编辑
  handleEdit(addDataId);
}
// 简单做个环境替换
function onChange() {
  dataSource.value.map((res) => {
    res.from = isDevelop.value ? "https://datac-test.ssc-hn.com/" : "http://10.0.23.64:9145/";
    return {
      res,
    };
  });
}
</script>
<template>
  <a-table :columns="columns" :data-source="dataSource" :pagination="false">
    <template #bodyCell="{ column, text, record }">
      <!-- 可编辑行 -->
      <template v-if="['from', 'cookieName', 'to'].includes(column.dataIndex)">
        <div>
          <a-input v-if="editableData[record.id]" v-model:value="editableData[record.id][column.dataIndex]" />
          <span v-else>{{ text }}</span>
        </div>
      </template>

      <!-- 操作列 -->
      <template v-if="column.key === 'action'">
        <span v-if="editableData[record.id]">
          <a-button type="link" @click="handleSave(record.id)">保存</a-button>
          <a-button type="link" danger @click="handleCancel(record.id)">撤销</a-button>
        </span>
        <span v-else>
          <a-button type="text" @click="handleEdit(record.id)">编辑</a-button>
          <a-popconfirm title="确认删除？" @confirm="handleDelete(record)">
            <a-button type="text" danger>删除</a-button>
          </a-popconfirm>
        </span>
      </template>
    </template>
  </a-table>

  <a-row class="handle">
    <a-col :span="4">
      <a-button type="primary" @click="handleAdd">新增</a-button>
    </a-col>
  </a-row>

  <a-row class="handle">
    <a-col :span="8">
      是否开启同步：
      <a-switch v-model:checked="isOpenSync" checked-children="开" un-checked-children="关"></a-switch>
    </a-col>

    <a-col :span="8">
      是否为鸿泰吉项目：
      <a-switch
        v-model:checked="isHTJProject"
        @change="onChange"
        checked-children="是"
        un-checked-children="否"
      ></a-switch>
    </a-col>

    <a-col :span="8" v-if="isHTJProject">
      是否为测试环境：
      <a-switch
        v-model:checked="isDevelop"
        @change="onChange"
        checked-children="是"
        un-checked-children="否"
      ></a-switch>
    </a-col>
  </a-row>
</template>

<style>
.handle {
  margin-top: 1rem;
}
</style>
