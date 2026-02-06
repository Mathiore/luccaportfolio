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
  
  let opacity = 1;
  const startFade = 0.8; // Start fading heavily at the end
  
  if (props.scrollProgress > startFade) {
      opacity = Math.max(0, 1 - ((props.scrollProgress - startFade) / (1 - startFade)));
  }

  return {
    backgroundColor: 'white',
    opacity: opacity
  };
});

const textStyle = computed(() => {
  // No longer needed to manage opacity separately, handled by parent
  return {};
});
</script>

<template>
  <div class="background-layer" :style="backgroundStyle">
    <div class="text-wrapper" :style="textStyle">
        <!-- Desktop SVG Implementation -->
        <svg class="text-svg desktop-only" viewBox="0 0 1000 300" preserveAspectRatio="xMidYMid meet">
            <defs>
                <mask id="video-mask" maskUnits="userSpaceOnUse">
                    <rect x="0" y="0" width="1000" height="300" fill="black" />
                    <text x="500" y="150" text-anchor="middle" dominant-baseline="middle" fill="white" font-family="Anton" font-weight="bold" font-size="200" dy=".1em">LUCCA NUNES</text>
                </mask>
            </defs>
            <foreignObject x="0" y="0" width="1000" height="300" mask="url(#video-mask)">
                 <video 
                    class="in-text-video" 
                    autoplay 
                    muted 
                    loop 
                    playsinline
                    webkit-playsinline
                    crossorigin="anonymous"
                    src="/carsmov.mp4">
                 </video>
            </foreignObject>
        </svg>

        <!-- Mobile CSS Implementation (Stencil) -->
        <div class="mobile-stencil-container">
            <video 
                class="mobile-video" 
                autoplay 
                muted 
                loop 
                playsinline
                webkit-playsinline
                crossorigin="anonymous"
                src="/carsmov.mp4">
            </video>
            <div class="stencil-overlay">
                <span>LUCCA</span>
                <span>NUNES</span>
            </div>
        </div>
    </div>
    <img src="/models/lucca.png" class="profile-bg" :style="textStyle" alt="" />
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Anton&display=swap');

.desktop-only {
    display: block;
}
.mobile-stencil-container {
    display: none;
}

.profile-bg {
    position: absolute;
    bottom: -30vh; 
    left: 50%;
    transform: translateX(-50%);
    height: 90vh; 
    max-width: 90vw;
    object-fit: contain;
    pointer-events: none;
    z-index: 2; 
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
    z-index: 0; 
    background-color: white; 
}

.text-wrapper {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    pointer-events: none;
    transition: opacity 0.1s linear;
    width: 100%;
    height: 100%;
    z-index: 1; 
    position: absolute;
    top: 0;
    left: 0;
    overflow: hidden; 
}
/* Removed old .masked-title styles as they seem unused or legacy */

.text-svg {
    width: 100%;
    height: auto;
}

.in-text-video {
    width: 100%;
    height: 100%;
    object-fit: cover;
}

@media (max-width: 768px) {
    .desktop-only {
        display: none !important;
    }
    
    .mobile-stencil-container {
        display: grid; /* Stack children */
        place-items: center;
        width: 100%;
        height: auto;
        position: relative;
        margin-top: 15vh;
    }

    .mobile-video {
        grid-area: 1 / 1;
        width: 100%;
        height: 100%;
        max-height: 40vh; /* Increased to fit stacked text */
        object-fit: cover;
    }

    .stencil-overlay {
        grid-area: 1 / 1;
        width: 101%; /* Slight overlap to prevent cracks */
        height: 101%;
        background-color: white; /* The "Mask" Color */
        color: black; /* The "Hole" Color */
        display: flex;
        flex-direction: column; /* Stack vertically */
        align-items: center;
        justify-content: center;
        font-family: 'Anton', sans-serif;
        font-size: 35vw; /* Much larger font */
        font-weight: bold;
        line-height: 0.85;
        mix-blend-mode: screen; /* Magic: White stays White, Black shows Video */
        z-index: 2;
    }

    .profile-bg {
        bottom: -23vh; 
        height: 85vh;
    }
    
    .text-wrapper {
        justify-content: flex-start; 
        padding-top: 0; /* Remove padding as container has margin */
    }
}



</style>
