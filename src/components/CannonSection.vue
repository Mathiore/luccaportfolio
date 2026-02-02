<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue';
import * as THREE from 'three';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';
import { RoomEnvironment } from 'three/examples/jsm/environments/RoomEnvironment.js';

const canvasRef = ref(null);
const containerRef = ref(null);
const overlayRef = ref(null);
const stickyWrapperRef = ref(null);
let animationId = null;
let scene, camera, renderer, model;
let targetCameraPos = new THREE.Vector3(0, 0, 0.8); // Final position (Close Up)
let startCameraPos = new THREE.Vector3(0, 15, 5.0); // Start position (High and aligned for swoop)
let animationProgress = 0;
let isAnimating = false;

// Initialize Three.js
const init = () => {
  // Use window dimensions because the canvas is in a sticky 100vh container
  // accessing containerRef (which is 400vh) caused the renderer to be massive and distorted.
  const width = window.innerWidth;
  const height = window.innerHeight;

  // Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x000000); // Black background as per theme

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

  // Handle Resize
  window.addEventListener('resize', onResize);
  
  // Handle Scroll
  window.addEventListener('scroll', onScroll);
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
        <canvas ref="canvasRef" class="webgl-canvas"></canvas>
        <div ref="overlayRef" class="viewfinder-overlay">
            <div class="ocr-text top-left">REC</div>
            <div class="ocr-text top-right">BAT 100%</div>
            <div class="focus-box"></div>
            <div class="ocr-text bottom-left">ISO 800</div>
            <div class="ocr-text bottom-right">1/60 F2.8</div>
        </div>
    </div>
  </section>
</template>

<style scoped>
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
}

.webgl-canvas {
  width: 100%;
  height: 100%;
  display: block;
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

/* Viewfinder UI Elements */
.focus-box {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    width: 150px;
    height: 100px;
    border: 2px solid rgba(255, 255, 255, 0.8);
}

.focus-box::after {
    content: '+';
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    color: white;
    font-size: 20px;
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
