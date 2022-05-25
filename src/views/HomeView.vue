<script setup>
import { ref, reactive } from 'vue'
import { RouterLink } from 'vue-router'

const challengerLeague = reactive({})
const authFailed = ref(false)

const proxyHost = import.meta.env.VITE_PROXY_HOST
const targetURI = '/lol/league/v4/challengerleagues/by-queue/RANKED_SOLO_5x5'

fetch(proxyHost + targetURI)
  .then(res => res.json())
  .then(data => {
    Object.assign(challengerLeague, data)
    challengerLeague.entries.forEach(entry => {
      entry.show = false
    })
  })
  .catch(() => {
    authFailed.value = true
  })
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
  color: blue;
  cursor: pointer;
}
</style>
