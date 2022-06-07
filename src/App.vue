<script setup>
import { RouterView, RouterLink } from 'vue-router'

const logoURL = `${import.meta.env.BASE_URL}images/logo.png`
</script>

<template>
  <RouterLink :to="{ name: 'home' }">
    <div id="logo">
      <img :src="logoURL">
      <span>VISUALIZER</span>
    </div>
  </RouterLink>
  <RouterView />
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=Share+Tech+Mono&display=swap');

* {
  color: rgba(255, 255, 255, 0.87);
}
body {
  background-color: #0F171E;
  /* overflow: auto; */
  /* position: relative; */
}
body:before {
  content: "";
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: -1;

  /* display: block; */
  background-image: url('https://images.contentstack.io/v3/assets/blt731acb42bb3d1659/bltc28d43095579c8b5/5d87c24e8d8a3163705e3070/summonersrift.jpg');
  background-size: cover;
  /* width: 100%; */
  /* height: 100%; */

  /* -webkit-filter: blur(5px); */
  /* -moz-filter: blur(5px); */
  /* -o-filter: blur(5px); */
  /* -ms-filter: blur(5px); */
  filter: blur(7px);
  transform: scale(1.02); /* remove white border */
}
a {
  color: #B3944C;
}

#logo {
  position: relative;
}
#logo img {
  width: 300px;
}
#logo span{
  position: absolute;
  top: 20px;
  padding-left: 10px;
  font-family: 'Share Tech Mono', monospace;
  font-size: 90px;
  background: linear-gradient(to bottom right, #946E32, #E0CA75);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  transform: scaleY(1.5);
}
#logo span a:hover, a:visited, a:link, a:active {
  text-decoration: none;
}
</style>
