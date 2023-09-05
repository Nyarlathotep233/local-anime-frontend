<script setup lang="ts">
import { reactive } from 'vue'
import axios from 'axios'
import { ElMessage } from 'element-plus'
import { wildcardPattern } from 'wildcard-regex'

const form = reactive({
  rssUrl: '',
  notContainRule: '720|合集|繁体|JPTC|繁日|\\d+-\\d+|BIG5|MKV'
})

const state = reactive({
  tableData: []
})
async function handleAddRss() {
  const { rssUrl, notContainRule } = form
  if (!rssUrl) {
    ElMessage({
      message: 'RSS Url为空',
      type: 'warning'
    })
    return
  }
  const { data } =
    (await axios.post('http://localhost:3000/qbt/addRssUrl', {
      rssUrl,
      notContainRule
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
    (await axios.post('http://localhost:3000/qbt/getRssDetail', {
      rssUrl
    })) || {}

  const items = data?.rss?.channel?.item
  console.log('items: ', items)

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
    return ''
  }
  return 'success-row'
}
</script>

<template>
  <div class="main">
    <el-form :model="form" label-width="120px">
      <el-form-item label="RSS Url">
        <el-input v-model="form.rssUrl" clearable />
      </el-form-item>
      <el-form-item label="不包含规则">
        <el-input v-model="form.notContainRule" clearable />
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
<style>
.el-table .warning-row {
  --el-table-tr-bg-color: var(--el-color-warning-light-9);
}
.el-table .success-row {
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
</style>
