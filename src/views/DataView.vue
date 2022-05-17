<script setup>
import { useRoute, useRouter } from 'vue-router'
import { ref } from 'vue'
import JsonViewer from 'vue-json-viewer'

const fileNotFound = ref(false)
const data = ref()
const dataURI = ref(useRoute().params.dataURI)
const dataURL = new URL(`../assets/sampleData/${dataURI.value}.json`, import.meta.url).href
fetch(dataURL)
  .then(res => res.json())
  .then(json => data.value = json)
  .catch(() => fileNotFound.value = true)

const router = useRouter()
function reloadWithDataURI () {
  window.location.href = `/lol-vis/data/${dataURI.value}` 
}
</script>

<template>
  <h2>{{ dataURI }}.json</h2>
  {{ dataURL }}
  <template v-if="fileNotFound">
    <h3>File not found, please try another file name:</h3>
    <form @submit.prevent="reloadWithDataURI">
      <input type="text" v-model="dataURI">
      <button>Try again</button>
    </form>
  </template>

  <JsonViewer v-if="data" :value="data" />
</template>
