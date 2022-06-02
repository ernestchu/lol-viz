<script setup>
import { useRoute } from 'vue-router'
import { ref, reactive } from 'vue'
const { summonerId } = useRoute().params

const matches = ref([])
const showErrorMsg = ref(false)
const summonerName = ref('')
let puuid = ''

const proxyHost = import.meta.env.VITE_PROXY_HOST
const summonersURI = '/lol/summoner/v4/summoners'
const matchesURI = '/lol/match/v5/matches/by-puuid'
const matchesQueryString = '?queue=420' // solo q
const matchStatsURI = '/lol/match/v5/matches'

fetch(`${proxyHost}${summonersURI}/${summonerId}`)
  .then(res => res.json())
  .then(data => {
    summonerName.value = data.name
    puuid = data.puuid
    fetch(`${proxyHost}${matchesURI}/${puuid}/ids${matchesQueryString}`)
      .then(res => res.json())
      .then(data => {
        matches.value.push(...data)
      })
  })
  .catch((e) => {
    console.error(e)
    showErrorMsg.value = true
  })

const matchStats = reactive({})
const nMatches = ref(20)
function fetchMatchStats() {
  // matches.value.forEach(match => {
  matches.value.slice(0, nMatches.value).forEach(match => {
    // use a subarray to develop to prevent reaching the limit.
    fetch(`${proxyHost}${matchStatsURI}/${match}`)
      .then(res => res.json())
      .then(data => {
        // console.log(data) // this is very rich
        const index = data.metadata.participants.indexOf(puuid)
        const stats = data.info.participants[index]
        delete stats.challenges
        delete stats.perks
        stats.show = false
        matchStats[match] = stats
      })
      .catch((e) => {
        console.error(e, 'May have reached the rate limit.')
        showErrorMsg.value = true
      })
  })
}
</script>

<template>
  <template v-if="showErrorMsg">
    <div>
      <h2>The page has encountered an error!</h2>
      See the console for further informations.
    </div>
  </template>

  <template v-if="matches.length">
    Summoner: {{ summonerName }} <br />
    <input v-model="nMatches" type="range" min="1" max="20" />{{
      nMatches
    }}
    match(es) per fetch<br />
    <button @click="fetchMatchStats">Fetch match stats</button>
    (will sent up to 20 requests, note the rate limit of 20 req/sec, 100
    req/min)
    <h3>Matches</h3>
    <ul>
      <li v-for="match in matches">
        <span
          :class="{ 'match-no': !!matchStats[match] }"
          @click="
            matchStats[match]
              ? (matchStats[match].show = !matchStats[match].show)
              : null
          "
          >{{ match }}</span
        >
        <ul v-if="matchStats[match] && matchStats[match].show">
          <RouterLink :to="{ name: 'timeline', params: { matchId: match } }">
            View match timeline
          </RouterLink>
          <template v-for="(statValue, statKey) in matchStats[match]">
            <li v-if="statKey != 'show'">{{ statKey }}: {{ statValue }}</li>
          </template>
        </ul>
      </li>
    </ul>
  </template>
</template>

<style scoped>
button {
  color: black;
}
.match-no {
  color: #B3944C;
  cursor: pointer;
}
</style>
