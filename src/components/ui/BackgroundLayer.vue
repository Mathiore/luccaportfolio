<script setup>
import { computed } from 'vue';

const props = defineProps({
  scrollProgress: {
    type: Number,
    required: true,
    default: 0
  }
});

const backgroundStyle = computed(() => {
  const val = Math.floor(255 * (1 - props.scrollProgress));
  return {
    backgroundColor: `rgb(${val}, ${val}, ${val})`
  };
});

const textStyle = computed(() => {
  let opacity = 0.8;
  if (props.scrollProgress > 0.7) {
      const fadeOp = Math.max(0, 1 - ((props.scrollProgress - 0.7) / 0.25));
      opacity = fadeOp * 0.8;
  }
  return { opacity };
});
</script>

<template>
  <div class="background-layer" :style="backgroundStyle">
    <h1 class="cursive-bg-text" :style="textStyle">Lucca Nunes</h1>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Mrs+Saint+Delafield&display=swap');

.background-layer {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 0; /* Behind canvas */
    background-color: white; /* Start White */
}

.cursive-bg-text {
    font-family: 'Mrs Saint Delafield', cursive;
    font-size: 22rem; /* Very large */
    color: #b91004ff; /* Gold */
    margin: 0;
    opacity: 0.8;
    pointer-events: none;
    text-align: center;
    line-height: 1;
}

@media (max-width: 768px) {
    .cursive-bg-text {
        font-size: 12rem; /* Larger for mobile */
    }
}
</style>
