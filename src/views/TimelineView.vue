<script setup>
import { useRoute } from 'vue-router'
import { ref, reactive } from 'vue'
import JsonViewer from 'vue-json-viewer'

const timeline = reactive({})
const authFailed = ref(false)

const proxyHost = import.meta.env.VITE_PROXY_HOST
const timelineURI = `/lol/match/v5/matches/${
  useRoute().params.matchId
}/timeline`

fetch(`${proxyHost}${timelineURI}`)
  .then(res => res.json())
  .then(data => {
    Object.assign(timeline, data)
  })
  .catch(() => {
    authFailed.value = true
  })
</script>

<template>
  <template v-if="authFailed">
    <h2>Invalid or expired Riot API Token!</h2>
    Follow the
    <a
      href="https://github.com/ernestchu/lol-vis#setup-for-riot-api"
      target="_blank"
      >instruction</a
    >
    to setup a valid token.
  </template>

  MatchId: {{ useRoute().params.matchId }}
  <h3>Timeline</h3>
  <JsonViewer :value="timeline" />
</template>
