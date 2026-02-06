<script setup>
import { onMounted, onUnmounted, ref } from 'vue';

const scrollContainer = ref(null);

const scroll = (direction) => {
  if (scrollContainer.value) {
    const scrollAmount = window.innerWidth * 0.8; // Scroll mostly a full page
    const currentScroll = scrollContainer.value.scrollLeft;
    const target = direction === 'left' ? currentScroll - scrollAmount : currentScroll + scrollAmount;
    
    scrollContainer.value.scrollTo({
      left: target,
      behavior: 'smooth'
    });
  }
};

defineProps({
  album: {
    type: Object,
    required: true
  }
});

defineEmits(['close']);

// Lock body scroll when open
onMounted(() => {
  document.body.style.overflow = 'hidden';
});

onUnmounted(() => {
  document.body.style.overflow = '';
});
</script>

<template>
  <div class="album-modal" @click.self="$emit('close')">
    <div class="album-content">
      <button class="close-btn" @click="$emit('close')">✕</button>
      <h2 class="album-title">{{ album.title }}</h2>
      
      <div ref="scrollContainer" class="photo-grid">
        <div v-for="(img, index) in album.images" :key="index" class="photo-item">
            <div class="photo-frame">
                <img :src="img" :alt="album.title + ' ' + index" loading="lazy" />
            </div>
        </div>
      </div>
      
      <!-- Mobile Navigation Arrows -->
      <button class="nav-arrow left" @click="scroll('left')">‹</button>
      <button class="nav-arrow right" @click="scroll('right')">›</button>
    </div>
  </div>
</template>

<style scoped>

@import url('https://fonts.googleapis.com/css2?family=Meie+Script&display=swap');

.album-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background-color: rgba(0, 0, 0, 0.9);
  z-index: 1000;
  display: flex;
  justify-content: center;
  align-items: center;
  backdrop-filter: blur(5px);
  animation: fadeIn 0.3s ease-out;
}

.album-content {
  width: 90%;
  height: 90%;
  background: transparent;
  display: flex;
  flex-direction: column;
  position: relative;
}

.close-btn {
  position: absolute;
  top: 0;
  right: 0;
  background: none;
  border: none;
  color: white;
  font-size: 2rem;
  cursor: pointer;
  z-index: 1010;
  font-family: 'Courier New', Courier, monospace;
}

.nav-arrow {
  display: none; /* Hidden on Desktop */
  position: absolute;
  top: 55%;
  transform: translateY(-50%);
  background: rgba(0,0,0,0.3);
  color: white;
  border: none;
  font-size: 4rem;
  padding: 0 15px;
  cursor: pointer;
  z-index: 1010;
  border-radius: 5px;
  transition: background 0.3s;
  user-select: none;
}

.nav-arrow:hover {
  background: rgba(0,0,0,0.6);
}

.nav-arrow.left { left: 0; }
.nav-arrow.right { right: 0; }

.album-title {
  font-family: 'Meie Script', cursive;
  color: #D4AF37; /* Gold */
  font-size: 4rem;
  text-align: center;
  margin: 0 0 20px 0;
  text-shadow: 0 2px 4px rgba(0,0,0,0.5);
}

.photo-grid {
  flex: 1;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 20px;
  overflow-y: auto;
  padding: 20px;
  /* Scrollbar styling */
  scrollbar-width: thin;
  scrollbar-color: #D4AF37 transparent;
}

.photo-grid::-webkit-scrollbar {
  width: 8px;
}
.photo-grid::-webkit-scrollbar-track {
  background: transparent;
}
.photo-grid::-webkit-scrollbar-thumb {
  background-color: #D4AF37;
  border-radius: 4px;
}

.photo-item {
  display: flex;
  justify-content: center;
  align-items: center;
  opacity: 0;
  animation: slideUp 0.5s ease-out forwards;
}

/* Stagger animations if possible, but pure CSS index staggering is hard without dynamic style. 
   Just simple fade in for now. */

.photo-frame {
  background-color: white;
  padding: 10px; /* White border effect */
  box-shadow: 0 4px 15px rgba(0,0,0,0.5);
  transition: transform 0.3s ease;
  width: 100%;
  aspect-ratio: 1/1; /* Square contour */
}

.photo-frame:hover {
  transform: scale(1.05) rotate(1deg);
  z-index: 10;
}

.photo-frame img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
  filter: grayscale(20%) contrast(110%); /* Analog feel */
  transition: filter 0.3s ease;
}

.photo-frame:hover img {
    filter: grayscale(0%) contrast(100%);
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes slideUp {
  from { opacity: 0; transform: translateY(20px); }
  to { opacity: 1; transform: translateY(0); }
}

/* Mobile adjustments (Carousel) */
@media (max-width: 768px) {
    .album-title {
        font-size: 3rem;
    }
    
    .photo-grid {
        display: flex; /* Horizontal layout */
        flex-direction: row;
        overflow-x: auto;
        overflow-y: hidden;
        grid-template-columns: none; /* Reset grid */
        gap: 0; /* Gap handled by padding or just margin if needed, but 0 is fine for full width */
        scroll-snap-type: x mandatory;
        padding: 0; /* Remove padding to full bleed or keep minimal */
        align-items: center;
        -webkit-overflow-scrolling: touch;
        scrollbar-width: none; /* Hide scrollbar for cleaner look */
    }
    
    .photo-grid::-webkit-scrollbar {
        display: none;
    }

    .photo-item {
        flex: 0 0 100%; /* Full width */
        width: 100%;
        scroll-snap-align: center;
        padding: 20px; /* Padding inside the item to show gap/frame */
        box-sizing: border-box;
        height: auto; /* Let it adapt */
    }

    .photo-frame {
        padding: 10px;
        box-shadow: 0 4px 10px rgba(0,0,0,0.3);
        aspect-ratio: 1/1; 
    }
    
    .nav-arrow {
        display: block;
    }
}
</style>
