<script setup>
import { defineEmits, ref } from 'vue';
import { albums } from '../../config/albums.js';

const emit = defineEmits(['select-folder', 'open-album']);

const hoverIndex = ref(null);
const galleryWrapper = ref(null);

const selectAlbum = (album) => {
  emit('open-album', album);
};

const selectLink = (id) => {
    emit('select-folder', id);
};

const handleWheel = (e) => {
    if (galleryWrapper.value) {
        // Convert vertical scroll to horizontal
        e.preventDefault();
        galleryWrapper.value.scrollLeft += e.deltaY;
    }
};

const scrollGallery = (direction) => {
    if (galleryWrapper.value) {
        const scrollAmount = window.innerWidth * 0.5;
        const current = galleryWrapper.value.scrollLeft;
        galleryWrapper.value.scrollTo({
            left: direction === 'left' ? current - scrollAmount : current + scrollAmount,
            behavior: 'smooth'
        });
    }
};
</script>

<template>
  <div class="portfolio-container">
    <!-- Header / Nav -->
    <header class="portfolio-header">
        <div class="logo">
            <span class="rec-dot"></span>
            REC
        </div>
        <nav class="nav-links">
            <a href="#" @click.prevent="selectLink('about')">SOBRE</a>
            <a href="#" @click.prevent="selectLink('contact')">CONTATO</a>
        </nav>
    </header>

    <!-- Main Gallery (Horizontal Scroll) -->
    <div class="gallery-container-relative">
        <button class="nav-arrow left desktop-only" @click="scrollGallery('left')">‹</button>
        
        <div 
            class="gallery-wrapper" 
            ref="galleryWrapper"
            @wheel="handleWheel"
        >
            <div class="gallery-track">
                <div 
                    v-for="(album, index) in albums" 
                    :key="album.id" 
                    class="album-card"
                    @mouseenter="hoverIndex = index"
                    @mouseleave="hoverIndex = null"
                    @click="selectAlbum(album)"
                >
                    <div class="image-container">
                        <img :src="album.cover" :alt="album.title" loading="lazy" />
                        <div class="overlay"></div>
                    </div>
                    <div class="card-info">
                        <span class="album-number">{{ (index + 1).toString().padStart(2, '0') }}</span>
                        <h3 class="album-title">{{ album.title }}</h3>
                        <div class="view-btn">
                            <span>VER PROJETO</span>
                            <svg width="20" height="20" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
                                <path d="M5 12H19M19 12L12 5M19 12L12 19" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
                            </svg>
                        </div>
                    </div>
                </div>
            </div>
        </div>
        
        <button class="nav-arrow right desktop-only" @click="scrollGallery('right')">›</button>
    </div>

    <!-- Footer / Micro interaction -->
    <div class="lens-view-indicator">
        <div class="lens-crosshair"></div>
        <span>LENS VIEW: ACTIVATED</span>
    </div>
  </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Anton&family=Inter:wght@300;400;600&display=swap');

.portfolio-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: black; /* Deep black base */
    color: white;
    display: flex;
    flex-direction: column;
    z-index: 50;
    font-family: 'Inter', sans-serif;
    overflow: hidden;
    animation: fadeIn 0.8s ease-out;
}

@keyframes fadeIn {
    from { opacity: 0; transform: scale(1.05); }
    to { opacity: 1; transform: scale(1); }
}

/* HEADER */
.portfolio-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 2rem 5vw;
    z-index: 10;
}

.logo {
    font-family: 'Anton', sans-serif;
    font-size: 1rem;
    letter-spacing: 2px;
    display: flex;
    align-items: center;
    color: #ff0000;
    animation: blink 2s infinite;
    gap: 10px;
}

.rec-dot {
    width: 12px;
    height: 12px;
    background-color: #ff0000;
    border-radius: 50%;
    display: inline-block;
    animation: blink 2s infinite;
    box-shadow: 0 0 5px #ff0000;
}

@keyframes blink {
    0% { opacity: 1; }
    50% { opacity: 0; }
    100% { opacity: 1; }
}

.nav-links {
    display: flex;
    gap: 2rem;
}

.nav-links a {
    color: rgba(255,255,255,0.6);
    text-decoration: none;
    font-size: 0.9rem;
    letter-spacing: 1px;
    font-weight: 600;
    transition: color 0.3s;
}

.nav-links a:hover {
    color: white;
}

/* GALLERY */
.gallery-container-relative {
    flex: 1;
    position: relative;
    display: flex;
    align-items: center;
    overflow: hidden; /* Arrows float in, wrapper handles scroll */
}

.gallery-wrapper {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    overflow-x: auto;
    overflow-y: hidden;
    padding-left: 5vw; 
    padding-right: 5vw;
    scrollbar-width: none; 
    -ms-overflow-style: none;
    cursor: grab;
    scroll-behavior: auto; /* Manual smooth scroll handling or CSS smooth */
    mask-image: linear-gradient(to right, transparent 0%, black 5%, black 95%, transparent 100%);
}

/* Nav Arrows Styles */
.nav-arrow {
    position: absolute;
    top: 50%;
    transform: translateY(-50%);
    background: transparent;
    border: none;
    color: white;
    font-size: 3rem;
    cursor: pointer;
    z-index: 20;
    padding: 20px;
    opacity: 0.5;
    transition: opacity 0.3s, transform 0.3s;
    font-family: 'Inter', sans-serif;
    font-weight: 300;
}

.nav-arrow:hover {
    opacity: 1;
    transform: translateY(-50%) scale(1.2);
    color: #D4AF37;
}

.nav-arrow.left { left: 1vw; }
.nav-arrow.right { right: 1vw; }

.gallery-wrapper::-webkit-scrollbar {
    display: none;
}

.gallery-track {
    display: flex;
    gap: 1.5rem;
    padding: 2rem 0; /* Vertical breathing room */
}

.album-card {
    position: relative;
    flex: 0 0 300px; /* Base width */
    height: 45vh; /* Responsive height */
    border-radius: 4px; /* Slight roundness */
    overflow: hidden;
    cursor: pointer;
    transition:  transform 0.4s cubic-bezier(0.25, 0.46, 0.45, 0.94), flex-basis 0.4s;
    background-color: #111;
    group: card;
}

.album-card:hover {
    transform: translateY(-10px);
    z-index: 5;
}

/* Image styling */
.image-container {
    width: 100%;
    height: 100%;
    position: relative;
}

.image-container img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    transition: transform 0.6s ease;
    filter: grayscale(100%) contrast(1.2);
}

.album-card:hover .image-container img {
    transform: scale(1.1);
    filter: grayscale(0%) contrast(1);
}

.overlay {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: linear-gradient(to top, rgba(0,0,0,0.9) 0%, rgba(0,0,0,0.2) 50%, transparent 100%);
    opacity: 0.8;
}

/* Text Info */
.card-info {
    position: absolute;
    bottom: 0;
    left: 0;
    width: 100%;
    padding: 1.5rem;
    transform: translateY(10px);
    transition: transform 0.4s ease;
}

.album-card:hover .card-info {
    transform: translateY(0);
}

.album-number {
    font-size: 0.8rem;
    color: #D4AF37; /* Gold accent */
    font-weight: bold;
    display: block;
    margin-bottom: 0.5rem;
    opacity: 0;
    transform: translateY(-10px);
    transition: all 0.3s ease 0.1s;
}

.album-card:hover .album-number {
    opacity: 1;
    transform: translateY(0);
}

.album-title {
    font-family: 'Anton', sans-serif;
    font-size: 2rem;
    margin: 0 0 0.5rem 0;
    text-transform: uppercase;
    line-height: 0.9;
    color: white;
    text-shadow: 0 2px 10px rgba(0,0,0,0.5);
}

.view-btn {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 0.8rem;
    font-weight: 600;
    color: white;
    opacity: 0;
    transition: opacity 0.3s ease;
}

.album-card:hover .view-btn {
    opacity: 1;
}

/* Footer / Micro interactions */
.lens-view-indicator {
    position: absolute;
    bottom: 2rem;
    right: 3rem;
    display: flex;
    align-items: center;
    gap: 1rem;
    opacity: 0.5;
    font-family: 'Courier New', monospace;
    font-size: 0.7rem;
    color: #D4AF37;
    pointer-events: none;
}

.lens-crosshair {
    width: 20px;
    height: 20px;
    border: 1px solid #D4AF37;
    border-radius: 50%;
    position: relative;
}

.lens-crosshair::after {
    content: '';
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    width: 2px;
    height: 2px;
    background: #D4AF37;
}

/* Mobile Responsive */
@media (max-width: 768px) {
    .portfolio-header {
        padding: 1.5rem;
    }
    
    .gallery-track {
        flex-direction: column; /* Stack vertically on mobile? Or Horizontal? Horizontal is better for gallery feel */
        flex-direction: row; 
    }
    
    .album-card {
        flex: 0 0 75vw; /* Large width on mobile */
        height: 55vh;
        scroll-snap-align: center;
    }
    
    .gallery-wrapper {
        scroll-snap-type: x mandatory;
        padding-left: 12.5vw; /* Center the first item (100 - 75) / 2 */
        padding-right: 12.5vw;
        mask-image: none; /* Remove fade mask on mobile specifically if performance needs */
    }
    
    .image-container img {
        filter: grayscale(0%); /* Always color on mobile */
    }
    
    .album-number, .view-btn {
        opacity: 1; /* Always visible info on mobile */
        transform: none;
    }
    
    .card-info {
        transform: none;
    }
}
</style>
