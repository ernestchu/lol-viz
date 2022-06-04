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
    const exp = 8
    const entriesHierarchy = d3.hierarchy({ children: challengerLeague.entries })
                          .sum(d => Math.pow(d.leaguePoints, exp))

    d3.treemap()
      .size([1500, 750])
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
  })
  .catch((e) => {
    console.log(e)
    showErrorMsg.value = true
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
    <div class="summoner-block" v-for="entry in challengerLeague.entries" :style="entry.style">
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
    </div>
  </div>
</template>

<style scoped>
.treemap {
  position: relative;
  margin-bottom: 1000px;
}
.summoner-block {
  position: absolute;
}
.summoner-detail {
  font-size: 10px;
  position: absolute;
  margin: 10px;
}
.summoner-name {
  font-size: 16px;
  color: white;
  cursor: pointer;
}
</style>
