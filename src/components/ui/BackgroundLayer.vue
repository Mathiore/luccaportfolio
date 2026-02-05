<script setup>
import { computed } from 'vue';

const props = defineProps({
  scrollProgress: {
    type: Number,
    required: true,
    default: 0
  },
  isLoading: {
    type: Boolean,
    default: false
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
    <img src="/models/lucca.png" class="profile-bg" :style="textStyle" alt="" />
    <div class="text-wrapper" :style="textStyle">
        <h1 class="cursive-bg-text">Lucca Nunes</h1>    
    </div>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Barriecito&family=Pirata+One&display=swap');

.profile-bg {
    position: absolute;
    bottom: -23vh;
    left: 50%;
    transform: translateX(-50%);
    height: 85vh; /* Large height to stand behind camera */
    max-width: 90vw;
    object-fit: contain;
    pointer-events: none;
    z-index: -1; /* Ensure it is behind text if overlap */
    /* Opacity controlled by inline style binding */
}

.background-layer {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: flex-start;
    padding-top: 5vh;
    z-index: 0; /* Behind canvas */
    background-color: white; /* Start White */
}

.text-wrapper {
    display: flex;
    flex-direction: column;
    align-items: center;
    pointer-events: none;
    transition: opacity 0.1s linear;
}

.cursive-bg-text {
    font-family: 'Pirata One', cursive;
    font-size: 22rem; /* Very large */
    color: #d10000ff;
    margin: 0;
    pointer-events: none;
    text-align: center;
    line-height: 1;
}

@media (max-width: 768px) {
    .cursive-bg-text {
        font-size: 10rem; /* Smaller for mobile to fit */
    }
    .subtitle-text {
        font-size: 3rem;
    }
}
</style>
