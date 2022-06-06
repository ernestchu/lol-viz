<script setup>
import { useRoute } from 'vue-router'
import { ref, reactive } from 'vue'
const { summonerId } = useRoute().params
import * as d3 from 'd3'

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
  var champList = {};
  var updatedChampList = [];
  var gameList = [];
  const matchPromises = []
  matches.value.slice(0, nMatches.value).forEach(match => {
    // use a subarray to develop to prevent reaching the limit.
    const matchPromise = new Promise((resolve, reject) => {
      fetch(`${proxyHost}${matchStatsURI}/${match}`)
        .then(res => res.json())
        .then(data => {
          //console.log(data) // this is very rich
          const index = data.metadata.participants.indexOf(puuid)
          const stats = data.info.participants[index]
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
          gameList.push({"name":stats.championName,"Kills":stats.kills, "Deaths":stats.deaths, "Assists":stats.assists, "Damage":stats.totalDamageDealtToChampions, "Games":1, "Wins":stats.win, "ID":match });
          delete stats.challenges
          delete stats.perks
          stats.show = false
          matchStats[match] = stats
          // process complete! fulfill the promise
          resolve()
        })
        .catch((e) => {
          reject(e)
          console.error('May have reached the rate limit.')
        })
    })
    matchPromises.push(matchPromise)
  })
  // call visualization when all the matches have processed
  Promise.all(matchPromises)
    .then(() => visualization(gameList, updatedChampList))
}
function visualization(gameList, champList){
  d3.selectAll('svg').remove();
  var updatedGameList = [];
  var group = champList.map(d => (d.name));
  var max = d3.max(champList.map(d => (d.Games)))
  const subgroups = ['wins', 'loses']
  const margin = {top: 10, right: 230, bottom: 120, left: 100},
    width = 900 - margin.left - margin.right,
    height = 700 - margin.top - margin.bottom;
  const svg = d3.select("#my_dataviz")
    .append("svg")
      .attr("width", width + margin.left + margin.right)
      .attr("height", height + margin.top + margin.bottom)
    .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);
  var tooltip = d3.select("#my_dataviz")
    .append("div")
    .style("opacity", 0)
    .attr("class", "tooltip")
    .style("background-color", "white")
    .style("border", "solid")
    .style("border-width", "1px")
    .style("border-radius", "5px")
    .style("padding", "10px")

  gameList.sort(function(a, b) {          
      if (a.name === b.name) {
         // Price is only important when cities are the same
         return b.Wins - a.Wins;
      }
      return a.name > b.name ? 1 : -1;
   });
  for(var i=1; i<gameList.length; i++){
    if(gameList[i].name===gameList[i-1].name){
      gameList[i].Games=gameList[i-1].Games+1;
    }
  }
  console.log(gameList);

  var mouseover = function() {
    tooltip
      .style("opacity", 1)
    d3.select(this)
      .style("r", 20)
      .style("stroke", "black")
  }
  var mousemove = function(d) {
    let pt = d3.pointer(event, this)
    console.log(pt)
    tooltip
      .html("Champion: " + d.target.__data__.name +"\nKills: " + d.target.__data__.Kills + "\nDeaths: " + d.target.__data__.Deaths + "\nAssists: " + d.target.__data__.Assists + "\nDamage: " + d.target.__data__.Damage)
      .attr('left', pt[0]+'px') //取[x]
      .attr('top', pt[1]+'px') //取[Y]
  }
  var mouseleave = function() {
    tooltip
      .style("opacity", 0)
    d3.select(this)
      .style("r", 10)
      .style("stroke", "none")
  }
  gameList.forEach(object =>{
    var y = d3.scaleBand()
      .range([0, height])
      .domain(group)
      .padding([0.2])
    svg.append("g")
      .style("font", "13px sans-serif")
      .call(d3.axisLeft(y));
    var x = d3.scaleLinear()
      .domain([0, max])
      .range([0, width]);
    svg.append("g")
      .style("font", "13px sans-serif")
      .attr("transform", `translate(0, ${height})`)
      .call(d3.axisBottom(x));
    var dots = svg.selectAll("circle")
      .data(gameList)
      .enter().append("circle")
      .attr("class", "dot")
      .attr("r", 10)
      .attr("cx", function(d) {
        return x(d.Games)
      })
      .attr("cy", function(d) {
        return y(d.name) + y.bandwidth()/2
      })
      .style("fill", function(d) {
        if(d.Wins == true){
          return "skyblue"
        }
        else{
          return "red"
        };
      })
      .on("mouseover", mouseover)
      .on("mousemove", mousemove)
      .on("mouseleave", mouseleave)
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
