<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';
import { RoomEnvironment } from 'three/examples/jsm/environments/RoomEnvironment.js';

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

// Grid of Cars
const carValues = [
    { path: '/models/auditt.glb', scale: 0.8, name: 'Audi TT' },
    { path: '/models/ferrari.glb', scale: 0.8, name: 'Ferrari' },
    { path: '/models/gallardo.glb', scale: 0.8, name: 'Gallardo' },
    { path: '/models/lambo.glb', scale: 0.8, name: 'Lamborghini' },
    { path: '/models/masserati.glb', scale: 0.8, name: 'Maserati' },
    { path: '/models/mclaren.glb', scale: 0.8, name: 'McLaren' },
];
const gridModels = [];
const gridGroup = new THREE.Group();
gridGroup.visible = false; // Hidden initially

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
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.shadowMap.enabled = true;
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
  const loader = new GLTFLoader();
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
};

// Store grid objects for responsive updates
let gridPivots = [];

const loadGrid = (loader) => {
    scene.add(gridGroup);
    
    // Load all models first
    carValues.forEach((carDef, index) => {
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
        const totalRows = Math.ceil(carValues.length / cols);
        
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
    if (!containerRef.value || !camera) return;
    
    // If falling animation is still running, letting it finish avoids conflict.
    // Ideally we might want to blend, but simple blocking is safer for now.
    if (isAnimating) return;
    
    // Check if we are past the initial animation
    // But actually, let's map scroll directly if the user wants "scroll down effect"
    // The previous request was "falls from top", then "scroll gives zoom".
    
    const rect = containerRef.value.getBoundingClientRect();
    const viewportHeight = window.innerHeight;
    
    // Calculate progress through the section (which will be taller now)
    // 0 = top of section enters viewport
    // 1 = bottom of section leaves viewport
    // But we want it pinned.
    
    // progress = (viewportHeight - rect.top) / (rect.height + viewportHeight); 
    // This is valid for scrolling through.
    
    // Let's use simple sticky logic:
    // If top is <= 0, map -top to progress.
    
    let scrollP = -rect.top / (rect.height - viewportHeight);
    scrollP = Math.max(0, Math.min(1, scrollP));
    
    // Map scrollP to Camera Z
    // Start at our "close" position (Z=0.8) and zoom INTO the lens (Z=0.1 or less)
    // Wait, the user wants "falls then scroll".
    
    // If animation is running (falling), let it finish or override?
    // Let's assume falling happens on entry, then scrolling takes over.
    
    if (!isAnimating) {
         // Scroll moves from 'targetCameraPos' (0.8) to 'insidePos' (0.1)
         const startZ = 0.8;
         const endZ = 0.15;
         
         const currentZ = startZ - (scrollP * (startZ - endZ));
         camera.position.z = currentZ;
         
         // Update overlay opacity
         if(overlayRef.value) {
             overlayRef.value.style.opacity = scrollP > 0.8 ? (scrollP - 0.8) * 5 : 0;
         }

         // Rotate model inversely to scroll to reveal the back
         if (model) {
             model.rotation.y = scrollP * Math.PI; 
             // Hide model at the very end to simulate looking "through" it
             model.visible = scrollP < 0.98;
             
             // Show Grid only at the end
             gridGroup.visible = scrollP >= 0.98;
         }

         // Update background color (White to Black)
         if (backgroundRef.value) {
             const val = Math.floor(255 * (1 - scrollP));
             backgroundRef.value.style.backgroundColor = `rgb(${val}, ${val}, ${val})`;
             
             // Fade out text at the end
             const textEl = backgroundRef.value.querySelector('.cursive-bg-text');
             if (textEl) {
                 if (scrollP > 0.7) {
                     // Fade from 1 down to 0 between 0.7 and 0.95
                     const fadeOp = Math.max(0, 1 - ((scrollP - 0.7) / 0.25));
                     textEl.style.opacity = fadeOp * 0.8; // Multiply by base opacity
                 } else {
                     textEl.style.opacity = 0.8;
                 }
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
  }

  // Slight rotation of model for dynamism REMOVED
  
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
  if (renderer) renderer.dispose();
});

</script>

<template>
  <section ref="containerRef" class="cannon-section">
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
  height: 400vh; /* Very tall to allow scrolling */
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
    font-size: 15rem; /* Very large */
    color: #b91004ff; /* Gold */
    margin: 0;
    opacity: 0.8;
    pointer-events: none;
    text-align: center;
    line-height: 1;
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


</style>
