<script setup>
import { ref, reactive } from 'vue'

const apiToken = ref(import.meta.env.VITE_RIOT_API_KEY)
const testData = reactive({})

const baseURL = 'https://na1.api.riotgames.com'
const targetURI = '/lol/league/v4/challengerleagues/by-queue/RANKED_SOLO_5x5'

function testFetch () {
  fetch(`${baseURL}${targetURI}?api_key=${apiToken.value}`)
  .then(res => res.json())
  .then(data => {
    Object.assign(testData, data)
    testData.entries.forEach(entry => {
      entry.show = false
    })
  })
}
</script>

<template>
<h2>This page tests some Riot APIs</h2>
<label for="api-token">Riot API Token: </label>
<input type="password" v-model="apiToken" id="api-token">
<button @click="testFetch">fetch</button> <br />
<a href="https://developer.riotgames.com" target="_blank">Regenerate token</a>
<h3>Fetched data</h3>
<ul v-if="testData">
  <li v-for="(v, k) in testData">
    <ul v-if="k === 'entries'">
      <li v-for="entry in v">
        <span
          class="summoner-name"
          @click="entry.show = !entry.show"
        >
          {{ entry.summonerName }}
        </span>
        <ul v-if="entry.show">
          <template v-for="(entryV, entryK) in entry">
            <li v-if="entryK != 'show'">
              {{ entryK }}: {{ entryV }}
            </li>
          </template>
        </ul>
      </li>
    </ul>
    <template v-else>{{ k }}: {{ v }}</template>
  </li>
</ul>
</template>

<style scoped>
.summoner-name {
  color: blue;
  cursor: pointer;
}
</style>
