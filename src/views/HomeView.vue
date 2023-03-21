<script setup lang="ts">
import { reactive } from 'vue'
import axios from 'axios'

const state = reactive({ rssUrl: '' })
async function handleAddRss() {
  console.log('state: ', state)
  const res = await axios.post('http://localhost:3000/qbt/addRssUrl', { rssUrl: state.rssUrl })
  console.log('res: ', res)
}
</script>

<template>
  <main>
    <v-text-field label="RSS Url" v-model="state.rssUrl"></v-text-field>
    <v-btn @click="handleAddRss"> 订阅这条RSS </v-btn>
  </main>
</template>
