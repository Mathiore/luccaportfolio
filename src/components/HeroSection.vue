<script setup>
import { onMounted, ref } from 'vue';

const isLoaded = ref(false);

onMounted(() => {
  // Trigger animations after mount
  setTimeout(() => {
    isLoaded.value = true;
  }, 100);
});
</script>

<template>
  <section class="hero-container">
    <!-- Background Layer -->
    <div class="background-layer">
      <picture>
        <source media="(max-width: 768px)" srcset="../assets/carFundoMobile.png" />
        <img src="../assets/carrofundo.png" alt="Garage Background" class="bg-img" />
      </picture>
      <div class="overlay"></div>
    </div>

    <!-- Text Layer (Middle) -->
    <div class="text-layer" :class="{ 'animate-text': isLoaded }">
      <h1 class="main-title">
        <span class="block">FOTOGRAFIA</span>
        <span class="block highlight">PROFISSIONAL</span>
      </h1>
      <p class="subtitle">LUCCA NUNES</p>
    </div>

    <!-- Foreground Layer (Car) -->
    <div class="car-layer">
      <picture>
        <source media="(max-width: 768px)" srcset="../assets/carFrenteMobile.png" />
        <img src="../assets/carroFrente.png" alt="Car Front" class="car-img" :class="{ 'animate-car': isLoaded }" />
      </picture>
    </div>
    
    <!-- Scroll indicator -->
    <div class="scroll-indicator">
        <div class="mouse"></div>
    </div>
  </section>
</template>

<style scoped>
.hero-container {
  position: relative;
  width: 100%;
  height: 100vh;
  overflow: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: #000;
}

/* Background */
.background-layer {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
}

.bg-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  opacity: 0.6; /* Dimmed for better text contrast */
}

.overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(to bottom, rgba(0,0,0,0.3) 0%, rgba(0,0,0,0.8) 100%);
}

/* Text Layer - Behind the car */
.text-layer {
  position: absolute;
  z-index: 2; /* Middle layer */
  text-align: center;
  top: 40%; /* Initial position */
  left: 50%;
  transform: translate(-50%, 20%); /* Start slightly lower */
  opacity: 0;
  transition: all 1.5s cubic-bezier(0.22, 1, 0.36, 1);
  width: 100%;
}

.animate-text {
  opacity: 1;
  transform: translate(-50%, -30%); /* Moved up further from -10% */
}

.main-title {
  font-family: 'Outfit', sans-serif;
  font-weight: 900;
  font-size: clamp(3rem, 8vw, 8rem);
  line-height: 0.9;
  text-transform: uppercase;
  color: #fff;
  letter-spacing: -2px;
  text-shadow: 0 10px 30px rgba(0,0,0,0.5);
}

.highlight {
  color: #fff; /* Changed from transparent to match FOTOGRAFIA */
  display: block;
}

.subtitle {
  margin-top: 1rem;
  font-size: 1.2rem;
  letter-spacing: 4px;
  color: rgba(255,255,255,0.6);
  text-transform: uppercase;
}

/* Car Layer - Front */
.car-layer {
  position: absolute;
  bottom: 0px; 
  left: 50%;
  transform: translateX(-50%);
  z-index: 3; /* Top layer */
  width: 100%;
  display: flex;
  justify-content: center;
  pointer-events: none; /* Allow clicking through if needed */
}

.car-img {
  width: auto;
  max-width: 100%; /* Increased from 90% */
  height: auto;
  max-height: 100%; /* Increased from 80vh */
  object-fit: cover;
  filter: drop-shadow(0 -10px 50px rgba(0,0,0,0.5));
  
  /* Initial state */
  transform: translateY(100px) scale(0.95);
  opacity: 0;
  transition: all 1.2s cubic-bezier(0.22, 1, 0.36, 1) 0.2s; /* Delay slightly */
}

.car-img.animate-car {
  transform: translateY(0) scale(1.1); /* Slight scale up for extra impact */
  opacity: 1;
}

/* Scroll Indicator */
.scroll-indicator {
    position: absolute;
    bottom: 30px;
    z-index: 10;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    flex-direction: column;
    align-items: center;
    opacity: 0.7;
}

.mouse {
    width: 26px;
    height: 42px;
    border: 2px solid #fff;
    border-radius: 20px;
    position: relative;
}

.mouse::before {
    content: '';
    position: absolute;
    top: 6px;
    left: 50%;
    transform: translateX(-50%);
    width: 4px;
    height: 4px;
    background: #fff;
    border-radius: 50%;
    animation: scroll 2s infinite;
}

@keyframes scroll {
    0% { transform: translate(-50%, 0); opacity: 1; }
    100% { transform: translate(-50%, 15px); opacity: 0; }
}

/* Responsive adjustments */
@media (max-width: 768px) {
    .main-title {
        font-size: 3.5rem;
    }
    .car-img {
        max-width: 120%; /* Bleed out on mobile */
    }
}
</style>
