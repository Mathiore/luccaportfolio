<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';
import { RoomEnvironment } from 'three/examples/jsm/environments/RoomEnvironment.js';

import { cars } from '../config/cars.js';
import { albums } from '../config/albums.js';
import LoadingScreen from './ui/LoadingScreen.vue';
import AlbumModal from './ui/AlbumModal.vue';
import CameraOverlay from './ui/CameraOverlay.vue';
import BackgroundLayer from './ui/BackgroundLayer.vue';
import PortfolioMenu from './ui/PortfolioMenu.vue';

const canvasRef = ref(null);
const containerRef = ref(null);
const stickyWrapperRef = ref(null);
let animationId = null;
let scene, camera, renderer, model;
let targetCameraPos = new THREE.Vector3(0, 0, 0.8); // Final position (Close Up)
let startCameraPos = new THREE.Vector3(0, 15, 5.0); // Start position (High and aligned for swoop)
let animationProgress = 0;
let isAnimating = false;

// Loading State
const isLoading = ref(true);
const loadingProgress = ref(0);
const currentScrollP = ref(0); // For RAF sync

// Mobile detection
const isMobile = window.innerWidth < 768;

// Album State
const showAlbum = ref(false);
const selectedAlbum = ref(null);

const isMenuVisible = ref(false); // Controls 2D Menu visibility

// Grid of Cars Removed
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();

// Touch State
let touchStartData = { x: 0, y: 0, time: 0 };

// Initialize Three.js
const init = () => {
  // Use window dimensions because the canvas is in a sticky 100vh container
  // accessing containerRef (which is 400vh) caused the renderer to be massive and distorted.
  const width = window.innerWidth;
  const height = window.innerHeight;

  // Scene
  scene = new THREE.Scene();
  // scene.background = new THREE.Color(0x000000); //  Removed to allow CSS background visibility


  // Camera
  camera = new THREE.PerspectiveCamera(45, width / height, 0.01, 1000);
  camera.position.copy(startCameraPos);
  camera.lookAt(0, 0, 0);

  // Renderer
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true,
    alpha: true
  });
  renderer.setSize(width, height);
  renderer.setPixelRatio(isMobile ? 1 : Math.min(window.devicePixelRatio, 2));
  renderer.shadowMap.enabled = !isMobile;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.outputColorSpace = THREE.SRGBColorSpace;


  // Environment
  const pmremGenerator = new THREE.PMREMGenerator(renderer);
  scene.environment = pmremGenerator.fromScene(new RoomEnvironment()).texture;

  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  scene.add(ambientLight);

  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(5, 10, 7);
  dirLight.castShadow = true;
  scene.add(dirLight);

  const spotLight = new THREE.SpotLight(0xff0000, 10);
  spotLight.position.set(-5, 5, 0);
  spotLight.lookAt(0, 0, 0);
  scene.add(spotLight);

  // Load Model
  const manager = new THREE.LoadingManager();
  
  manager.onLoad = () => {
      // All models loaded
      console.log('All resources loaded');
      
      // PRE-COMPILE / WARM-UP RENDER
      // We must ensure the objects are actually IN THE FRUSTUM for the renderer to process them.
      if (renderer && scene && camera) {
          // Save Camera State
          const originalPos = camera.position.clone();
          const originalQuat = camera.quaternion.clone();

          // Move camera to a position where it definitely sees the grid (Z = -8 to -10)
          camera.position.set(0, 5, 10);
          camera.lookAt(0, 0, -10);
          camera.updateMatrixWorld();
          
          // Force render to upload all geometry/textures to GPU
          renderer.render(scene, camera);
          
          // Restore Camera
          camera.position.copy(originalPos);
          camera.quaternion.copy(originalQuat);
          camera.updateMatrixWorld();
      }

      // Minimal delay just to let UI update
      setTimeout(() => {
          isLoading.value = false;
      }, 100);
  };
  
  manager.onProgress = (url, itemsLoaded, itemsTotal) => {
      loadingProgress.value = Math.floor((itemsLoaded / itemsTotal) * 100);
  };
  
  manager.onError = (url) => {
      console.error('There was an error loading ' + url);
  };

  const loader = new GLTFLoader(manager);
  const dracoLoader = new DRACOLoader();
  dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.7/');
  loader.setDRACOLoader(dracoLoader);

  loader.load('/models/Cannon.glb', (gltf) => {
    model = gltf.scene;
    // Center model
    const box = new THREE.Box3().setFromObject(model);
    const center = box.getCenter(new THREE.Vector3());
    model.position.sub(center); // Center at 0,0,0
    
    // Manual visual offset to lift the camera up
    model.position.y += 0.06;
    
    // Scale fitting
    // Adjust scale if needed based on model size
    const size = box.getSize(new THREE.Vector3());
    const maxDim = Math.max(size.x, size.y, size.z);
    
    const targetSize = 0.13; 
    let scale = 0.4;
    if (maxDim > 0) {
        scale = targetSize / maxDim;
    }
    model.scale.set(scale, scale, scale);

    model.traverse((child) => {
      if (child.isMesh) {
        child.castShadow = true;
        child.receiveShadow = true;
        
        if (child.material) {
            child.material.side = THREE.DoubleSide;
            child.material.format = THREE.RGBAFormat;
            child.material.depthWrite = true;
            
            child.material.color.setHex(0x333333); 
            child.material.roughness = 0.5;
            child.material.metalness = 0.7;
        }
      }
    });

    // Set initial rotation to face front (180 deg)
    model.rotation.y = Math.PI;

    scene.add(model);
    
    // Start falling animation after load
    startAnimation();

  }, undefined, (error) => {
    console.error('An error happened loading the Cannon:', error);
  });
  
  // Grid loading removed

  // Handle Resize
  window.addEventListener('resize', onResize);
  
  // Handle Scroll
    window.addEventListener('scroll', onScroll);
    window.addEventListener('mousemove', onPointerMove);
    window.addEventListener('click', onCanvasClick);
    window.addEventListener('touchstart', onTouchStart, { passive: false }); // Passive false if we want to preventDefault, but we likely won't
    window.addEventListener('touchend', onTouchEnd);
};

// Store grid objects (Removed)




const startAnimation = () => {
    isAnimating = true;
    animationProgress = 0;
};

const onScroll = () => {
    if (!containerRef.value) return;
    
    // Calculate raw scroll progress
    const rect = containerRef.value.getBoundingClientRect();
    const viewportHeight = window.innerHeight;
    
    let scrollP = -rect.top / (rect.height - viewportHeight);
    scrollP = Math.max(0, Math.min(1, scrollP));
    
    // Store for RAF
    currentScrollP.value = scrollP;
};

const animate = () => {
  animationId = requestAnimationFrame(animate);

  if (isAnimating) {
      animationProgress += 0.01; // Speed of fall
      if (animationProgress >= 1) {
          animationProgress = 1;
          isAnimating = false; // Stop strictly animating position, allow scroll to take over
      }

      // Easing function for smooth fall (easeOutCubic)
      const t = 1 - Math.pow(1 - animationProgress, 3);
      
      // Interpolate position
      camera.position.lerpVectors(startCameraPos, targetCameraPos, t);
      camera.lookAt(0, 0, 0); // Keep looking at target
      
      // Ensure model is facing correctly during animation
      if (model) model.rotation.y = Math.PI;
  } else {
      // Scroll controlled animation
      const scrollP = currentScrollP.value;
      const startZ = 0.8;
      const endZ = 0.15;
      
      const currentZ = startZ - (scrollP * (startZ - endZ));
      if (camera) camera.position.z = currentZ;
      
      // Scene updates synced with RAF
      if (model) {
          // Offsetting rotation by PI (180 deg) because the model is naturally backwards
          model.rotation.y = Math.PI + (scrollP * Math.PI); 
          
          const switchThreshold = 0.90;
          model.visible = scrollP < switchThreshold;
          
          isMenuVisible.value = scrollP >= switchThreshold;
      }
  }

  // Slight rotation of model for dynamism REMOVED
  
  // Hover Logic Removed
  
  // Rotate Grid Models Removed




  renderer.render(scene, camera);
};

const onPointerMove = (event) => {
    pointer.x = (event.clientX / window.innerWidth) * 2 - 1;
    pointer.y = - (event.clientY / window.innerHeight) * 2 + 1;
};

const checkIntersection = (clientX, clientY) => {
    // 3D Intersection logic removed.
    // PortfolioMenu handles clicks now.
};

const onCanvasClick = (event) => {
    // Determine if it was a mouse click (detail is 1 usually, or pointerType)
    // We let touchEnd handle mobile taps to avoid 300ms delay and ghost clicks issues
    // However, sometimes firing both is fine if we check state.
    checkIntersection(event.clientX, event.clientY);
};

const onTouchStart = (e) => {
    if (e.touches.length === 1) {
        touchStartData.x = e.touches[0].clientX;
        touchStartData.y = e.touches[0].clientY;
        touchStartData.time = Date.now();
    }
};

const onTouchEnd = (e) => {
    if (e.changedTouches.length > 0) {
        const t = e.changedTouches[0];
        const dx = t.clientX - touchStartData.x;
        const dy = t.clientY - touchStartData.y;
        const dt = Date.now() - touchStartData.time;
        
        // Tap threshold: < 10px movement, < 300ms duration
        if (Math.abs(dx) < 10 && Math.abs(dy) < 10 && dt < 300) {
             checkIntersection(t.clientX, t.clientY);
        }
    }
};
const handleFolderSelection = (id) => {
    if (id === 'albums') {
        // Open first album or a dashboard - using first album as placeholder
        selectedAlbum.value = albums[0];
        showAlbum.value = true; // Still use this modal for now
    }
    // Handle others (about, contact, license)
};

const onResize = () => {
  if (!containerRef.value) return;
  // Canvas is sticky 100vh, so width/height is viewport usually
  const width = window.innerWidth; 
  const height = window.innerHeight;
  
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
  
  // updateGridLayout(); // Function removed
};

onMounted(() => {
  init();
  animate();
  
  const observer = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
          if (entry.isIntersecting) {
              startAnimation();
          }
      });
  });
  
  if (containerRef.value) observer.observe(containerRef.value);
});

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', onResize);
  window.removeEventListener('mousemove', onPointerMove);
  window.removeEventListener('resize', onResize);
  window.removeEventListener('mousemove', onPointerMove);
  window.removeEventListener('click', onCanvasClick);
  window.removeEventListener('touchstart', onTouchStart);
  window.removeEventListener('touchend', onTouchEnd);
  if (renderer) renderer.dispose();
});

</script>

<template>
  <section ref="containerRef" class="cannon-section">
    <!-- Loading Overlay Logic-->
    <LoadingScreen :is-loading="isLoading" :progress="loadingProgress" />
    
    <!-- Album Modal -->
    <AlbumModal v-if="showAlbum" :album="selectedAlbum" @close="showAlbum = false" />
  
    <div ref="stickyWrapperRef" class="sticky-wrapper">
        <BackgroundLayer :scroll-progress="currentScrollP" :is-loading="isLoading" />
        <canvas ref="canvasRef" class="webgl-canvas"></canvas>
        <PortfolioMenu v-if="isMenuVisible" @select-folder="handleFolderSelection" />
        <CameraOverlay :scroll-progress="currentScrollP" />
    </div>
  </section>
</template>

<style scoped>
.cannon-section {
  position: relative;
  width: 100%;
  height: 300vh; /* Reduced height for faster interaction */
  background-color: #000;
}

.sticky-wrapper {
    position: sticky;
    top: 0;
    width: 100%;
    height: 100vh;
    overflow: hidden;
    /* Default background black to match section */
    background-color: #000000; 
}

.webgl-canvas {
  position: relative;
  width: 100%;
  height: 100%;
  display: block;
  z-index: 10;
}
</style>
