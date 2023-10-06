<script setup lang="ts">
import { reactive } from 'vue'

import axios from 'axios'
import { ElMessage } from 'element-plus'
import { wildcardPattern } from 'wildcard-regex'

const TAG_TYPE_OF_STATUS = {
  READY: 'success',
  UNREADY: 'warning'
}

const form = reactive({
  rssUrl: '',
  notContainRule: '',
  downLoadPath: ''
})

const state = reactive({
  tableData: [],
  qbtInfo: {} as any,
  qbtConfigForm: {
    host: '',
    username: '',
    password: '',
    defaultDownLoadPath: ''
  },
  intervalInstance: null as any
})
async function handleAddRss() {
  const { rssUrl } = form
  if (!rssUrl) {
    ElMessage({
      message: 'RSS Url为空',
      type: 'warning'
    })
    return
  }
  const { data } =
    (await axios.post('/qbt/addRssUrl', {
      ...form
    })) || {}

  const { success } = data || {}

  if (success) {
    ElMessage({
      message: '订阅成功',
      type: 'success'
    })
  }
}

async function handleRequestRss() {
  const { rssUrl } = form
  if (!rssUrl) {
    ElMessage({
      message: 'RSS Url为空',
      type: 'warning'
    })
    return
  }

  const { data } =
    (await axios.post('/qbt/getRssDetail', {
      rssUrl
    })) || {}

  const items = data?.rss?.channel?.item

  state.tableData = items?.map?.((item: any) => {
    const { title } = item
    return { title: title?._text }
  }) || [{ title: items?.title?._text }]
}

function myWildcardRegExp(wildcard: string) {
  const pattern = wildcardPattern(wildcard)

  const result = new RegExp(pattern.substr(1, pattern.length - 2))

  return result
  // return wildcardRegExp(wildcard)
}

function matchesExpression(articleTitle: string, expression: string) {
  // const QRegularExpression whitespace {u"\\s+"_qs};

  if (!expression) {
    // A regex of the form "expr|" will always match, so do the same for wildcards
    return true
  }

  // if (useRegex)
  // {
  // }

  // Only match if every wildcard token (separated by spaces) is present in the article name.
  // Order of wildcard tokens is unimportant (if order is important, they should have used *).
  const wildcards = expression.split(' ')

  for (const wildcard of wildcards) {
    const regex = myWildcardRegExp(wildcard)

    if (!regex.test(articleTitle)) {
      return false
    }
  }

  return true
}

const matchesMustNotContainExpression = (title: string): boolean => {
  const { notContainRule } = form
  const rules = notContainRule.split('|')

  const result = !rules.some((expression) => {
    return matchesExpression(title, expression)
  })

  return result
}

const tableRowClassName = ({ row: { title } }: { row: { title: string } }) => {
  if (!matchesMustNotContainExpression(title)) {
    return 'error-row'
  }
  return 'success-row'
}

async function getQbtStatus() {
  const { data } = (await axios.get('/qbt/getQbtStatus')) || {}

  return data
}

async function initQbtInfo() {
  state.qbtInfo = await getQbtStatus()
}

async function getQbtServerConfig() {
  const { data } = (await axios.get('/qbt/getQbtServerConfig')) || {}

  return data
}

async function initQbtConfig() {
  state.qbtConfigForm = (await getQbtServerConfig()).serverInfo
}

async function syncQbt() {
  initQbtInfo()

  if (state.intervalInstance) {
    clearInterval(state.intervalInstance)
  }
  state.intervalInstance = setInterval(async () => {
    initQbtInfo()
  }, 10000)
}

async function getMikanInfo() {
  const { data } = (await axios.get('/mikan/test')) || {}

  return data
}

async function handleSetServerConfig() {
  const { data } =
    (await axios.post('/qbt/setServerConfig', {
      setting: {
        serverInfo: {
          ...state.qbtConfigForm
        }
      }
    })) || {}

  if (data.success) {
    ElMessage.success('更新配置成功')
    // location.reload()
  }

  init()
}

async function initDefaultConfig() {
  const { data } = (await axios.get('/qbt/getDefaultConfig')) || {}
  form.notContainRule = data?.defaultNotContainRule
  form.downLoadPath = data?.defaultDownLoadPath
}

async function init() {
  getMikanInfo()
  syncQbt()
  initQbtConfig()
}

init()
initDefaultConfig()
</script>

<template>
  <div class="main">
    <div class="tag-list">
      <el-tag :type="TAG_TYPE_OF_STATUS[state.qbtInfo?.status] || ''">
        {{ state.qbtInfo.status }}
      </el-tag>
      <el-tag> VERSION: {{ state.qbtInfo.version }} </el-tag>
    </div>

    <hr />
    <el-form :model="state.qbtConfigForm" label-width="120px">
      <el-form-item label="qbit webUI 地址">
        <el-input v-model="state.qbtConfigForm.host" clearable />
      </el-form-item>
      <el-form-item label="用户名">
        <el-input v-model="state.qbtConfigForm.username" clearable />
      </el-form-item>
      <el-form-item label="密码">
        <el-input v-model="state.qbtConfigForm.password" clearable show-password />
      </el-form-item>
      <el-form-item label="默认下载地址">
        <el-input v-model="state.qbtConfigForm.defaultDownLoadPath" clearable />
      </el-form-item>
      <el-form-item>
        <el-button @click="handleSetServerConfig">更新 qbittorrent 代理服务设置</el-button>
      </el-form-item>
    </el-form>
    <hr />

    <el-form :model="form" label-width="120px">
      <el-form-item label="RSS Url">
        <el-input v-model="form.rssUrl" clearable />
      </el-form-item>
      <el-form-item label="不包含规则">
        <el-input v-model="form.notContainRule" clearable />
      </el-form-item>
      <el-form-item label="下载地址">
        <el-input v-model="form.downLoadPath" clearable />
      </el-form-item>
      <el-form-item>
        <el-button @click="handleRequestRss">解析这条RSS</el-button>
        <el-button @click="handleAddRss"> 订阅这条RSS </el-button>
      </el-form-item>
    </el-form>
  </div>
  <div class="main chapter-list" v-if="state.tableData.length">
    <el-table :row-class-name="tableRowClassName" :data="state.tableData" style="width: 100%">
      <el-table-column prop="title" label="title" />
    </el-table>
  </div>
</template>
<style lang="scss" scoped>
::v-deep .el-table .error-row {
  --el-table-tr-bg-color: var(--el-color-error-light-9);
}
::v-deep .el-table .success-row {
  --el-table-tr-bg-color: var(--el-color-success-light-9);
}
.main {
  align-self: center;
}
.chapter-list {
  margin-left: 30px;
  height: 100%;
  overflow: auto;
  -ms-overflow-style: none;
  overflow: -moz-scrollbars-none;
}
.chapter-list::-webkit-scrollbar {
  width: 0 !important;
}

.tag-list {
  display: flex;
  & > * {
    margin-right: 10px;
  }
}
hr {
  margin: 10px 0;
}
</style>
