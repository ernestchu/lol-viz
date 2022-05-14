<script setup>
import { useRoute } from 'vue-router'
const { summonerId } = useRoute().params
import { ref, reactive } from 'vue'

const matches = ref([])
const authFailed = ref(false)

const apiToken = import.meta.env.VITE_RIOT_API_KEY
const baseURL = 'https://na1.api.riotgames.com'
const baseURLBroad = 'https://americas.api.riotgames.com'
const summonersURI = `/lol/summoner/v4/summoners`
const matchesURI = `/lol/match/v5/matches/by-puuid`

fetch(`${baseURL}${summonersURI}/${summonerId}?api_key=${apiToken}`)
  .then(res => res.json())
  .then(data => {
    const { puuid } = data
    fetch(`${baseURLBroad}${matchesURI}/${puuid}/ids?api_key=${apiToken}&count=100`)
      .then(res => res.json())
      .then(data => {
        matches.value.push(...data)
      })
  })
  .catch(() => {
    authFailed.value = true
  })

</script>

<template>
  <template v-if="authFailed">
  <h2>Invalid or expired Riot API Token!</h2>
  Follow the <a href="https://github.com/ernestchu/lol-vis#setup-for-riot-api" target="_blank">instruction</a> to setup a valid token.
  </template>

  <ul v-if="matches">
    <li v-for="match in matches">{{ match }}</li>
  </ul>
</template>
