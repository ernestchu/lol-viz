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
  var champList = {};
  var updatedChampList = [];
  matches.value.slice(0, nMatches.value).forEach(match => {
    // use a subarray to develop to prevent reaching the limit.
    fetch(`${proxyHost}${matchStatsURI}/${match}`)
      .then(res => res.json())
      .then(data => {
        //console.log(data) // this is very rich
        const index = data.metadata.participants.indexOf(puuid)
        const stats = data.info.participants[index]
        var i = 0
        if(stats.championName in champList){
          updatedChampList[champList[stats.championName]].Games++;
          updatedChampList[champList[stats.championName]].Kills += stats.kills;
          updatedChampList[champList[stats.championName]].Deaths += stats.deaths;
          updatedChampList[champList[stats.championName]].Assists += stats.assists;
          updatedChampList[champList[stats.championName]].Damage += stats.totalDamageDealtToChampions;
          if(stats.win==true){
            updatedChampList[champList[stats.championName]].Wins++;
          }
          updatedChampList[champList[stats.championName]].WinRate = Math.floor((updatedChampList[champList[stats.championName]].Wins/updatedChampList[champList[stats.championName]].Games) * 100) + '%';
        }else{
          champList[stats.championName] = updatedChampList.length;
          updatedChampList.push({"name":stats.championName,"Games":1,"Wins":0,"Kills":0,"Deaths":0,"Assists":0,"Damage":0,"WinRate":0});
          updatedChampList[champList[stats.championName]].Kills += stats.kills;
          updatedChampList[champList[stats.championName]].Deaths += stats.deaths;
          updatedChampList[champList[stats.championName]].Assists += stats.assists;
          updatedChampList[champList[stats.championName]].Damage += stats.totalDamageDealtToChampions;
          if(stats.win==true){
            updatedChampList[champList[stats.championName]].Wins++;
          }
          updatedChampList[champList[stats.championName]].WinRate = Math.floor((updatedChampList[champList[stats.championName]].Wins/updatedChampList[champList[stats.championName]].Games) * 100) + '%';
        }
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
  //console.log(updatedChampList);
  visualization(updatedChampList);
}
function visualization(champList){
  console.log(champList);
  console.log(champList[0]); // Why shows undefined?
  const margin = {top: 10, right: 30, bottom: 20, left: 50},
    width = 460 - margin.left - margin.right,
    height = 360 - margin.top - margin.bottom;
  const svg = d3.select("#my_dataviz")
    .append("svg")
      .attr("width", width + margin.left + margin.right)
      .attr("height", height + margin.top + margin.bottom)
    .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

  champList.forEach(object =>{
    const x = d3.scaleBand()
      .domain(object.name)
      .range([0, width])
      .padding([0.2])
    svg.append("g")
      .attr("transform", `translate(0, ${height})`)
      .call(d3.axisBottom(x).tickSizeOuter(0));
    const y = d3.scaleLinear()
      .domain([0, 100])
      .range([ height, 0 ]);
    svg.append("g")
      .call(d3.axisLeft(y));
  });
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
  <div id="my_dataviz">
    <h3>Visualization</h3>
  </div>
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
