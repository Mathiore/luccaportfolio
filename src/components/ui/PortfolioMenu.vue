<script setup>
import { ref, computed, onMounted, onBeforeUnmount } from 'vue';
import { albums } from '../../config/albums.js';

// State
const viewState = ref('lens'); // 'lens' | 'gallery'
const isMobile = ref(false);

// Formatting Albums for Menu
const menuItems = computed(() => albums.map((album, index) => ({
    id: album.id,
    title: album.title, 
    count: album.images ? album.images.length : 0,
    index
})));

const currentAlbumIndex = ref(0);
const currentImageIndex = ref(0);

// Ensure valid indices
const currentAlbum = computed(() => albums[currentAlbumIndex.value] || albums[0]);
const currentImages = computed(() => currentAlbum.value.images || []);
const currentImage = computed(() => {
    if (viewState.value === 'lens') return currentAlbum.value.cover;
    return currentImages.value[currentImageIndex.value] || currentAlbum.value.cover;
});

// Lens Effect State - USING PIXELS NOW
const lensPos = ref({ x: 0, y: 0 });
const targetPos = ref({ x: 0, y: 0 });
let animationFrame = null;

const onMouseMove = (e) => {
    if (viewState.value !== 'lens' || isMobile.value) return;
    targetPos.value = { x: e.clientX, y: e.clientY };
};

const updateLens = () => {
    if (viewState.value === 'lens') {
        if (isMobile.value) {
            // Automated wandering movement for mobile
            const time = Date.now() * 0.0008; // speed
            const cx = window.innerWidth / 2;
            const cy = window.innerHeight / 2;
            const radiusX = window.innerWidth * 0.3; 
            const radiusY = window.innerHeight * 0.2;
            
            // Lissajous-ish path
            const tx = cx + Math.sin(time) * radiusX;
            const ty = cy + Math.cos(time * 1.3) * radiusY;
            
            targetPos.value = { x: tx, y: ty };

            lensPos.value.x += (targetPos.value.x - lensPos.value.x) * 0.05;
            lensPos.value.y += (targetPos.value.y - lensPos.value.y) * 0.05;
        } else {
            // Mouse follower
            lensPos.value.x += (targetPos.value.x - lensPos.value.x) * 0.1;
            lensPos.value.y += (targetPos.value.y - lensPos.value.y) * 0.1;
        }
    }
    animationFrame = requestAnimationFrame(updateLens);
};

// Compute styles based on pixel position
const lensStyle = computed(() => {
    // Always return variables for dynamic positioning
    return {
        '--lens-x': `${lensPos.value.x}px`,
        '--lens-y': `${lensPos.value.y}px`
    };
});

// Navigation
const enterGallery = () => {
    viewState.value = 'gallery';
};

const selectAlbum = (index) => {
    currentAlbumIndex.value = index;
    currentImageIndex.value = 0;
};

const selectImage = (index) => {
    currentImageIndex.value = index;
};

// Lifecycle
onMounted(() => {
    isMobile.value = window.innerWidth < 768;
    
    // Set initial album to Gallardo as requested
    const gallardoIndex = albums.findIndex(a => a.id === 'gallardo');
    if (gallardoIndex !== -1) {
        currentAlbumIndex.value = gallardoIndex;
    }

    // Initialize center strictly
    const cx = window.innerWidth / 2;
    const cy = window.innerHeight / 2;
    lensPos.value = { x: cx, y: cy };
    targetPos.value = { x: cx, y: cy };

    window.addEventListener('resize', () => {
        isMobile.value = window.innerWidth < 768;
    });
    
    updateLens();
});

onBeforeUnmount(() => {
    if (animationFrame) cancelAnimationFrame(animationFrame);
});
</script>

<template>
    <div class="portfolio-mega-container" :class="{ 'gallery-mode': viewState === 'gallery' }">
        
        <!-- LENS VIEW STATE -->
        <div 
            class="lens-view-layer" 
            :class="{ active: viewState === 'lens' }"
            @mousemove="onMouseMove" 
            @click="enterGallery"
        >
            <!-- 1. Blurred Background Image -->
            <div class="bg-blurred" :style="{ backgroundImage: `url(${currentImage})` }"></div>
            
            <!-- Vignette & Darkening -->
            <div class="vignette-overlay"></div>
            
            <!-- 2. Sharp Foreground Image (Clipped) -->
            <!-- We use mask-position with calculated pixel offsets for perfect alignment -->
            <div class="fg-sharp" :style="{ ...lensStyle, backgroundImage: `url(${currentImage})` }"></div>
            
            <!-- Lens UI Border/Reticle -->
            <div class="lens-reticle" :style="lensStyle">
                <div class="corner tl"></div>
                <div class="corner tr"></div>
                <div class="corner bl"></div>
                <div class="corner br"></div>
            </div>

            <div class="prompt-text">
                <span>CLICK TO ENTER</span>
            </div>

            <!-- GRID OVERLAY -->
            <div class="grid-overlay">
                <!-- Corners -->
                <div class="corner-mark tl"></div>
                <div class="corner-mark tr"></div>
                <div class="corner-mark bl"></div>
                <div class="corner-mark br"></div>

                <!-- Grid Points -->
                <div class="grid-points">
                    <div v-for="n in 24" :key="n" class="point"></div>
                </div>

                <!-- Bottom Numbering -->
                <div class="grid-numbers">
                    <span class="num">02</span>
                    <div class="sep"></div>
                    <span class="num">03</span>
                </div>
            </div>

            <!-- ZOOM RULER (Decoration) -->
            <div class="zoom-ruler">
                <div class="ruler-track">
                    <div v-for="n in 61" :key="n" class="tick" :class="{ major: (n-1) % 10 === 0 }">
                        <span v-if="(n-1) === 10" class="tick-label">01</span>
                        <span v-if="(n-1) === 30" class="tick-label">02</span>
                        <span v-if="(n-1) === 50" class="tick-label">03</span>
                        
                        <!-- Red Dot indicator near 02 -->
                        <div v-if="(n-1) === 30" class="red-dot-indicator"></div>
                    </div>
                </div>
            </div>
        </div>

        <!-- GALLERY VIEW STATE -->
        <div class="gallery-view-layer" :class="{ active: viewState === 'gallery' }">
            
            <!-- Col 1: Clean Brand Menu -->
            <nav class="brand-menu">
                <div class="brand-list">
                    <div 
                        v-for="(item, idx) in menuItems" 
                        :key="item.id"
                        class="brand-item"
                        :class="{ active: currentAlbumIndex === idx }"
                        @click="selectAlbum(idx)"
                    >
                        <span class="brand-name">{{ item.title }}</span>
                        <span class="brand-count">{{ item.count }}</span>
                    </div>
                </div>
            </nav>

            <!-- Col 2: Main Image -->
            <main class="main-display">
                <transition name="fade-fast" mode="out-in">
                    <div :key="currentImage" class="main-image-wrapper">
                        <img :src="currentImage" alt="Gallery Selection" class="main-img" />
                    </div>
                </transition>
            </main>

            <!-- Col 3: Thumbnails -->
            <aside class="thumbnails-sidebar">
                <div class="thumbs-scroll">
                    <div 
                        v-for="(img, idx) in currentImages" 
                        :key="idx"
                        class="thumb-item"
                        :class="{ active: currentImageIndex === idx }"
                        @click="selectImage(idx)"
                    >
                        <img :src="img" loading="lazy" />
                    </div>
                </div>
            </aside>

        </div>

    </div>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600&display=swap');

/* GLOBAL */
.portfolio-mega-container {
    position: absolute;
    top: 0; left: 0;
    width: 100%; height: 100vh;
    background: #0a0a0a;
    color: #e0e0e0;
    font-family: 'Inter', sans-serif;
    overflow: hidden;
    z-index: 100;
}

/* LENS VIEW */
.lens-view-layer {
    position: absolute;
    width: 100%; height: 100%;
    z-index: 20;
    opacity: 0;
    pointer-events: none;
    transform: scale(1.1);
    transition: opacity 0.8s ease, transform 1s ease;
}

.lens-view-layer.active {
    opacity: 1;
    transform: scale(1);
    pointer-events: auto;
    cursor: none;
}

.bg-blurred {
    position: absolute;
    top: -5%; left: -5%;
    width: 110%; height: 110%;
    background-size: cover;
    background-position: center;
    filter: blur(4px) brightness(0.4); 
    transition: background-image 0.5s ease;
}

.vignette-overlay {
    position: absolute;
    top: 0; left: 0;
    width: 100%; height: 100%;
    background: radial-gradient(circle, transparent 40%, #0a0a0a 100%);
    pointer-events: none;
}

.fg-sharp {
    position: absolute;
    top: 0; left: 0;
    width: 100%; height: 100%;
    background-size: cover;
    background-position: center;
    mask-image: linear-gradient(black, black);
    mask-size: 300px 300px;
    mask-repeat: no-repeat;
    mask-position: calc(var(--lens-x) - 150px) calc(var(--lens-y) - 150px);
    -webkit-mask-image: linear-gradient(black, black);
    -webkit-mask-size: 300px 300px;
    -webkit-mask-repeat: no-repeat;
    -webkit-mask-position: calc(var(--lens-x) - 150px) calc(var(--lens-y) - 150px);
    border: 1px solid rgba(255,255,255,0.05); 
    filter: contrast(1.1);
}

@media (max-width: 768px) {
    .fg-sharp {
        mask-position: center center;
        -webkit-mask-position: center center;
        mask-size: 250px 250px;
        -webkit-mask-size: 250px 250px;
    }
}

.lens-reticle {
    position: absolute;
    width: 300px; height: 300px;
    left: var(--lens-x);
    top: var(--lens-y);
    transform: translate(-50%, -50%);
    pointer-events: none;
}

@media (max-width: 768px) {
    .lens-reticle {
        left: 50%; top: 50%;
        width: 250px; height: 250px;
        transform: translate(-50%, -50%);
    }
}

.lens-reticle .corner {
    position: absolute;
    width: 8px; height: 8px;
    border-color: rgba(255,255,255,0.8);
    border-style: solid;
    border-width: 0;
}
.lens-reticle .tl { top: 0px; left: 0px; border-top-width: 1px; border-left-width: 1px; }
.lens-reticle .tr { top: 0px; right: 0px; border-top-width: 1px; border-right-width: 1px; }
.lens-reticle .bl { bottom: 0px; left: 0px; border-bottom-width: 1px; border-left-width: 1px; }
.lens-reticle .br { bottom: 0px; right: 0px; border-bottom-width: 1px; border-right-width: 1px; }

.prompt-text {
    position: absolute;
    bottom: 120px; /* Moved up for ruler */
    width: 100%;
    text-align: center;
    font-size: 0.75rem;
    letter-spacing: 0.2em;
    opacity: 0.5;
    text-transform: uppercase;
}

/* ZOOM RULER STYLE */
.zoom-ruler {
    position: absolute;
    bottom: 30px;
    left: 50%;
    transform: translateX(-50%);
    width: 80%;
    max-width: 800px;
    height: 50px;
    pointer-events: none;
    display: flex;
    justify-content: center;
    align-items: flex-end;
}

.ruler-track {
    display: flex;
    gap: 12px;
    align-items: flex-end;
}

.tick {
    width: 1px;
    height: 8px;
    background: rgba(255,255,255,0.3);
    position: relative;
}

.tick.major {
    height: 16px;
    background: rgba(255,255,255,0.6);
}

.tick-label {
    position: absolute;
    top: -20px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 0.7rem;
    color: rgba(255,255,255,0.9);
    font-weight: 600;
}

.red-dot-indicator {
    position: absolute;
    top: -30px; /* Above label roughly */
    left: 50%;
    transform: translateX(-50%);
    width: 4px;
    height: 4px;
    background-color: #ff0000;
    border-radius: 50%;
    box-shadow: 0 0 5px #ff0000;
}


/* GRID OVERLAY STYLE */
.grid-overlay {
    position: absolute;
    top: 0; left: 0;
    width: 100%; height: 100%;
    pointer-events: none;
    z-index: 15;
    padding: 30px;
    box-sizing: border-box;
}

.grid-points {
    display: grid;
    grid-template-columns: repeat(6, 1fr);
    grid-template-rows: repeat(4, 1fr);
    width: 100%; height: 100%;
    place-items: center;
}

.point {
    width: 3px; height: 3px;
    background: #fff;
    opacity: 0.8;
    box-shadow: 0 0 2px rgba(255,255,255,0.5);
}

.corner-mark {
    position: absolute;
    width: 30px; height: 30px;
    border-color: rgba(255,255,255,0.8);
    border-style: solid;
    border-width: 0;
}
.corner-mark.tl { top: 30px; left: 30px; border-top-width: 2px; border-left-width: 2px; }
.corner-mark.tr { top: 30px; right: 30px; border-top-width: 2px; border-right-width: 2px; }
.corner-mark.bl { bottom: 30px; left: 30px; border-bottom-width: 2px; border-left-width: 2px; }
.corner-mark.br { bottom: 30px; right: 30px; border-bottom-width: 2px; border-right-width: 2px; }

.grid-numbers {
    position: absolute;
    bottom: 40px;
    left: 80px;
    display: flex;
    align-items: center;
    gap: 15px;
    font-family: 'Inter', sans-serif;
    font-size: 0.75rem;
    color: rgba(255,255,255,0.9);
    font-weight: 600;
    letter-spacing: 0.1em;
}
.grid-numbers .sep {
    width: 30px; height: 1px; background: rgba(255,255,255,0.4);
}


/* GALLERY VIEW LAYER */
.gallery-view-layer {
    position: absolute;
    top: 0; left: 0;
    width: 100%; height: 100%;
    background: #0a0a0a;
    z-index: 10;
    display: grid;
    grid-template-columns: 260px 1fr 180px;
    opacity: 0;
    transform: scale(0.98);
    transition: opacity 0.8s ease, transform 0.8s cubic-bezier(0.2, 0.8, 0.2, 1);
    pointer-events: none;
}

.gallery-view-layer.active {
    opacity: 1;
    transform: scale(1);
    z-index: 30;
    pointer-events: auto;
}

/* Brand Menu */
.brand-menu {
    padding: 3rem 0 3rem 3rem; 
    display: flex;
    flex-direction: column;
    justify-content: center;
}

.brand-list {
    display: flex;
    flex-direction: column;
    gap: 1.2rem;
}

.brand-item {
    font-family: 'Inter', sans-serif;
    font-size: 1.1rem;
    font-weight: 500;
    color: #444; 
    cursor: pointer;
    transition: color 0.3s ease, transform 0.3s ease;
    display: flex;
    align-items: center;
    gap: 0.8rem;
    position: relative;
    width: fit-content;
}

.brand-name {
    position: relative;
    z-index: 1;
}

.brand-count {
    font-size: 0.7rem;
    padding: 2px 5px;
    background: #1a1a1a;
    border-radius: 4px;
    color: #444;
    transition: all 0.3s ease;
    font-weight: 400;
}

.brand-item:hover {
    color: #888;
}
.brand-item:hover .brand-count {
    color: #888;
    background: #2a2a2a;
}

.brand-item.active {
    color: #fff; 
    font-weight: 700;
    transform: translateX(5px);
}
.brand-item.active .brand-count {
    color: #000;
    background: #fff;
    font-weight: 700;
}

/* Main Display */
.main-display {
    padding: 2rem;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
}

.main-image-wrapper {
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
}

.main-img {
    max-width: 100%;
    max-height: 100%;
    object-fit: contain;
    box-shadow: 0 10px 40px rgba(0,0,0,0.5); 
}

/* Sidebar Thumbs */
.thumbnails-sidebar {
    padding: 2rem 1rem 2rem 0;
    overflow-y: auto;
    scrollbar-width: none;
    display: flex;
    flex-direction: column;
    justify-content: center; 
}

.thumbs-scroll {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
}

.thumb-item {
    width: 100%;
    aspect-ratio: 16/9;
    border-radius: 2px;
    overflow: hidden;
    opacity: 0.3;
    filter: grayscale(100%);
    transition: all 0.3s ease;
    cursor: pointer;
    border: 1px solid transparent;
}

.thumb-item img {
    width: 100%; height: 100%; object-fit: cover;
}

.thumb-item:hover {
    opacity: 0.6;
}

.thumb-item.active {
    opacity: 1;
    filter: grayscale(0%);
    border-color: rgba(255,255,255,0.4);
}

/* TRANSITIONS */
.fade-fast-enter-active,
.fade-fast-leave-active {
    transition: opacity 0.3s ease;
}
.fade-fast-enter-from, .fade-fast-leave-to { opacity: 0; }

/* RESPONSIVE */
@media (max-width: 1024px) {
    .gallery-view-layer {
        grid-template-columns: 200px 1fr 140px;
    }
}

@media (max-width: 768px) {
    .gallery-view-layer {
        grid-template-columns: 1fr;
        grid-template-rows: auto 1fr auto;
    }

    .brand-menu {
        padding: 1rem 0;
        grid-row: 1;
        background: #0a0a0a;
        border-bottom: 1px solid #1a1a1a;
        width: 100%;
        overflow-x: auto;
    }

    .brand-list {
        flex-direction: row;
        padding: 0 1.5rem;
        width: max-content;
        gap: 2rem;
    }

    .brand-item {
        font-size: 0.9rem;
        transform: none !important; 
    }

    .thumbnails-sidebar {
        grid-row: 3;
        padding: 1rem 0;
        width: 100%;
        border-top: 1px solid #1a1a1a;
        justify-content: flex-start;
    }

    .thumbs-scroll {
        flex-direction: row;
        padding: 0 1rem;
        overflow-x: auto;
        width: 100%;
    }

    .thumb-item {
        width: 80px;
        height: 50px;
        flex-shrink: 0;
    }
    
    .fg-sharp {
        /* Mobile size 250px */
        mask-size: 250px 250px;
        -webkit-mask-size: 250px 250px;
        
        /* Dynamic position with smaller offset (125px) */
        mask-position: calc(var(--lens-x) - 125px) calc(var(--lens-y) - 125px);
        -webkit-mask-position: calc(var(--lens-x) - 125px) calc(var(--lens-y) - 125px);
    }
    
    .lens-reticle {
        /* Use variables for position, override any forced center if present */
        left: var(--lens-x); 
        top: var(--lens-y);
        
        width: 250px; height: 250px;
        /* transform translate -50% -50% is inherited from base and correct */
    }
}
</style>
