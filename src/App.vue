<template>
  <div class="app-container">
    <CambodiaFlag class="flag-component" />

    <div class="portrait-overlay">
      <div class="overlay-content">
        <svg class="rotate-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <rect x="5" y="2" width="14" height="20" rx="2" ry="2"></rect>
          <line x1="12" y1="18" x2="12.01" y2="18"></line>
        </svg>
        <h2>Rotate Device</h2>
        <p>Please rotate your phone to landscape mode to experience the cinematic simulation.</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import CambodiaFlag from './components/CambodiaFlag.vue';
</script>

<style>
* {
  box-sizing: border-box;
}

html, body, #app {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
  background-color: #000; /* Prevents white flashes */
}

.app-container {
  width: 100%;
  height: 100%;
  position: relative;
}

.flag-component {
  width: 100%;
  height: 100%;
}

/* Hide the overlay by default */
.portrait-overlay {
  display: none;
}

/* Target portrait orientation on mobile/tablet devices.
  max-width ensures this doesn't accidentally trigger on a vertically snapped desktop browser window.
*/
@media screen and (max-width: 1024px) and (orientation: portrait) {
  .flag-component {
    /* Optionally hide or blur the canvas to save processing power while they rotate */
    filter: blur(10px);
    pointer-events: none;
  }

  .portrait-overlay {
    display: flex;
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(10, 10, 10, 0.95); /* Deep dark glass look */
    backdrop-filter: blur(10px);
    -webkit-backdrop-filter: blur(10px);
    z-index: 9999;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    text-align: center;
    padding: 2rem;
    color: white;
  }

  .overlay-content {
    display: flex;
    flex-direction: column;
    align-items: center;
  }

  .rotate-icon {
    width: 64px;
    height: 64px;
    margin-bottom: 24px;
    color: rgba(255, 255, 255, 0.9);
    animation: rotateDevice 2s ease-in-out infinite;
  }

  .overlay-content h2 {
    margin: 0 0 12px 0;
    font-size: 1.5rem;
    font-weight: 600;
    letter-spacing: 1px;
  }

  .overlay-content p {
    margin: 0;
    color: rgba(255, 255, 255, 0.6);
    font-size: 0.95rem;
    line-height: 1.5;
    max-width: 280px;
  }
}

/* Subtle animation hinting the user to rotate 90 degrees */
@keyframes rotateDevice {
  0% { transform: rotate(0deg); }
  40%, 60% { transform: rotate(-90deg); }
  100% { transform: rotate(0deg); }
}
</style>