<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

const isScrolled = ref(false);

const checkScroll = () => {
  isScrolled.value = window.scrollY > 50;
};

onMounted(() => {
  window.addEventListener('scroll', checkScroll);
});

onUnmounted(() => {
  window.removeEventListener('scroll', checkScroll);
});
</script>

<template>
  <nav class="navbar" :class="{ 'scrolled': isScrolled }">
    <div class="nav-links">
      <a href="#about" class="nav-item">Sobre mim</a>
      <a href="#albums" class="nav-item">Albuns</a>
      <a href="#contact" class="nav-item">Contato</a>
    </div>
  </nav>
</template>

<style scoped>
.navbar {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  padding: 2rem 0;
  z-index: 100;
  display: flex;
  justify-content: center;
  transition: all 0.4s ease;
}

.navbar.scrolled {
  padding: 1rem 0;
  background: rgba(10, 10, 10, 0.8);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.nav-links {
  display: flex;
  gap: 4rem;
  align-items: center;
}

.nav-item {
  color: rgba(255, 255, 255, 0.7);
  text-decoration: none;
  font-family: 'Outfit', sans-serif;
  text-transform: uppercase;
  font-size: 0.9rem;
  font-weight: 500;
  letter-spacing: 2px;
  position: relative;
  transition: color 0.3s ease;
}

.nav-item::after {
  content: '';
  position: absolute;
  bottom: -5px;
  left: 50%;
  transform: translateX(-50%);
  width: 0;
  height: 1px;
  background-color: #fff;
  transition: width 0.3s ease;
}

.nav-item:hover {
  color: #fff;
}

.nav-item:hover::after {
  width: 100%;
}

@media (max-width: 768px) {
  .nav-links {
    gap: 1.5rem;
  }
  
  .nav-item {
    font-size: 0.8rem;
  }
}
</style>
