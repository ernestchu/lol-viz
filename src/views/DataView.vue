<script setup>
import { useRoute, useRouter } from 'vue-router'
import { ref } from 'vue'
import JsonViewer from 'vue-json-viewer'

const fileNotFound = ref(false)
const data = ref()
const dataURI = ref(useRoute().params.dataURI)
const router = useRouter()
const expandDepth = ref(1)

function fetchWithDataURI() {
  fetch(`${import.meta.env.BASE_URL}sample-data/${dataURI.value}.json`)
    .then(res => res.json())
    .then(json => (data.value = json))
    .catch(() => (fileNotFound.value = true))
}
fetchWithDataURI()

function reloadWithDataURI() {
  router.push({ params: { dataURI: dataURI.value } })
  fileNotFound.value = false
  fetchWithDataURI()
}
</script>

<template>
  <h2>{{ dataURI }}.json</h2>
  <template v-if="fileNotFound">
    <h3>File not found, please try another file name:</h3>
    <form @submit.prevent="reloadWithDataURI">
      <input type="text" v-model="dataURI" />
      <button>Try again</button>
    </form>
  </template>

  <template v-else>
  <input type="range" v-model="expandDepth" min=1 max=7> Expand depth: {{ expandDepth }}
  <JsonViewer v-if="data" :value="data" :expand-depth=expandDepth :key="expandDepth" />
  </template>
</template>
