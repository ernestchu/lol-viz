<script setup>
import { useRoute } from 'vue-router'
import { ref, reactive } from 'vue'
const { summonerId } = useRoute().params

const matches = ref([])
const authFailed = ref(false)
const summonerName = ref('')
let puuid = ''

const apiToken = import.meta.env.VITE_RIOT_API_KEY
const baseURL = 'https://na1.api.riotgames.com'
const baseURLBroad = 'https://americas.api.riotgames.com'
const summonersURI = '/lol/summoner/v4/summoners'
const matchesURI = '/lol/match/v5/matches/by-puuid'
const matchStatsURI = '/lol/match/v5/matches'

fetch(`${baseURL}${summonersURI}/${summonerId}?api_key=${apiToken}`)
  .then(res => res.json())
  .then(data => {
    summonerName.value = data.name
    puuid = data.puuid
    fetch(`${baseURLBroad}${matchesURI}/${puuid}/ids?api_key=${apiToken}`)
      .then(res => res.json())
      .then(data => {
        matches.value.push(...data)
      })
  })
  .catch(() => {
    authFailed.value = true
  })

const matchStats = reactive({})
function fetchMatchStats () {
  // matches.value.forEach(match => {
  matches.value.slice(0, 1).forEach(match => { // use a subarray to develop to prevent reaching the limit.
    fetch(`${baseURLBroad}${matchStatsURI}/${match}?api_key=${apiToken}`)
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
      .catch(() => {
        console.error('May have reached the rate limit.')
      })
  })
}

</script>

<template>
  <template v-if="authFailed">
  <h2>Invalid or expired Riot API Token!</h2>
  Follow the <a href="https://github.com/ernestchu/lol-vis#setup-for-riot-api" target="_blank">instruction</a> to setup a valid token.
  </template>

  <template v-if="matches.length">
    Summoner: {{ summonerName }} <br />
    <button @click="fetchMatchStats">Fetch match stats</button>
    (will sent 20 requests, note the rate limit of 20 req/sec, 100 req/min)
    <h3>Matches</h3>
    <ul>
      <li v-for="match in matches">
        <span
          :class="{ 'match-no': !!matchStats[match] }"
          @click="matchStats[match] ? matchStats[match].show = !matchStats[match].show : null"
        >{{ match }}</span>
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
.match-no {
  color: blue;
  cursor: pointer;
}
</style>
