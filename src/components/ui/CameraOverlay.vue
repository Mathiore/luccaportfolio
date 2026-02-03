<script setup>
import { computed } from 'vue';

const props = defineProps({
  scrollProgress: {
    type: Number,
    required: true,
    default: 0
  }
});

const overlayStyle = computed(() => {
  const opacity = props.scrollProgress > 0.8 ? (props.scrollProgress - 0.8) * 5 : 0;
  return { opacity };
});
</script>

<template>
  <div class="viewfinder-overlay" :style="overlayStyle">
    <div class="ocr-text top-left">REC</div>
    <div class="ocr-text top-right">BAT 100%</div>
    <div class="ocr-text bottom-left">ISO 800</div>
    <div class="ocr-text bottom-right">1/60 F2.8</div>
  </div>
</template>

<style scoped>
.viewfinder-overlay {
    position: absolute;
    top: 0; 
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    transition: opacity 0.1s linear;
    z-index: 20;
    border: 20px solid rgba(0,0,0,0.8);
}

.ocr-text {
    position: absolute;
    color: white;
    font-family: 'Courier New', Courier, monospace;
    font-weight: bold;
    font-size: 1.2rem;
    text-shadow: 0 0 5px black;
    padding: 30px;
}

.top-left { top: 0; left: 0; color: red; animation: blink 1s infinite; }
.top-right { top: 0; right: 0; }
.bottom-left { bottom: 0; left: 0; }
.bottom-right { bottom: 0; right: 0; }

@keyframes blink {
    50% { opacity: 0; }
}
</style>
