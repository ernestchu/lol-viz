<script setup>
import { useRoute } from 'vue-router'
import { ref, reactive } from 'vue'

/* ddragon */
const ddragonSRMAP = ref('')
const ddragonChampSquarePrefix = ref('')
const ddragonChampSquare = id => ddragonChampSquarePrefix.value + `/${id}.png`
const champions = {}
const ddragonVersions = 'https://ddragon.leagueoflegends.com/api/versions.json'
const positionNormFactor = 1 / 17500
const match = reactive({})
const levels = reactive([
  { data: [] },
  { data: [] },
  { data: [] }
])
const selectedIndicators = reactive({
  show: [ false, false ],
  styles: []
})
const authFailed = ref(false)
const matchId = useRoute().params.matchId
const windowInnerWidth = ref(window.innerWidth)
window.addEventListener('resize', () => {
  windowInnerWidth.value = window.innerWidth
})

fetch(ddragonVersions)
  .then(res => res.json())
  .then(data => {
    ddragonSRMAP.value = `https://ddragon.leagueoflegends.com/cdn/${data[0]}/img/map/map11.png`
    ddragonChampSquarePrefix.value = `https://ddragon.leagueoflegends.com/cdn/${data[0]}/img/champion`
    const ddragonChampions = `https://ddragon.leagueoflegends.com/cdn/${data[0]}/data/en_US/champion.json`

    const proxyHost = import.meta.env.VITE_PROXY_HOST
    const matchStatsURI = `/lol/match/v5/matches/${matchId}`
    const timelineURI = `/lol/match/v5/matches/${matchId}/timeline`

    Promise.all([
      fetch(ddragonChampions),
      /* fetch(`${proxyHost}${matchStatsURI}`), */
      fetch('/sample-data/NA1_4309929537.json'),
      /* fetch(`${proxyHost}${timelineURI}`) */
      fetch(`/sample-data/timeline.json`)
    ]).then(res => Promise.all([ res[0].json(), res[1].json(), res[2].json() ]))
      .then(dataArray => {
        /* ########### champion info ########## */
        let { data } = dataArray[0]
        /* keep keys/values we need */
        for (const [key, value] of Object.entries(data)) {
          /* dont use the championsId as key, fuck Fiddlesticks */
          champions[data[key]['key']] = [ 'id', 'key', 'name' ].reduce((prev, curr) => {
            prev[curr] = data[key][curr]
            return prev
          }, {})
        }
        /* #################################### */


        /* ########### match stats ########### */
        data = dataArray[1]
        const participantKeys = [
          'kills',
          'deaths',
          'assists',
          'championId',
          'teamId',
          'summonerName',
          'puuid'
        ]
        /* keep keys/values we need */
        match.participants = data.info.participants.map(
          participant => participantKeys.reduce((prev, curr) => {
            prev[curr] = participant[curr]
            return prev
          }, {})
        )
        match.participants.forEach(participant => {
          participant.champion = champions[participant.championId]
          delete participant.championId
        })
        /* ################################### */


        /* ############## timeline  ##############*/
        data = dataArray[2]
        const discardEventTypes = [
          'PAUSE_END',
          'ITEM_PURCHASED',
          'ITEM_UNDO',
          'ITEM_SOLD',
          'ITEM_DESTROYED',
          'SKILL_LEVEL_UP',
          'LEVEL_UP',
          'WARD_PLACED',
          'WARD_KILL',
          'TURRET_PLATE_DESTROYED'
        ]
        // extract timeline
        data.info.frames.forEach(frame => {
          frame.events = frame.events.filter(evt => !discardEventTypes.includes(evt.type))
        })
        match.timeline = data

        // extract participant
        const participantIds = {}
        data.info.participants.forEach(participant => {
          participantIds[participant.puuid] = participant.participantId
        })
        const temp = {}
        match.participants.forEach(participant => {
          temp[participantIds[participant.puuid]] = participant
        })
        match.participants = temp
        
        levels[0].style = getLevelStyle(0)
        for (let i = 0; i < match.timeline.info.frames.length; i+=5) {
          levels[0].data.push(match.timeline.info.frames.slice(i, i+5))
        }
        /* #######################################*/
      })
      .catch((e) => {
        console.err(e)
        authFailed.value = true
      })
  })

function selectMap (event, index, level, data) {
  // remove sub levels
  ((startLevel, endLevel) => {
    while(startLevel <= endLevel) {
      levels[startLevel].data = []
      delete levels[startLevel].style
      delete levels[startLevel].parentIndex
      startLevel += 1
    }
  })(level, 2)

  if (event.target.parentNode.classList.contains('selected')) {
    event.target.parentNode.classList.replace('selected', 'map-hover')
    return
  }
  event.target.parentNode.classList.replace('map-hover', 'selected')
  document.querySelectorAll(`.level${level - 1}`).forEach(node => {
    if (node !== event.target.parentNode) {
      node.classList.replace('selected', 'map-hover')
    }
  })

  levels[level].parentIndex = index
  levels[level].style = getLevelStyle(level)
  levels[level].data = data

  let translateZ = 0;
  for (let i = level; i >= 1; i--) {
    translateZ += levels[i - 1].style['--sepZ'].slice(0, -2) *
                  levels[i].parentIndex
  }
  selectedIndicators.styles[level - 1] = {
    left: levels[level - 1].style.left,
    width: parseFloat(levels[level].style.left.slice(0, -1)) -
           parseFloat(levels[level - 1].style.left.slice(0, -1)) + 'px',
    height: levels[level - 1].style.height,
    'padding-top': levels[level - 1].style['--top-padding'],
    'padding-bottom': levels[level - 1].style['--bottom-padding'],
    transform: `translateZ(${translateZ}px)`
  }
  selectedIndicators.show[level - 1] = true
}

function getLevelStyle (level, parentIndex) {
  const nTopLevel = Math.ceil(match.timeline.info.frames.length / 5)
  // the Coefficient need to be fine-tuned
  const width = window.innerWidth * 0.12 / (1 + nTopLevel * 0.032)
  const sepZ = 0.8 * width

  const style = {
    '--sepZ': `${sepZ}px`,
    width:  `${width}px`,
    height:  `${width}px`,
    position: 'absolute',
    'transform-style': 'preserve-3d',
    '--selected-translateX': '240%',
    '--horizontal-padding': `${0.08 * width}px`,
    '--bottom-padding': `${0.04 * width}px`,
    '--top-padding': `${0.22 * width}px`,
    '--border-width': `${0.016 * width}px`,
    '--title-font-size': `${0.088 * width}px`
  }

  if (level == 0) {
    style.left = `${nTopLevel * sepZ * 0.4}px`
  }
  else {
    style.left = `${
      parseFloat(levels[level - 1].style.left.slice(0, -2)) +
      (
      parseFloat(levels[level - 1].style.width.slice(0, -2)) +
      parseFloat(levels[level - 1].style['--horizontal-padding'].slice(0, -2)) * 2 +
      parseFloat(levels[level - 1].style['--border-width'].slice(0, -2)) * 2) *
      parseFloat(levels[level - 1].style['--selected-translateX'].slice(0, -1)
      ) * 0.01
    }px`,
    style.transform = `translateZ(${levels[level - 1].style['--sepZ'].slice(0, -2) * levels[level].parentIndex}px)`

    if (level === 1) {
      style.transform = `translateZ(${levels[level - 1].style['--sepZ'].slice(0, -2) * levels[level].parentIndex}px)`
    } else if (level === 2) {
      style.transform = `translateZ(${
        levels[level - 2].style['--sepZ'].slice(0, -2) * levels[level - 1].parentIndex + 
        levels[level - 1].style['--sepZ'].slice(0, -2) * levels[level].parentIndex
      }px)`
    }
  }

  return style
}
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
  {{ windowInnerWidth }}
  <h3>Timeline</h3>
  
  <div style="margin-bottom: 100px">
    <img
      class="champion-square"
      v-for="participant in match.participants"
      :src="ddragonChampSquare(participant.champion.id)"
    >
  </div>
  

  <div class="timeline">
    <div :style="levels[0].style">
      <TransitionGroup>
        <div
          v-if="!!levels[0].data.length"
          v-for="(level1, index) in levels[0].data"
          :key="index"
          class="map map-hover level0"
          :style="{ '--index': index  }"
          @click="selectMap($event, index, 1, level1)"
        >
          <div class="map-title">
            TIME FRAME ID: {{ level1[0].timestamp }}
          </div>
          <div
            class="ring"
            v-for="(participantFrame, participantId) in level1[0].participantFrames"
            :style="{
              '--ring-hue': match.participants[participantId].teamId === 100 ? 240 : 1,
              left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
                participantFrame.position.x * positionNormFactor * levels[0].style.width.slice(0, -2) + 'px)',
              bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
                participantFrame.position.y * positionNormFactor * levels[0].style.height.slice(0, -2) + 'px)'
            }"
          >
            <img
              class="champion-icon"
              :src="ddragonChampSquare(match.participants[participantId].champion.id)"
            >
          </div>
          <img :src="ddragonSRMAP">
        </div>
      </TransitionGroup>
    </div>

    <div :style="levels[1].style">
      <TransitionGroup name="level-1">
        <div
          v-if="!!levels[1].data.length"
          v-for="(level2, index) in levels[1].data"
          :key="levels[1].parentIndex + '-' + index"
          class="map map-hover level1"
          :style="{ '--index': index }"
          @click="selectMap($event, index, 2, level2.events)"
        >
          <div class="map-title">
            TIME FRAME ID: {{ level2.timestamp }}
          </div>
          <div
            class="ring"
            v-for="(participantFrame, participantId) in level2.participantFrames"
            :style="{
              '--ring-hue': match.participants[participantId].teamId === 100 ? 240 : 1,
              left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
                participantFrame.position.x * positionNormFactor * levels[1].style.width.slice(0, -2) + 'px)',
              bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
                participantFrame.position.y * positionNormFactor * levels[1].style.height.slice(0, -2) + 'px)'
            }"
          >
            <img
              class="champion-icon"
              :src="ddragonChampSquare(match.participants[participantId].champion.id)"
            >
          </div>
          <img :src="ddragonSRMAP">
        </div>
      </TransitionGroup>
    </div>

    <div :style="levels[2].style">
      <TransitionGroup name="level-1">
        <div
          v-if="!!levels[2].data.length"
          v-for="(level3, index) in levels[2].data"
          :key="levels[2].parentIndex + '-' + index"
          class="map map-hover level2"
          :style="{ '--index': index }"
          @click="selectMap($event, index, 3, level3.events)"
        >
          <div class="map-title">
            TIME FRAME ID: {{ level3.timestamp }}
          </div>
          <img :src="ddragonSRMAP">
        </div>
      </TransitionGroup>
    </div>

    <div class="selected-indicator" v-if="selectedIndicators.show[0]" :style="selectedIndicators.styles[0]"></div>
    <div class="selected-indicator" v-if="selectedIndicators.show[1]" :style="selectedIndicators.styles[1]"></div>

  </div>
  <!--
  <div id="cont">
    <div id="box"></div>
  </div>
  -->
</template>

<style scoped>
img {
  width : 100%;
  height: 100%;
}
img.champion-square {
  width: 100px;
  border-radius:50%;
}
.timeline {
  position: relative;
  transform: rotateX(340deg) rotateY(335deg);
  height: 1500px;
  transform-style: preserve-3d;
}
.map {
  position: absolute;
  cursor: pointer;
  width:  100%;
  height: 100%;
  transition: transform .5s, opacity 0s ease .5s;
  transform: translateZ(calc(var(--index) * var(--sepZ)));
  background-color: #ddd4;
  padding: var(--top-padding) var(--horizontal-padding) var(--bottom-padding) var(--horizontal-padding);
  border: var(--border-width) solid #eee;
  backdrop-filter: blur(1.5px);
  border-radius: 5px;
}
.map-hover:hover { /* use this to prevent still hovering when selected */
  transform: translateZ(calc(var(--index) * var(--sepZ)))
             translateX(25%);
}
.selected {
  opacity: 0;
  transform: translateZ(calc(var(--index) * var(--sepZ)))
             translateX(var(--selected-translateX));
}
.map-title {
  position: absolute;
  font-size: var(--title-font-size);
  font-weight: bold;
  top: 5px;
  font-family: 'Share Tech Mono', monospace;
}
.ring {
  position: absolute;
  background: linear-gradient(
    hsl(var(--ring-hue), 60%, 60%),
    hsl(var(--ring-hue), 60%, 15%),
    hsl(var(--ring-hue), 60%, 25%));
  width:  10%;
  height: 10%;
  padding: 1%;
  border-radius: 50%;
}
img.champion-icon {
  border-radius: 50%;
}

.selected-indicator {
  position: absolute;
  border: dotted #eee;
  border-right: none;
  border-radius: 5px;
}


#cont{
  /* background: linear-gradient(to bottom, red 0 50%, blue 50% 100%); */
  background: linear-gradient(hsl(43, 40%, 60%),  hsl(43, 40%, 15%), hsl(43, 40%, 25%));
  width: 300px;
  height: 300px;
  border-radius: 1000px;
  padding: 10px;
}

#box{
  background: black;
  width: 300px;
  height: 300px;
  border-radius: 1000px;
}

/* Vue Transition */
.level-1-enter-active {
  transition: transform .8s, opacity .01s;
  transition-delay: .5s;
}
.level-1-leave-active {
  transition: opacity .5s;
}
.level-1-enter-from {
  opacity: 0;
  transform: translateZ(0);
}
.level-1-leave-to {
  opacity: 0;
}
</style>
