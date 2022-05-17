<script setup>
import { useRoute } from 'vue-router'
import JsonViewer from 'vue-json-viewer'

</script>

<template>
  MatchId: {{ useRoute().params.dataURI }}

  <!-- <JsonViewer :value="timeline" /> -->
</template>
