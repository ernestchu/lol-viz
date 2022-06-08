<script setup>
const { win, bandwidth } = defineProps([ 'win', 'bandwidth' ])
let style = {
  '--scale': bandwidth / 195,
  '--color1': 'purple',
  '--color2': 'blue',
  '--color3': '#4E6A49',
  // '--color1': 'purple',
  // '--color2': '#CEAFBB', 
  // '--color3': '#4F5D75', 
}
if (!win) {
  // style.filter = 'grayscale(1)'
  style = {
    '--scale': bandwidth / 195,
    // '--color1': 'purple',
    // '--color2': 'blue',
    // '--color3': '#4E6A49',
    '--color1': 'black',
    '--color2': 'red', 
    '--color3': '#4F5D75', 
  }
}
</script>
<template>
<div :style="style">
  <div class="lava"></div>
  <div class="sphere"></div>
</div>
</template>

<style scoped>
.sphere {
  position: absolute;
  left: 50%;
  top: 50%;
  border-radius: 100%;
  transform: translate(-50%, -50%) rotate(30deg) scale(var(--scale));
  width: 195px;
  height: 195px;
  box-shadow: 1px 10px 50px -20px, inset -1px 30px 50px -10px #fff, inset -1px -30px 50px -10px #210, inset 0 120px 7px -70px rgba(255, 255, 240, 0.2);
  z-index: 50;
}

.lava {
  -webkit-filter: blur(1px) contrast(20) blur(15px);
  position: absolute;
  left: 50%;
  top: 50%;
  overflow: hidden;
  border-radius: 100%;
  transform: translate(-50%, -50%) scale(var(--scale));
  width: 190px;
  height: 190px;
  box-shadow: 0 0 50px -20px var(--color1);
  z-index: 10;
}

.lava:before {
  -webkit-filter: blur(8px);
  content: "";
  display: block;
  position: relative;
  left: 50%;
  top: 50%;
  opacity: 0.8;
  border-radius: 100%;
  transform: translate(-50%, -50%);
  width: 200px;
  height: 200px;
  -webkit-animation: spin2 20s infinite linear;
          animation: spin2 20s infinite linear;
  background-color: var(--color1);
  background-image: linear-gradient(transparent 29.29%, var(--color3) 29.29%, var(--color2) 70.71%, transparent 70.71%), linear-gradient(90deg, transparent 29.29%, var(--color3) 29.29%, var(--color2) 70.71%, transparent 70.71%);
  z-index: 10;
  box-shadow: 0 0 50px -20px var(--color1);
}

.lava:after {
  -webkit-filter: blur(5px);
  content: "";
  display: block;
  position: absolute;
  opacity: 0.5;
  left: 50%;
  top: 50%;
  border-radius: 100%;
  transform: translate(-50%, -50%) rotate(45deg);
  width: 200px;
  height: 200px;
  -webkit-animation: spin1 10s infinite linear;
          animation: spin1 10s infinite linear;
  background: linear-gradient(transparent 29.29%, var(--color3) 29.29%, var(--color2) 70.71%, transparent 70.71%), linear-gradient(90deg, transparent 29.29%, var(--color3) 29.29%, var(--color2) 70.71%, transparent 70.71%);
  z-index: 10;
  box-shadow: 0 0 50px -20px var(--color1);
}

@-webkit-keyframes spin1 {
  0% {
    transform: translate(-50%, -50%) rotate(45deg);
  }
  100% {
    transform: translate(-50%, -50%) rotate(405deg);
  }
}

@keyframes spin1 {
  0% {
    transform: translate(-50%, -50%) rotate(45deg);
  }
  100% {
    transform: translate(-50%, -50%) rotate(405deg);
  }
}
@-webkit-keyframes spin2 {
  0% {
    transform: translate(-50%, -50%) rotate(360deg);
  }
  100% {
    transform: translate(-50%, -50%) rotate(0deg);
  }
}
@keyframes spin2 {
  0% {
    transform: translate(-50%, -50%) rotate(360deg);
  }
  100% {
    transform: translate(-50%, -50%) rotate(0deg);
  }
}
</style>
