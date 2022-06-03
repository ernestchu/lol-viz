<script setup>
import { ref, reactive } from 'vue'
import { RouterLink } from 'vue-router'

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
  })
  .catch(() => {
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

  <ul v-if="challengerLeague">
    <li v-for="(value, key) in challengerLeague">
      <ul v-if="key === 'entries'">
        <li v-for="entry in value">

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
        </li>
      </ul>
      <template v-else>{{ key }}: {{ value }}</template>
    </li>
  </ul>
</template>

<style scoped>
.summoner-name {
  color: #B3944C;
  cursor: pointer;
}
</style>
