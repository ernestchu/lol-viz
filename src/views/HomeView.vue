<script setup>
import { ref, reactive, onMounted } from 'vue'
import { RouterLink } from 'vue-router'
import * as d3 from 'd3'

const challengerLeague = reactive({})
const showErrorMsg = ref(false)

const proxyHost = import.meta.env.VITE_PROXY_HOST
const targetURI = '/lol/league/v4/challengerleagues/by-queue/RANKED_SOLO_5x5'

fetch(proxyHost + targetURI)
  .then(res => res.json())
  .then(data => {
    Object.assign(challengerLeague, data)
    challengerLeague.entries.forEach(entry => {
      entry.show = false
    })
    challengerLeague.entries.sort((a, b) => b.leaguePoints - a.leaguePoints)
    computeTreemap()
  })
  .catch((e) => {
    console.log(e)
    showErrorMsg.value = true
  })

function computeTreemap () {
  const exp = 8
  const entriesHierarchy = d3.hierarchy({ children: challengerLeague.entries })
                        .sum(d => Math.pow(d.leaguePoints, exp))
  d3.treemap()
    .size([window.innerWidth * 0.98, 750])
    .padding(10)
    (entriesHierarchy)

  const colorScale = d3.scaleLinear()
    .range(['#B3944C88', 'lightblue'])
    .domain([
      d3.max(challengerLeague.entries.map(d => d.leaguePoints)),
      d3.min(challengerLeague.entries.map(d => d.leaguePoints))
    ])
  const coordinates = entriesHierarchy.leaves()
  challengerLeague.entries.forEach((entry, index) => {
    const { x0, x1, y0, y1 } = coordinates[index]
    const style = {
      left  : x0 + 'px',
      top   : y0 + 'px',
      width : (x1 - x0) + 'px',
      height: (y1 - y0) + 'px',
      'background-color': colorScale(entry.leaguePoints)
    }

    entry.style = style
  })
}

  /* use timeout to throttle the window resizing update */
let resizeTimeout
window.addEventListener('resize', () => {
  if(!!resizeTimeout) {
    clearTimeout(resizeTimeout)
  }
  resizeTimeout = setTimeout(() => {
    // update window.innerWidth to the treemap
    computeTreemap()
  }, 100) // minimal update interval: 0.1 sec
})

function zoomIn (entry) {
  entry.isChosen = true
  // make a hard copy of the original style
  entry.styleBackup = JSON.parse(JSON.stringify(entry.style))
  entry.style.top = 0
  entry.style.left = 0
  entry.style.width = window.innerWidth * 0.98 + 'px'
  entry.style.height = 750 + 'px'
  entry.style['z-index'] = 999
  entry.style['background-color'] = RGBAToHexA(entry.style['background-color']).slice(0, -2)


  // get champion mastry info
  fetch(`https://na1.api.riotgames.com/lol/champion-mastery/v4/champion-masteries/by-summoner/${entry.summonerId}?api_key=RGAPI-0a372b87-e4a8-4995-90bf-5fa18b78ef09`)
    .then(res => res.json())
    .then(data => {
      data = data.slice(0, 15)

      // uncomment to examine the data
      /* data.forEach(champ => { */
      /*   console.log(champ.championPoints) */
      /*   console.log(champions[champ.championId]) */
      /* }) */

      const x = d3.scaleLinear()
        .domain([
          d3.min(data.map(d => d.championPoints)),
          d3.max(data.map(d => d.championPoints))
        ])
        .range([0, 1000])
      const y = d3.scaleBand()
        .domain(data.map(d => d.championId))
        .range([0, 600])
        .padding(.1)

      const masteryBars = []
      data.forEach(d => {
        masteryBars.push({
          championId: d.championId,
          style: {
            left:  y.bandwidth() * 1.3 + '0px', // save space for the icons
            top: y(d.championId) + 'px',
            width: x(d.championPoints) + 'px',
            height: y.bandwidth() + 'px',
            'background-color': 'red'
          },
          imgStyle: {
            left: -y.bandwidth() * 1.3 + 'px',
            width: y.bandwidth() + 'px'
          }
        })
      })
      entry.masteryBars = masteryBars
    })
}

function zoomOut ($event, entry) {
  $event.stopPropagation()
  entry.isChosen = false
  entry.style = JSON.parse(JSON.stringify(entry.styleBackup))
  delete entry.styleBackup
  delete entry.masteryBars
}

function RGBAToHexA(rgba) {
  let sep = rgba.indexOf(",") > -1 ? "," : " "; 
  rgba = rgba.substr(5).split(")")[0].split(sep);
                
  // Strip the slash if using space-separated syntax
  if (rgba.indexOf("/") > -1)
    rgba.splice(3,1);

  for (let R in rgba) {
    let r = rgba[R];
    if (r.indexOf("%") > -1) {
      let p = r.substr(0,r.length - 1) / 100;

      if (R < 3) {
        rgba[R] = Math.round(p * 255);
      } else {
        rgba[R] = p;
      }
    }
  }
  let r = (+rgba[0]).toString(16),
      g = (+rgba[1]).toString(16),
      b = (+rgba[2]).toString(16),
      a = Math.round(+rgba[3] * 255).toString(16);

  if (r.length == 1)
    r = "0" + r;
  if (g.length == 1)
    g = "0" + g;
  if (b.length == 1)
    b = "0" + b;
  if (a.length == 1)
    a = "0" + a;

  return "#" + r + g + b + a;
}

const champions = reactive({})
let ddragonChampSquarePrefix = ''
function ddragonChampSquare (id) {
  return ddragonChampSquarePrefix + `/${id}.png`
}
fetch('https://ddragon.leagueoflegends.com/api/versions.json')
  .then(res => res.json())
  .then(data => {
    ddragonChampSquarePrefix = `https://ddragon.leagueoflegends.com/cdn/${data[0]}/img/champion`
    fetch(`https://ddragon.leagueoflegends.com/cdn/${data[0]}/data/en_US/champion.json`)
      .then(res => res.json())
      .then(({ data }) => {
        /* ########### champion info ########## */
        /* keep keys/values we need */
        for (const [key, value] of Object.entries(data)) {
          /* dont use the championsId as key, fuck Fiddlesticks */
          champions[data[key]['key']] = [ 'id', 'key', 'name' ].reduce((prev, curr) => {
            prev[curr] = data[key][curr]
            return prev
          }, {})
        }
        /* #################################### */
      })
  })

</script>

<template>
  <template v-if="showErrorMsg">
    <div>
      <h2>The page has encountered an error!</h2>
      See the console for further informations.
    </div>
  </template>

  <div id="d3"></div>

  <div class="treemap" v-if="challengerLeague">
    <div class="summoner-block" v-for="entry in challengerLeague.entries" :style="entry.style" @click="entry.isChosen ? null : zoomIn(entry)">
      <div v-if="entry.isChosen" class="close-button" @click="zoomOut($event, entry)">
        <div class="line"></div>
        <div class="line"></div>
      </div>
      <!-- <button v-if="entry.isChosen" @click="zoomOut(entry)"></button> -->
      <div class="summoner-detail" v-if="+entry.style.width.slice(0, -2) > 100">
        <span class="summoner-name" @click="entry.show = !entry.show">
          {{ entry.summonerName }}
        </span>
        <ul v-if="entry.show">
          <RouterLink
            :to="{
              name: 'matches',
              params: { summonerId: entry.summonerId }
            }"
          >
            View matches
          </RouterLink>
          <template v-for="(entryValue, entryKey) in entry">
            <li v-if="entryKey != 'show'">
              {{ entryKey }}: {{ entryValue }}
            </li>
          </template>
        </ul>
      </div>
      <div class="mastery-bar-chart" v-if="entry.isChosen && entry.masteryBars">
        <div v-for="bar in entry.masteryBars" :style="bar.style">
          <img :src="ddragonChampSquare(champions[bar.championId].id)" :style="bar.imgStyle">
        </div>
      </div>
    </div>
  </div>

</template>

<style scoped>
.mastery-bar-chart {
  position: relative;
  margin: 30px;
}
.mastery-bar-chart > div {
  position: absolute;
}
.mastery-bar-chart > div > img {
  position: absolute;
  top: 0px;
}

.treemap {
  position: relative;
  margin-bottom: 1000px;
}
.summoner-block {
  position: absolute;
}
.summoner-detail {
  font-size: 10px;
  margin: 10px;
}
.summoner-name {
  font-size: 16px;
  font-family: 'Share Tech Mono', monospace;
  color: white;
  cursor: pointer;
}

button {
  background-color: transparent;
  background-repeat: no-repeat;
  border: none;
  cursor: pointer;
  overflow: hidden;
  outline: none;
}


.close-button {
  width: 30px;
  height: 30px;
  position: absolute;
  right: 0px;
  border: 2px solid #eee;
  background-color: #ddd4;
  border-radius: 50%;
  transition: all .3s;
}
.close-button:hover {
  transform: scale(1.2);
  cursor: pointer;
  background-color: #ddd0;
  border-color: #eee0;
}
.close-button .line {
  position: absolute;
  width: 70%;
  left: 10%;
  height: 10%;
  top: 45%;
  background-color: white;
}
.close-button .line:nth-child(1) {
  transform: rotate(45deg);
}
.close-button .line:nth-child(2) {
  transform: rotate(-45deg);
}

</style>
