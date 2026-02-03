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

const canvasRef = ref(null);
const containerRef = ref(null);
const overlayRef = ref(null);
const backgroundRef = ref(null);
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

// Grid of Cars (using imported config)
// gridModels array tracks the scene objects
const gridModels = [];
const gridGroup = new THREE.Group();
gridGroup.visible = false; // Hidden initially
const interactionPlanes = []; // Planes for hover detection
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
          const originalGridVisible = gridGroup.visible;
          gridGroup.visible = true;
          
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
          
          gridGroup.visible = originalGridVisible;
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
    
    // Scale fitting
    // Adjust scale if needed based on model size
    // Cannon might be huge or small, let's guess a reasonable scale or normalize it
    const size = box.getSize(new THREE.Vector3());
    const maxDim = Math.max(size.x, size.y, size.z);
    if (maxDim > 5) {
        const scale = 3 / maxDim;
        model.scale.set(scale, scale, scale);
    }

    model.traverse((child) => {
      if (child.isMesh) {
        child.castShadow = true;
        child.receiveShadow = true;
        
        // Fix transparency issues
        if (child.material) {
            child.material.side = THREE.DoubleSide;
            child.material.format = THREE.RGBAFormat;
            child.material.depthWrite = true;
        }
      }
    });

    scene.add(model);
    
    // Start falling animation after load
    startAnimation();

  }, undefined, (error) => {
    console.error('An error happened loading the Cannon:', error);
  });
  
  // Load Grid Models
  loadGrid(loader);

  // Handle Resize
  window.addEventListener('resize', onResize);
  
  // Handle Scroll
    window.addEventListener('scroll', onScroll);
    window.addEventListener('mousemove', onPointerMove);
    window.addEventListener('click', onCanvasClick);
    window.addEventListener('touchstart', onTouchStart, { passive: false }); // Passive false if we want to preventDefault, but we likely won't
    window.addEventListener('touchend', onTouchEnd);
};

// Store grid objects for responsive updates
let gridPivots = [];

const loadGrid = (loader) => {
    scene.add(gridGroup);
    
    // Load all models first
    cars.forEach((carDef, index) => {
        loader.load(carDef.path, (gltf) => {
             const car = gltf.scene;
             const box = new THREE.Box3().setFromObject(car);
             const size = box.getSize(new THREE.Vector3());
             const maxDim = Math.max(size.x, size.y, size.z);
             const scaleFactor = 1.5 / maxDim;
             car.scale.set(scaleFactor, scaleFactor, scaleFactor);

             const center = box.getCenter(new THREE.Vector3());
             car.position.sub(center.multiplyScalar(scaleFactor));
             
             const pivot = new THREE.Group();
             pivot.add(car);
             
             // Center car geometry in pivot
             const localBox = new THREE.Box3().setFromObject(car);
             const localCenter = localBox.getCenter(new THREE.Vector3());
             car.position.copy(localCenter).multiplyScalar(-1);
             
             // Disable frustum culling to prevent pop-in
             car.traverse((child) => {
                 if (child.isMesh) {
                     child.frustumCulled = false;
                 }
             });
             
             
             // Add 2D Square Frame
             const frameSize = 1.5; // Reduced Frame size to prevent overlap
             // Create a square shape path
             const points = [];
             points.push(new THREE.Vector3(-frameSize/2, -frameSize/2, 0));
             points.push(new THREE.Vector3(frameSize/2, -frameSize/2, 0));
             points.push(new THREE.Vector3(frameSize/2, frameSize/2, 0));
             points.push(new THREE.Vector3(-frameSize/2, frameSize/2, 0));
             
             const frameGeo = new THREE.BufferGeometry().setFromPoints(points);
             // Use LineLoop to close the square
             const frame = new THREE.LineLoop(frameGeo, new THREE.LineBasicMaterial({ color: 0xffffff, opacity: 0.5, transparent: true }));
             
             // Position frame roughly behind/around car but facing camera
             // Since cars are 3D, a 2D frame might clip if goes through.
             // Let's ensure it's slightly behind or centered.
             // If we want it "flat" relative to view, it might look odd if we orbit. 
             // But here view is fixed.
             // Let's simple enclose.
             
             frame.scale.set(1.5, 1, 1); // Aspect ratio
             pivot.add(frame);

             // Interaction Plane (Hover Background)
             const planeGeo = new THREE.PlaneGeometry(frameSize, frameSize);
             const planeMat = new THREE.MeshBasicMaterial({ 
                 color: 0xffffff, 
                 side: THREE.DoubleSide, 
                 transparent: true, 
                 opacity: 0 // Start invisible
             });
             const plane = new THREE.Mesh(planeGeo, planeMat);
             plane.position.z = -0.01; // Match frame depth roughly
             plane.scale.set(1.5, 1, 1); // Match frame aspect ratio
             plane.userData = { isHoverPlane: true, parentPivot: pivot };
             pivot.add(plane);
             interactionPlanes.push(plane);

             
             // Add Text Label
             const sprite = createTextSprite(carDef.name);
             sprite.position.set(0, -0.75, 1); // Bottom of frame (adjusted for smaller frame)
             pivot.add(sprite);

             pivot.userData = { index: index }; // Store index for grid calc
             gridModels.push(pivot);
             gridGroup.add(pivot);
             gridPivots.push(pivot);
             
             // Initial positioning
             updateGridLayout();
        });
    });
};

const updateGridLayout = () => {
    if (!containerRef.value) return;
    const isMobile = window.innerWidth < 768;
    
    // Adjust layout based on screen size
    // Mobile: 2 columns, 3 rows? Or 1 column?
    // User said "2 models appearing", implying cut off.
    // Let's do 2 columns on mobile, 3 on desktop.
    
    const cols = isMobile ? 2 : 3;
    const spacingX = isMobile ? 2.2 : 4.0; 
    const spacingY = isMobile ? 2.2 : 2.5;
    
    // On mobile, maybe push further back or scale down?
    // Let's scale the whole group down slightly on mobile or push Z back.
    const zPos = isMobile ? -10 : -8; // Closer on mobile (-10 vs -14) makes them larger inside the FOV 
    
    gridPivots.forEach((pivot) => {
        const index = pivot.userData.index;
        const col = index % cols;
        const row = Math.floor(index / cols);
        
        // Recalculate rows count for centering
        const totalRows = Math.ceil(cars.length / cols);
        
        const x = (col - (cols - 1) / 2) * spacingX;
        const y = (row - (totalRows - 1) / 2) * spacingY; // Vertical centering
        
        pivot.position.set(x, -y, zPos); // Invert Y for correct top-down order usually
    });
};

const createTextSprite = (text) => {
    const canvas = document.createElement('canvas');
    const size = 512;
    canvas.width = size;
    canvas.height = size / 4;
    const context = canvas.getContext('2d');
    
    context.fillStyle = 'rgba(0,0,0,0)'; // Transparent
    context.fillRect(0, 0, canvas.width, canvas.height);
    
    context.font = 'bold 40px Arial'; // Smaller font
    context.textAlign = 'center';
    context.textBaseline = 'middle';
    context.fillStyle = '#D4AF37'; // Gold
    context.fillText(text, size / 2, size / 8);
    
    const texture = new THREE.CanvasTexture(canvas);
    const material = new THREE.SpriteMaterial({ map: texture });
    const sprite = new THREE.Sprite(material);
    sprite.scale.set(2, 0.5, 1);
    
    return sprite;
};


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
    
    // Update DOM elements (overlay, etc) in scroll handler is fine as they are separate layer, 
    // but ideally could also be in RAF if position dependent.
    // Opacity changes are cheap.
    if(overlayRef.value) {
         overlayRef.value.style.opacity = scrollP > 0.8 ? (scrollP - 0.8) * 5 : 0;
    }
    
    // Update background color (White to Black) - DOM style update
    if (backgroundRef.value) {
         const val = Math.floor(255 * (1 - scrollP));
         backgroundRef.value.style.backgroundColor = `rgb(${val}, ${val}, ${val})`;
         
         // Fade out text at the end
         const textEl = backgroundRef.value.querySelector('.cursive-bg-text');
         if (textEl) {
             if (scrollP > 0.7) {
                 const fadeOp = Math.max(0, 1 - ((scrollP - 0.7) / 0.25));
                 textEl.style.opacity = fadeOp * 0.8;
             } else {
                 textEl.style.opacity = 0.8;
             }
         }
    }
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
  } else {
      // Scroll controlled animation
      const scrollP = currentScrollP.value;
      const startZ = 0.8;
      const endZ = 0.15;
      
      const currentZ = startZ - (scrollP * (startZ - endZ));
      if (camera) camera.position.z = currentZ;
      
      // Scene updates synced with RAF
      if (model) {
          model.rotation.y = scrollP * Math.PI; 
          
          const switchThreshold = 0.90;
          model.visible = scrollP < switchThreshold;
          
          gridGroup.visible = scrollP >= switchThreshold;
      }
  }

  // Slight rotation of model for dynamism REMOVED
  
  // Hover Logic
  if (gridGroup.visible) {
      raycaster.setFromCamera(pointer, camera);
      const intersects = raycaster.intersectObjects(interactionPlanes);
      
      // Determine hovered plane
      let hoveredPlane = null;
      if (intersects.length > 0) {
          hoveredPlane = intersects[0].object;
      }
      
      // Update Opacities
      interactionPlanes.forEach(plane => {
          const targetOpacity = (plane === hoveredPlane) ? 0.6 : 0.0; // 0.6 visible white
          plane.material.opacity += (targetOpacity - plane.material.opacity) * 0.1;
      });
      
      // Cursor pointer
      document.body.style.cursor = hoveredPlane ? 'pointer' : 'auto';
  }

  // Rotate Grid Models
  if (gridGroup.visible) {
      gridModels.forEach(pivot => {
          // The car is the first child (index 0) of the pivot group
          if (pivot.children[0]) {
            pivot.children[0].rotation.y += 0.01; 
          }
      });
  }




  renderer.render(scene, camera);
};

const onPointerMove = (event) => {
    pointer.x = (event.clientX / window.innerWidth) * 2 - 1;
    pointer.y = - (event.clientY / window.innerHeight) * 2 + 1;
};

const checkIntersection = (clientX, clientY) => {
    if (!gridGroup.visible) return;
    if (showAlbum.value) return; // Don't check if album is open

    pointer.x = (clientX / window.innerWidth) * 2 - 1;
    pointer.y = - (clientY / window.innerHeight) * 2 + 1;
    
    raycaster.setFromCamera(pointer, camera);
    const intersects = raycaster.intersectObjects(interactionPlanes);
    
    if (intersects.length > 0) {
        const hoveredPlane = intersects[0].object;
        if (hoveredPlane.userData.parentPivot) {
            const index = hoveredPlane.userData.parentPivot.userData.index;
            const carDef = cars[index];
            if (carDef && carDef.albumId) {
                const album = albums.find(a => a.id === carDef.albumId);
                if (album) {
                    selectedAlbum.value = album;
                    showAlbum.value = true;
                }
            }
        }
    }
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

const onResize = () => {
  if (!containerRef.value) return;
  // Canvas is sticky 100vh, so width/height is viewport usually
  const width = window.innerWidth; 
  const height = window.innerHeight;
  
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
  
  updateGridLayout();
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
        <div ref="backgroundRef" class="background-layer">
            <h1 class="cursive-bg-text">Lucca Nunes</h1>
        </div>
        <canvas ref="canvasRef" class="webgl-canvas"></canvas>
        <div ref="overlayRef" class="viewfinder-overlay">
            <div class="ocr-text top-left">REC</div>
            <div class="ocr-text top-right">BAT 100%</div>
            <!-- Focus box removed -->
            <div class="ocr-text bottom-left">ISO 800</div>
            <div class="ocr-text bottom-right">1/60 F2.8</div>
        </div>
    </div>
  </section>
</template>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Mrs+Saint+Delafield&display=swap');

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
    /* Default background white if logic hasn't run yet */
    background-color: #ffffff; 
}

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

.webgl-canvas {
  position: relative;
  width: 100%;
  height: 100%;
  display: block;
  z-index: 10;
}

.viewfinder-overlay {
    position: absolute;
    top: 0; 
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none;
    opacity: 0; /* Starts hidden */
    transition: opacity 0.1s linear;
    z-index: 20;
    border: 20px solid rgba(0,0,0,0.8); /* Simulated bezel maybe? Or just overlay */
}

/* Viewfinder UI Elements REMOVED focus box */

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



/* Loading styles moved to LoadingScreen.vue */



</style>
