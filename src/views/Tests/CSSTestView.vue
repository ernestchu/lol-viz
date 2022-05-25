<script setup>
const maps = [  0, 1, 2, 3, 4 ]

function selectMap (event) {
  event.target.parentNode.classList.add('selected')
  event.target.parentNode.classList.remove('map-hover')
}
</script>

<template>
  <div class="timeline">
    <template v-for="index in maps" :key="index">
      <div class="map map-hover" :style="{ '--index': index }" @click="selectMap">
        <img src="http://ddragon.leagueoflegends.com/cdn/6.8.1/img/map/map11.png" width="300">
      </div>
    </template>
  </div>
</template>

<style scoped>
.timeline {
  margin-left: 400px;
  width: 300px;
  height: 300px;
  position: relative;
  transform-style: preserve-3d;
  transform: rotateX(335deg) rotateY(335deg);
}
.map {
  position: absolute;
  width: 100%;
  height: 100%;
  transition: transform .4s;
  transform: translateZ(calc(var(--index) * 150px));
}
.map-hover:hover { /* use this to prevent still hovering when selected */
  cursor: pointer;
  transform: translateZ(calc(var(--index) * 150px))
             translateX(30px);
}
.selected {
  transform: rotateY(-335deg) rotateX(-335deg) translate(400px, 200px);
}
</style>
