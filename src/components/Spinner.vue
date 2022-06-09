<template>
  <div class="container">
    <transition>
      <span>
        <div class="bar1"></div>
        <div class="bar2"></div>
        <div class="bar3"></div>
        <div class="bar4"></div>
        <div class="bar5"></div>
        <div class="bar6"></div>
        <div class="bar7"></div>
        <div class="bar8"></div>
        <div class="bar9"></div>
        <div class="bar10"></div>
        <div class="bar11"></div>
        <div class="bar12"></div>
      </span>
    </transition>
  </div>
</template>

<style scoped>
div.container {
  transform: scale(2);
  position: relative;
  width: 25px;
  height: 25px;
  display: inline-block;
  padding: 5px;
  --color: #999;
}

div.container div[class^='bar'] {
  width: 6%;
  height: 16%;
  background: var(--color);
  position: absolute;
  left: 49%;
  top: 43%;
  opacity: 0;
  -webkit-border-radius: 50px;
  -webkit-box-shadow: 0 0 3px rgba(0, 0, 0, 0.2);
  -webkit-animation: fade 1s linear infinite;
}

@-webkit-keyframes fade {
  from {
    opacity: 1;
  }
  to {
    opacity: 0.25;
  }
}

div.container div.bar1 {
  -webkit-transform: rotate(0deg) translate(0, -130%);
  -webkit-animation-delay: 0s;
}

div.container div.bar2 {
  -webkit-transform: rotate(30deg) translate(0, -130%);
  -webkit-animation-delay: -0.9167s;
}

div.container div.bar3 {
  -webkit-transform: rotate(60deg) translate(0, -130%);
  -webkit-animation-delay: -0.833s;
}
div.container div.bar4 {
  -webkit-transform: rotate(90deg) translate(0, -130%);
  -webkit-animation-delay: -0.7497s;
}
div.container div.bar5 {
  -webkit-transform: rotate(120deg) translate(0, -130%);
  -webkit-animation-delay: -0.667s;
}
div.container div.bar6 {
  -webkit-transform: rotate(150deg) translate(0, -130%);
  -webkit-animation-delay: -0.5837s;
}
div.container div.bar7 {
  -webkit-transform: rotate(180deg) translate(0, -130%);
  -webkit-animation-delay: -0.5s;
}
div.container div.bar8 {
  -webkit-transform: rotate(210deg) translate(0, -130%);
  -webkit-animation-delay: -0.4167s;
}
div.container div.bar9 {
  -webkit-transform: rotate(240deg) translate(0, -130%);
  -webkit-animation-delay: -0.333s;
}
div.container div.bar10 {
  -webkit-transform: rotate(270deg) translate(0, -130%);
  -webkit-animation-delay: -0.2497s;
}
div.container div.bar11 {
  -webkit-transform: rotate(300deg) translate(0, -130%);
  -webkit-animation-delay: -0.167s;
}
div.container div.bar12 {
  -webkit-transform: rotate(330deg) translate(0, -130%);
  -webkit-animation-delay: -0.0833s;
}
.checkmark {
  display: inline-block;
  position: relative;
  left: -5px;
  transform: rotate(45deg);
  height: 12px;
  width: 6px;
  margin-left: 60%;
  border-bottom: 3px solid var(--color);
  border-right: 3px solid var(--color);
}

.v-enter-active,
.v-leave-active {
  transition: opacity 0.5s ease;
}
.v-enter-from,
.v-leave-to {
  opacity: 0;
}
</style>
