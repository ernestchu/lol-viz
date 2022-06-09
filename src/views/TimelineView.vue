<script setup>
import { useRoute } from 'vue-router'
import { ref, reactive, nextTick } from 'vue'
import Spinner from '@/components/Spinner.vue'

/* ddragon */
const ddragonSRMAP = ref('')
const ddragonChampSquarePrefix = ref('')
const ddragonChampSquare = id => ddragonChampSquarePrefix.value + `/${id}.png`
const champions = {}
const ddragonVersions = 'https://ddragon.leagueoflegends.com/api/versions.json'
const positionNormFactor = 1 / 17500
const match = reactive({})
const levels = reactive([
  { show: false },
  { show: false },
  { show: false }
])
const selectedIndicators = reactive([
  { show: false },
  { show: false }
])
const closeButtons = reactive([
  { show: false },
  { show: false }
])
const showErrorMsg = ref(false)
const matchId = useRoute().params.matchId
const pins = reactive({
  championLocations:  { show: true, displayMessage: 'Champion locations' },
  championKills:      { show: true,  displayMessage: 'Champion kills' },
  championMultiKills: { show: true,  displayMessage: 'Champion multiple kills'},
  monstersKills:      { show: true,  displayMessage: 'Monster kills' },
  buildingsKills:     { show: true,  displayMessage: 'Building kills' }
})

fetch(ddragonVersions)
  .then(res => res.json())
  .then(data => {
    ddragonSRMAP.value = `https://ddragon.leagueoflegends.com/cdn/${data[0]}/img/map/map11.png`
    ddragonSRMAP.value = `https://raw.communitydragon.org/latest/plugins/rcp-fe-lol-match-history/global/default/map11.png`
    ddragonChampSquarePrefix.value = `https://ddragon.leagueoflegends.com/cdn/${data[0]}/img/champion`
    const ddragonChampions = `https://ddragon.leagueoflegends.com/cdn/${data[0]}/data/en_US/champion.json`

    const proxyHost = import.meta.env.VITE_PROXY_HOST
    const matchStatsURI = `/lol/match/v5/matches/${matchId}`
    const timelineURI = `/lol/match/v5/matches/${matchId}/timeline`

    Promise.all([
      fetch(ddragonChampions),
      fetch(`${proxyHost}${matchStatsURI}`),
      /* fetch('/sample-data/NA1_4309929537.json'), */
      fetch(`${proxyHost}${timelineURI}`)
      /* fetch(`/sample-data/timeline.json`) */
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
          'TURRET_PLATE_DESTROYED',
          'OBJECTIVE_BOUNTY_PRESTART',
          'OBJECTIVE_BOUNTY_FINISH',
          'GAME_END'
        ]
        // extract timeline
        data.info.frames.forEach(frame => {
          frame.events = frame.events.filter(evt => !discardEventTypes.includes(evt.type))
        })
        match.timeline = data

        /* const t = {} */
        /* data.info.frames.forEach(f => { */
        /*   f.events.forEach(evt => { */
        /*     if (evt.type in t) { */
        /*       t[evt.type].push(evt) */
        /*     } else { */
        /*       t[evt.type] = [ evt ] */
        /*     } */
        /*   }) */
        /* }) */
        /* console.log(t['ELITE_MONSTER_KILL'].map(d => d.monsterType).reduce((a, b) => `${a}\n${b}`), '') */
        /* console.log(t['ELITE_MONSTER_KILL'].filter(d => d.monsterSubType).map(d => d.monsterSubType).reduce((a, b) => `${a}\n${b}`), '') */

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
        levels[0].data = []
        for (let i = 0; i < match.timeline.info.frames.length; i+=5) {
          levels[0].data.push(match.timeline.info.frames.slice(i, i+5))
        }
        levels[0].show = true
        /* #######################################*/
      })
      .catch((e) => {
        console.error(e)
        showErrorMsg.value = true
      })
  })

function selectMap (index, level, data) {
  closeMap(level - 1)
  nextTick(() => {
    const selectedNode = document.querySelectorAll(`.level${level - 1}`)[index]
    selectedNode.classList.replace('map-hover', 'selected')

    levels[level].parentIndex = index
    levels[level].style = getLevelStyle(level)
    levels[level].data = data
    levels[level].show = true

    updateSeletedInidicator(level - 1)
    updateCloseButton(level - 1)
  })
  /* TODO: prevent hovering when transitioning */
}

function closeMap (level) {
  for (let i = level + 1; i < levels.length; i++) {
    levels[i].show = false
    selectedIndicators[i - 1].show = false
    closeButtons[i - 1].show = false
  }

  document.querySelectorAll(`.level${level}`).forEach(node => {
    node.classList.add('deselected')
    node.classList.replace('selected', 'map-hover')
    nextTick(() => node.classList.remove('deselected'))
  })
}

function updateSeletedInidicator (index) {
  let translateZ = 0;
  for (let i = index + 1; i >= 1; i--) {
    translateZ += levels[i - 1].style['--sepZ'].slice(0, -2) *
                  levels[i].parentIndex
  }
  selectedIndicators[index].style = {
    left: levels[index].style.left,
    '--width': parseFloat(levels[index + 1].style.left.slice(0, -2)) -
               parseFloat(levels[index].style.left.slice(0, -2)) + 'px',
    height: levels[index].style.height,
    'padding-top': levels[index].style['--top-padding'],
    'padding-bottom': levels[index].style['--bottom-padding'],
    transform: `translateZ(${translateZ}px)`
  }
  selectedIndicators[index].show = true
}

function updateCloseButton (index) {
  let translateZ = 0;
  for (let i = index + 1; i >= 1; i--) {
    translateZ += levels[i - 1].style['--sepZ'].slice(0, -2) *
                  levels[i].parentIndex
  }
  const width = parseFloat(levels[index + 1].style.width.slice(0, -2)) * 0.2
  closeButtons[index].style = {
    width:  `${width}px`,
    height: `${width}px`,
    left: parseFloat(levels[index + 1].style.left.slice(0, -2)) +
          parseFloat(levels[index + 1].style.width.slice(0, -2)) + 'px',
    '--translateX': `-${(parseFloat(levels[index + 1].style.left.slice(0, -2)) -
                        parseFloat(levels[index].style.left.slice(0, -2))) * 0.5}px`,
    '--translateY': `-${width * 2}px`,
    '--translateZ': `${translateZ}px`
  }
  closeButtons[index].show = true
}

function getLevelStyle (level) {
  const nTopLevel = Math.ceil(match.timeline.info.frames.length / 5)
  // the Coefficient need to be fine-tuned
  const width = window.innerWidth / (6 + nTopLevel * 0.32)
  const sepZ = 0.8 * width

  const style = {
    '--window-innerWidth': window.innerWidth,
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
    '--title-font-size': `${0.1 * width}px`
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

/* use timeout to throttle the window resizing update */
let resizeTimeout
window.addEventListener('resize', () => {
  if(!!resizeTimeout) {
    clearTimeout(resizeTimeout)
  }
  resizeTimeout = setTimeout(() => {
    window.innerWidth
    levels.forEach((level, index) => {
      if (level.style && level.style['--window-innerWidth'] !== window.innerWidth) {
        level.style = getLevelStyle(index)
      }
    })
    selectedIndicators.forEach((selectedIndicator, index) => {
      if (selectedIndicator.show) {
        updateSeletedInidicator(index)
      }
    })
  }, 100) // minimal update interval: 0.1 sec
})

function minisecToMinute (minisec) {
  let sec = Math.floor(minisec / 1000)
  const min = Math.floor(sec / 60)
  sec -= min * 60
  return `${min} min ${sec ? + sec + ' sec' : ''}`
}

function eliteMonsterBgColor (event) {
  if (!event) {
    return 'revert-layer'
  }
  const colors = {
    BARON_NASHOR:   '#C35ED744',
    RIFTHERALD:     '#F624F644',
    ELDER_DRAGON:   '#3C364E44',
    AIR_DRAGON:     '#29546844',
    CHEMTECH_DRAGON:'#DFE53544',
    EARTH_DRAGON:   '#AC8F2B44',
    FIRE_DRAGON:    '#E06F1444',
    HEXTECH_DRAGON: '#0A2EA844',
    WATER_DRAGON:   '#60DAEB44'
  }
  const monsterType = event.monsterType === 'DRAGON' ? event.monsterSubType : event.monsterType
  return colors[monsterType] ? colors[monsterType] : 'revert-layer'
}

function eliteMonsterIcon (event) {
  const monsterType = event.monsterType === 'DRAGON' ? event.monsterSubType : event.monsterType
  return `${import.meta.env.BASE_URL}images/elite-monsters/${monsterType}.png`
}

const previewWindow = reactive({ show: false })
function showPreview ($event, event) {
  const { clientX, clientY } = $event
  const mapWidth = parseFloat(levels[0].style.width.slice(0, -2))
  previewWindow.style = {
    width: mapWidth * 1.5 + 'px',
    height: mapWidth * 0.375 + 'px',
    top: clientY - mapWidth * 0.4 + 'px',
    left: clientX + 'px',
  }
  const attrs = [ 'killerId', 'victimId', 'timestamp' ]
  attrs.forEach(attr => {
    previewWindow[attr] = event[attr]
  })
  previewWindow.timestamp = Math.floor(previewWindow.timestamp / 1000)
  previewWindow.timestamp = `
    ${Math.round(previewWindow.timestamp / 60)}:${(previewWindow.timestamp % 60).toString().padStart(2, '0')}
  `
  previewWindow.show = true
}
function hidePreview () {
  previewWindow.show = false
}
</script>

<template>

  <template v-if="showErrorMsg">
    <div>
      <h2>The page has encountered an error!</h2>
      See the console for further informations.
    </div>
  </template>

  <Transition>
  <div v-if="levels[0].show">
  <div class="overview">
    <div class="blue">
      <div v-for="participant in Object.values(match.participants).slice(0, 5)">
        <img class="champion-square" :src="ddragonChampSquare(participant.champion.id)">
        <div class="text">
          {{ participant.summonerName }}<br>
          <span>{{ participant.champion.name }}</span>
        </div>
        <div>{{ `${participant.kills}/${participant.deaths}/${participant.assists}` }}</div>
      </div>
    </div>
    <div class="red">
      <div v-for="participant in Object.values(match.participants).slice(5)">
        <img class="champion-square" :src="ddragonChampSquare(participant.champion.id)">
        <div class="text">
          {{ participant.summonerName }}<br>
          <span>{{ participant.champion.name }}</span>
        </div>
        <div>{{ `${participant.kills}/${participant.deaths}/${participant.assists}` }}</div>
      </div>
    </div>
  </div>

  <div class="pin-control">
    <div class="switch" v-for="(pin, index) in pins">
      <label class="form-switch">
        <input type="checkbox" v-model="pin.show">
        <i></i>
        {{ pin.displayMessage }}
      </label>
    </div>
  </div>
  

  <div class="timeline">
    <div :style="levels[0].style">
      <div
        v-if="levels[0].show"
        v-for="(level1, index) in levels[0].data"
        :key="index"
        class="map map-hover level0"
        :style="{
          '--index': index,
          'background-color': eliteMonsterBgColor(
            level1.map(level2 => level2.events)
                  .reduce((allEvts, evts) => allEvts.concat(evts), [])
                  .filter(evt => evt.type === 'ELITE_MONSTER_KILL')
                  [0]
          )
        }"
        @click="selectMap(index, 1, level1)"
      >
        <div class="map-title">
          TIME: {{ minisecToMinute(level1[0].timestamp) }}
        </div>
        <div
          class="map-num-child"
          :style="{ opacity: levels[1].show ? 0.3 : 1 }"
        >
          {{ level1.length }} frames <br>
          {{ level1.map(level2 => level2.events.length).reduce((sum, len) => sum + len, 0) }} events
        </div>
        <div
          v-if="pins.championLocations.show"
          class="ring"
          v-for="(participantFrame, participantId) in level1[0].participantFrames"
          :style="{
            '--ring-hue': match.participants[participantId].teamId === 100 ? 189 : 352,
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
        <div
          v-if="pins.monstersKills.show"
          class="ring"
          v-for="(event in level1.map(level2 => level2.events)
                                 .reduce((allEvts, evts) => allEvts.concat(evts), [])
                                 .filter(evt => evt.type === 'ELITE_MONSTER_KILL')"
          :style="{
            '--ring-hue': 43,
            left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
              event.position.x * positionNormFactor * levels[0].style.width.slice(0, -2) + 'px)',
            bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
              event.position.y * positionNormFactor * levels[0].style.height.slice(0, -2) + 'px)'
          }"
        >
          <img
            class="monster-icon"
            :src="eliteMonsterIcon(event)"
          >
        </div>
        <div
          v-if="pins.buildingsKills.show"
          class="building"
          v-for="(event in level1.map(level2 => level2.events)
                                 .reduce((allEvts, evts) => allEvts.concat(evts), [])
                                 .filter(evt => evt.type === 'BUILDING_KILL')"
          :style="{
            left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
              event.position.x * positionNormFactor * levels[0].style.width.slice(0, -2) + 'px)',
            bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
              event.position.y * positionNormFactor * levels[0].style.height.slice(0, -2) + 'px)'
          }"
        >
          <img
            :src="`https://raw.communitydragon.org/latest/plugins/rcp-fe-lol-match-history/global/default/${
              event.buildingType.split('_')[0].toLowerCase()
            }-${event.teamId}.png`"
          >
        </div>
        <div
          v-if="pins.championMultiKills.show"
          class="champion-kill-multi"
          v-for="(event in level1.map(level2 => level2.events)
                                 .reduce((allEvts, evts) => allEvts.concat(evts), [])
                                 .filter(evt => evt.killType === 'KILL_MULTI')"
          :style="{
            '--background-color': match.participants[event.killerId].teamId === 200 ? '#0D8BA0' : '#B61C30',
            '--multi-kill-length': event.multiKillLength,
            left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
              event.position.x * positionNormFactor * levels[0].style.width.slice(0, -2) + 'px)',
            bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
              event.position.y * positionNormFactor * levels[0].style.height.slice(0, -2) + 'px)'
          }"
        ></div>
        <div
          v-if="pins.championKills.show"
          class="champion-kill"
          v-for="(event in level1.map(level2 => level2.events)
                                 .reduce((allEvts, evts) => allEvts.concat(evts), [])
                                 .filter(evt => evt.type === 'CHAMPION_KILL')"
          @mouseenter="showPreview($event, event)"
          @mouseleave="hidePreview()"
          :style="{
            '--background-color': match.participants[event.victimId].teamId === 100 ? '#0D8BA0' : '#B61C30',
            left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
              event.position.x * positionNormFactor * levels[0].style.width.slice(0, -2) + 'px)',
            bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
              event.position.y * positionNormFactor * levels[0].style.height.slice(0, -2) + 'px)'
          }"
        ></div>
        <img :src="ddragonSRMAP">
      </div>
    </div>

    <div :style="levels[1].style">
      <TransitionGroup name="level-1">
        <div
          v-if="levels[1].show"
          v-for="(level2, index) in levels[1].data"
          :key="levels[1].parentIndex + '-' + index"
          :class="{
            map: true,
            'map-hover': !!level2.events.length,
            'map-no-hover': !level2.events.length,
            level1: true
          }"
          :style="{
            '--index': index,
            'background-color': eliteMonsterBgColor(
              level2.events.filter(evt => evt.type === 'ELITE_MONSTER_KILL')[0]
            )
          }"
          @click="!!level2.events.length ? selectMap(index, 2, level2.events) : null"
        >
          <div class="map-title">
            TIME: {{ minisecToMinute(level2.timestamp) }}
          </div>
          <div
            class="map-num-child"
            :style="{ opacity: levels[2].show ? 0.3 : 1 }"
          >
            {{ level2.events.length }} events
          </div>
          <div
            v-if="pins.championLocations.show"
            class="ring"
            v-for="(participantFrame, participantId) in level2.participantFrames"
            :style="{
              '--ring-hue': match.participants[participantId].teamId === 100 ? 189 : 352,
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
          <div
            v-if="pins.monstersKills.show"
            class="ring"
            v-for="(event in level2.events.filter(evt => evt.type === 'ELITE_MONSTER_KILL')"
            :style="{
              '--ring-hue': 43,
              left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
                event.position.x * positionNormFactor * levels[1].style.width.slice(0, -2) + 'px)',
              bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
                event.position.y * positionNormFactor * levels[1].style.height.slice(0, -2) + 'px)'
            }"
          >
            <img
              class="monster-icon"
              :src="eliteMonsterIcon(event)"
            >
          </div>
          <div
            v-if="pins.buildingsKills.show"
            class="building"
            v-for="(event in level2.events.filter(evt => evt.type === 'BUILDING_KILL')"
            :style="{
              left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
                event.position.x * positionNormFactor * levels[1].style.width.slice(0, -2) + 'px)',
              bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
                event.position.y * positionNormFactor * levels[1].style.height.slice(0, -2) + 'px)'
            }"
          >
            <img
              :src="`https://raw.communitydragon.org/latest/plugins/rcp-fe-lol-match-history/global/default/${
                event.buildingType.split('_')[0].toLowerCase()
              }-${event.teamId}.png`"
            >
          </div>
          <div
            v-if="pins.championMultiKills.show"
            class="champion-kill-multi"
            v-for="(event in level2.events.filter(evt => evt.killType === 'KILL_MULTI')"
            :style="{
              '--background-color': match.participants[event.killerId].teamId === 200 ? '#0D8BA0' : '#B61C30',
              '--multi-kill-length': event.multiKillLength,
              left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
                event.position.x * positionNormFactor * levels[1].style.width.slice(0, -2) + 'px)',
              bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
                event.position.y * positionNormFactor * levels[1].style.height.slice(0, -2) + 'px)'
            }"
          ></div>
          <div
            v-if="pins.championKills.show"
            class="champion-kill"
            v-for="(event in level2.events.filter(evt => evt.type === 'CHAMPION_KILL')"
            @mouseenter="showPreview($event, event)"
            @mouseleave="hidePreview()"
            :style="{
              '--background-color': match.participants[event.victimId].teamId === 100 ? '#0D8BA0' : '#B61C30',
              left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
                event.position.x * positionNormFactor * levels[1].style.width.slice(0, -2) + 'px)',
              bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
                event.position.y * positionNormFactor * levels[1].style.height.slice(0, -2) + 'px)'
            }"
          ></div>
          <img :src="ddragonSRMAP">
        </div>
      </TransitionGroup>
    </div>

    <div :style="levels[2].style">
      <TransitionGroup name="level-1">
        <div
          v-if="levels[2].show"
          v-for="(event, index) in levels[2].data"
          :key="levels[2].parentIndex + '-' + index"
          class="map map-hover level2"
          :style="{
            '--index': index,
            'background-color': eliteMonsterBgColor(
              event.type === 'ELITE_MONSTER_KILL' ? event : null
            ),
            cursor: 'revert'
          }"
        >
          <div class="map-title">
            TIME: {{ minisecToMinute(event.timestamp) }}
          </div>
          <div class="map-num-child map-event-type">{{ event.type }}</div>
          <div
            class="ring"
            v-if="pins.monstersKills.show && event.type === 'ELITE_MONSTER_KILL'"
            :style="{
              '--ring-hue': 43,
              left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
                event.position.x * positionNormFactor * levels[2].style.width.slice(0, -2) + 'px)',
              bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
                event.position.y * positionNormFactor * levels[2].style.height.slice(0, -2) + 'px)'
            }"
          >
            <img
              class="monster-icon"
              :src="eliteMonsterIcon(event)"
            >
          </div>
          <div
            class="building"
            v-if="pins.buildingsKills.show && event.type === 'BUILDING_KILL'"
            :style="{
              left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
                event.position.x * positionNormFactor * levels[1].style.width.slice(0, -2) + 'px)',
              bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
                event.position.y * positionNormFactor * levels[1].style.height.slice(0, -2) + 'px)'
            }"
          >
            <img
              :src="`https://raw.communitydragon.org/latest/plugins/rcp-fe-lol-match-history/global/default/${
                event.buildingType.split('_')[0].toLowerCase()
              }-${event.teamId}.png`"
            >
          </div>
          <div
            v-if="pins.buildingsKills.show && event.killType === 'KILL_MULTI'"
            class="champion-kill-multi"
            :style="{
              '--background-color': match.participants[event.killerId].teamId === 200 ? '#0D8BA0' : '#B61C30',
              '--multi-kill-length': event.multiKillLength,
              left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
                event.position.x * positionNormFactor * levels[2].style.width.slice(0, -2) + 'px)',
              bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
                event.position.y * positionNormFactor * levels[2].style.height.slice(0, -2) + 'px)'
            }"
          ></div>
          <div
            v-if="pins.championKills.show && event.type === 'CHAMPION_KILL'"
            class="champion-kill"
            @mouseenter="showPreview($event, event)"
            @mouseleave="hidePreview()"
            :style="{
              '--background-color': match.participants[event.victimId].teamId === 100 ? '#0D8BA0' : '#B61C30',
              left: 'calc(var(--border-width) + var(--horizontal-padding) + ' +
                event.position.x * positionNormFactor * levels[1].style.width.slice(0, -2) + 'px)',
              bottom: 'calc(var(--border-width) + var(--bottom-padding) + ' +
                event.position.y * positionNormFactor * levels[1].style.height.slice(0, -2) + 'px)'
            }"
          ></div>
          <img :src="ddragonSRMAP">
        </div>
      </TransitionGroup>
    </div>

    <template v-for="selectedIndicator in selectedIndicators">
      <Transition name="selected-indicator">
        <div class="selected-indicator" v-if="selectedIndicator.show" :style="selectedIndicator.style"></div>
      </Transition>
    </template>

    <template v-for="(closeButton, index) in closeButtons">
      <Transition name="close-button">
        <div class="close-button" v-if="closeButton.show" :style="closeButton.style" @click="closeMap(index)">
          <div class="line"></div>
          <div class="line"></div>
        </div>
      </Transition>
    </template>

  </div>
  </div>
  </Transition>
  <Spinner class="spinner" v-if="!levels[0].show" />

  <div
    v-if="previewWindow.show"
    class="preview-window"
    :style=previewWindow.style
  >
    <div
      class="ring"
      :style="{ '--ring-hue': match.participants[previewWindow.killerId].teamId === 100 ? 189 : 352 }">
      <img :src="ddragonChampSquare(match.participants[previewWindow.killerId].champion.id)">
    </div>
    <img src="https://raw.communitydragon.org/latest/plugins/rcp-fe-lol-match-history/global/default/kills.png" alt="">
    <div
      class="ring"
      :style="{ '--ring-hue': match.participants[previewWindow.victimId].teamId === 100 ? 189 : 352 }">
      <img :src="ddragonChampSquare(match.participants[previewWindow.victimId].champion.id)">
    </div>
    <span>{{ previewWindow.timestamp }}</span>
  </div>

</template>

<style scoped>
.spinner {
 position: fixed;
 top: 100px;
 left: 200px;
} 
img {
  width : 100%;
}
.overview {
  width: 700px;
  height: 270px;
  font-family: 'Share Tech Mono', monospace;
  font-size: 20px;
  position: relative;
  margin: 20px 0 100px 50px;
  background: linear-gradient(to right, #0000ff25, #ff000025);
  border-radius: 10px;
}
.overview .blue {
  position: absolute;
  top: 10px;
  left: 10px;
}
.overview .red {
  position: absolute;
  top: 10px;
  right: 10px;
  
}
.overview .blue > div,
.overview .red  > div {
  margin: 5px;
  display: flex;
  gap: 20px;
}
.overview .red  > div  {
  flex-direction: row-reverse;
}
.overview .blue > div > div.text,
.overview .red  > div > div.text {
  flex: 1;
}
.overview .red  > div > div.text {
  text-align: end;
}
.overview .blue > div > div.text >span,
.overview .red  > div > div.text >span {
  font-size: 15px;
  color: #999;
}
.overview img.champion-square {
  align-self: flex-start;
  width: 40px;
  border-radius:50%;
}
.timeline {
  position: relative;
  transform: rotateX(340deg) rotateY(335deg);
  height: 2000px;
  transform-style: preserve-3d;
}
.map {
  position: absolute;
  width:  100%;
  height: 100%;
  transition: transform .8s, opacity .7s ease .3s;
  transform: translateZ(calc(var(--index) * var(--sepZ)));
  background-color: #ddd4;
  padding: var(--top-padding) var(--horizontal-padding) var(--bottom-padding) var(--horizontal-padding);
  border: var(--border-width) solid #eee;
  backdrop-filter: blur(1.5px);
  -webkit-backdrop-filter: blur(1.5px);
  border-radius: 5px;
}
.map-hover:hover { /* use this to prevent still hovering when selected */
  cursor: pointer;
  transform: translateZ(calc(var(--index) * var(--sepZ)))
             translateX(60%);
}
.map-no-hover {
  background-color: #ddd0;
  border: var(--border-width) dashed #eee;
}
.map-no-hover:hover {
  transform: translateZ(calc(var(--index) * var(--sepZ)))
             translateX(60%);
}
.selected {
  opacity: 0;
  transform: translateZ(calc(var(--index) * var(--sepZ)))
             translateX(var(--selected-translateX));
}
.deselected {
  transition: all .5s;
}
.map-title {
  position: absolute;
  font-size: var(--title-font-size);
  font-weight: bold;
  top: 5px;
  font-family: 'Share Tech Mono', monospace;
}
.map-num-child {
  position: absolute;
  font-size: var(--title-font-size);
  bottom: 1%;
  right: -50%;
  font-family: 'Share Tech Mono', monospace;
}
.map-event-type {
  bottom: -13%;
}
.ring {
  position: absolute;
  background: linear-gradient(
    hsl(var(--ring-hue), 80%, 60%),
    hsl(var(--ring-hue), 80%, 15%),
    hsl(var(--ring-hue), 80%, 25%));
  width:  10%;
  height: 10%;
  padding: 1%;
  padding-bottom: .5%;
  border-radius: 50%;
}
img.champion-icon {
  border-radius: 50%;
}
img.monster-icon {
  border-radius: 50%;
}
.building {
  position: absolute;
  width:  10%;
  height: 10%;
}
.champion-kill {
  position: absolute;
  width:  4%;
  height: 4%;
  border-radius: 50%;
  border: 1px solid black;
  background-color: var(--background-color);
}
.champion-kill-multi {
  position: absolute;
  width:  calc(4% * var(--multi-kill-length));
  height: calc(4% * var(--multi-kill-length));
  opacity: .4;
  border-radius: 50%;
  background-color: var(--background-color);
}

.selected-indicator {
  width: var(--width);
  position: absolute;
  border: dotted #eee;
  border-right: none;
  border-radius: 5px;
}

.close-button {
  position: absolute;
  border: 2px solid #eee;
  background-color: #ddd4;
  border-radius: 50%;
  transform: translateZ(var(--translateZ)) translateY(var(--translateY));
  transition: all .3s;
}
.close-button:hover {
  transform: translateZ(var(--translateZ)) translateY(var(--translateY))
             scale(1.2);
  cursor: pointer;
  background-color: #ddd0;
  border-color: #eee0;
}
.close-button .line {
  position: absolute;
  width: 80%;
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

.preview-window {
  transform: rotateX(340deg) rotateY(335deg);
  position: fixed;
  background-color: #040D14EE;
  border: 3px solid #6D5024;
}
.preview-window .ring {
  position: revert;
  display: inline-block;
  background: linear-gradient(
    hsl(var(--ring-hue), 80%, 60%),
    hsl(var(--ring-hue), 80%, 15%),
    hsl(var(--ring-hue), 80%, 25%));
  width:  15%;
  height: revert;
  padding: 1.5%;
  padding-bottom: .5%;
  margin: 3%;
  border-radius: 50%;
}
.preview-window .ring img {
  border-radius: 50%;
}
.preview-window > img {
  width: 15%;
  margin: 0 2%;
}
.preview-window > span {
  position: absolute;
  font-family: 'Share Tech Mono', monospace;
  font-size: 1.7vw;
  top: 27%;
  right: 3%;
}



/* Vue Transition */
.v-enter-active,
.v-leave-active {
  transition: all 0.5s ease-out;
}

.v-enter-from,
.v-leave-to {
  opacity: 0;
  transform: scale(0.8);
}

.level-1-enter-active {
  transition: transform .8s, opacity .01s;
  transition-delay: .5s;
}
.level-1-enter-from {
  opacity: 0;
  transform: translateZ(0);
}
.level-1-leave-active {
  transition: transform .7s, opacity .4s;
}
.level-1-leave-to {
  opacity: 0;
  transform: translateZ(0);
}


.selected-indicator-enter-active,
.selected-indicator-leave-active {
  transition: width .4s;
}
.selected-indicator-enter-from,
.selected-indicator-leave-to {
  width: 0px;
}


.close-button-enter-active,
.close-button-leave-active {
  transition: all .8s;
}
.close-button-enter-from {
  transform: translateZ(var(--translateZ)) translateY(var(--translateY)) translateX(var(--translateX))
             rotate(-180deg);
  opacity: 0;
}
.close-button-leave-to,
.close-button-leave-to:hover {
  transform: translateZ(var(--translateZ)) translateY(var(--translateY)) translateX(var(--translateX))
             scale(1.2) rotate(-180deg);
  cursor: pointer;
  background-color: #ddd0;
  border-color: #eee0;
  opacity: 0;
}


.pin-control {
  position: absolute;
  top: 170px;
  left: 800px;
}
.switch {
  font-family: 'Share Tech Mono', monospace;
  margin: 20px 0 20px 20px;
}
.form-switch {
  display: inline-block;
  cursor: pointer;
  -webkit-tap-highlight-color: transparent;
}
.form-switch i {
  position: relative;
  display: inline-block;
  margin-right: .5rem;
  width: 46px;
  height: 26px;
  background-color: #e6e6e6;
  border-radius: 23px;
  vertical-align: text-bottom;
  transition: all 0.3s linear;
}
.form-switch i::before {
  content: "";
  position: absolute;
  left: 0;
  width: 42px;
  height: 22px;
  background-color: #fff;
  border-radius: 11px;
  transform: translate3d(2px, 2px, 0) scale3d(1, 1, 1);
  transition: all 0.25s linear;
}
.form-switch i::after {
  content: "";
  position: absolute;
  left: 0;
  width: 22px;
  height: 22px;
  background-color: #fff;
  border-radius: 11px;
  box-shadow: 0 2px 2px rgba(0, 0, 0, 0.24);
  transform: translate3d(2px, 2px, 0);
  transition: all 0.2s ease-in-out;
}
.form-switch:active i::after {
  width: 28px;
  transform: translate3d(2px, 2px, 0);
}
.form-switch:active input:checked + i::after { transform: translate3d(16px, 2px, 0); }
.form-switch input { display: none; }
.form-switch input:checked + i { background-color: #B3944C; }
.form-switch input:checked + i::before { transform: translate3d(18px, 2px, 0) scale3d(0, 0, 0); }
.form-switch input:checked + i::after { transform: translate3d(22px, 2px, 0); }
</style>
