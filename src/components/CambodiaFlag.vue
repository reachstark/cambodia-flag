<script setup>
import VideoExportModal from './VideoExportModal.vue';
import { gsap } from 'gsap';
import { ref, onMounted, onBeforeUnmount, watch } from 'vue';
import * as THREE from 'three';
import { library } from '@fortawesome/fontawesome-svg-core';
import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome';
import { 
  faPlayCircle, faPlay, faPause, faSync, 
  faVolumeHigh, faVolumeXmark, faClock, 
  faSliders, faWind, faExpandArrowsAlt, faWater, 
  faXmark, faCamera, faSun, faLightbulb, faVideo, faStar,
  faChevronDown, 
  faTriangleExclamation, faDownload,
  faMusic
  
} from '@fortawesome/free-solid-svg-icons';
import { FFmpeg } from '@ffmpeg/ffmpeg';
import { fetchFile, toBlobURL } from '@ffmpeg/util';

// Add icons to the library
library.add(
  faPlayCircle, faPlay, faPause, faSync, 
  faVolumeHigh, faVolumeXmark, faClock, 
  faSliders, faWind, faExpandArrowsAlt, faWater, 
  faXmark, faCamera, faSun, faLightbulb, faVideo, faStar,  
  faChevronDown, 
  faTriangleExclamation, faDownload, faMusic
);

const isModalOpen = ref(false);
const isExporting = ref(false);
const exportProgress = ref(0);
const exportStatus = ref("");
const shouldCancelExport = ref(false);
let activeFFmpeg = null;

const tracks = [
  { name: 'Vocal Version', path: '/anthem.mp3' },
  { name: 'Instrumental', path: '/anthem_instrumental.mp3' },
];

const currentTrackPath = ref(tracks[0].path);

const changeTrack = async () => {
  if (!anthemAudio.value) return;
  const wasPlaying = isPlaying.value;
  anthemAudio.value.src = currentTrackPath.value;
  anthemAudio.value.load();
  // If it was playing before the switch, resume playing the new track
  if (wasPlaying) {
    try {
      await anthemAudio.value.play();
      isPlaying.value = true;
    } catch (e) {
      console.error("Playback failed after track change:", e);
      isPlaying.value = false;
    }
  } else {
    isPlaying.value = false;
  }
};

const handleCancelExport = async () => {
  shouldCancelExport.value = true;
  exportStatus.value = "Aborting process...";

  // If we are in the Encoding Stage (Stage 2), we must kill the worker
  if (activeFFmpeg) {
    try {
      await activeFFmpeg.terminate(); 
      console.log("FFmpeg Worker terminated.");
    } catch (e) {
      console.log("FFmpeg already inactive.");
    }
  }
  
  // Call your cleanup function to reset the UI/Camera
  // We use a small delay to ensure the worker is actually dead
  setTimeout(() => {
    restoreAppState(window.innerWidth, window.innerHeight);
  }, 100);
};

const handleExportStart = async (config) => {
  isExporting.value = true;
  exportStatus.value = "Initializing FFmpeg...";

  try {
    await renderOfflineVideo(config);
  } catch (error) {
    console.error("Export failed:", error);
    exportStatus.value = "Error: Check console.";
    isExporting.value = false;
  }
};

const restoreAppState = (originalW, originalH) => {
  // 1. Restore Renderer & Camera to fit the user's actual screen
  renderer.setSize(originalW, originalH);
  camera.aspect = originalW / originalH;
  camera.updateProjectionMatrix();
  
  // 2. Re-run your flag filling logic
  if (typeof resizeFlagToFillScreen === 'function') {
    resizeFlagToFillScreen();
  }

  // 3. Reset UI States
  isExporting.value = false;
  isModalOpen.value = false;
  shouldCancelExport.value = false;
  exportProgress.value = 0;

  // 4. Restart the live animation loop
  animate(performance.now());
};


/**
 * Standard Single-Threaded Loader
 */
const loadFFmpeg = async () => {
  // Switched from core-mt to core for stability
  const baseURL = 'https://unpkg.com/@ffmpeg/core@0.12.6/dist/esm';
  
  console.log('--- 🚀 FFmpeg Loading (Single-Threaded) ---');
  
  try {
    const coreURL = await toBlobURL(`${baseURL}/ffmpeg-core.js`, 'text/javascript');
    const wasmURL = await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, 'application/wasm');

    await activeFFmpeg.load({
      coreURL,
      wasmURL,
      // workerURL is not used in single-threaded mode
    });
    
    console.log('✅ FFmpeg Engine Ready');
  } catch (error) {
    console.error("FFmpeg Load Error:", error);
    throw new Error(`FFmpeg Load Failed: ${error.message}`);
  }
};

/**
 * Robust Render Engine (Single-Threaded Version)
 */
const renderOfflineVideo = async (config) => {
  if (!anthemAudio.value || !renderer || !camera) return;

  // 1. Initialize State
  shouldCancelExport.value = false;
  isExporting.value = true;
  exportProgress.value = 0;
  exportStatus.value = "Preparing engine...";

  const { width, height, fps, duration, format } = config;
  const totalFrames = Math.ceil(duration * fps);
  const originalWidth = window.innerWidth;
  const originalHeight = window.innerHeight;

  // 2. Prepare Viewport
  renderer.setSize(width, height, false);
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  if (typeof resizeFlagToFillScreen === 'function') resizeFlagToFillScreen();

  // 3. Setup Instance
  activeFFmpeg = new FFmpeg();
  
  activeFFmpeg.on('progress', ({ progress }) => {
    // Stage 2: Encoding (50% to 100%)
    exportProgress.value = 50 + (progress * 50);
    exportStatus.value = `Encoding ${format.toUpperCase()}... \nPlease keep this tab active.`;
  });

  try {
    // Load the standard engine
    await loadFFmpeg();
    
    cancelAnimationFrame(animationId);

    // 4. Stage 1: Render Loop (Capture Frames)
    for (let frame = 0; frame < totalFrames; frame++) {
      if (shouldCancelExport.value) throw new Error("CANCELLED_BY_USER");

      updateFlagPhysics(frame / fps);
      renderer.render(scene, camera);

      // Capture frame as JPEG
      const blob = await new Promise(res => renderer.domElement.toBlob(res, 'image/jpeg', 0.92));
      const frameData = new Uint8Array(await blob.arrayBuffer());
      const frameName = `frame_${frame.toString().padStart(5, '0')}.jpg`;
      
      await activeFFmpeg.writeFile(frameName, frameData);

      exportProgress.value = (frame / totalFrames) * 50;
      exportStatus.value = `Rendering frame ${frame + 1}/${totalFrames}...`;

      // Keep UI responsive
      if (frame % 5 === 0) await new Promise(res => setTimeout(res, 1));
    }

    if (shouldCancelExport.value) throw new Error("CANCELLED_BY_USER");

    // 5. Stage 2: Audio Muxing & Encoding
    exportStatus.value = "Processing audio layers...";
    const audioData = await fetchFile('/anthem.mp3');
    await activeFFmpeg.writeFile('audio.mp3', audioData);

    const bitrate = width > 1920 ? '12M' : (width > 1280 ? '8M' : '4M');
    const outputFileName = `output.${format}`;

    // 6. FFmpeg Command (Removed -threads 0 for single-threaded stability)
    const ffmpegArgs = [
      '-framerate', `${fps}`,
      '-i', 'frame_%05d.jpg',
      '-i', 'audio.mp3',
      '-preset', 'ultrafast', // Still use ultrafast to minimize processing time
      '-shortest'
    ];

    if (format === 'mp4') {
      ffmpegArgs.push(
        '-c:v', 'libx264', 
        '-pix_fmt', 'yuv420p', 
        '-c:a', 'aac', 
        '-b:v', bitrate, 
        outputFileName
      );
    } else {
      ffmpegArgs.push(
        '-c:v', 'libvpx', 
        '-c:a', 'libvorbis', 
        '-b:v', bitrate, 
        outputFileName
      );
    }

    exportStatus.value = `Encoding video sequence...`;
    await activeFFmpeg.exec(ffmpegArgs);

    if (shouldCancelExport.value) throw new Error("CANCELLED_BY_USER");

    // 6. Download
    exportProgress.value = 100;
    exportStatus.value = "Finalizing download...";
    
    const data = await activeFFmpeg.readFile(outputFileName);
    const downloadUrl = URL.createObjectURL(new Blob([data.buffer], { type: `video/${format}` }));
    
    const link = document.createElement('a');
    link.href = downloadUrl;
    link.download = `Cambodia-Flag-Cinematic.${format}`;
    link.click();

    URL.revokeObjectURL(downloadUrl);
    restoreAppState(originalWidth, originalHeight);

  } catch (error) {
    console.error("Export Halted Error:", error);
    restoreAppState(originalWidth, originalHeight);
  } finally {
    activeFFmpeg = null;
  }
};



// --- UI & AUDIO STATE ---
const currentPreset = ref('Landscape');
const container = ref(null);
const anthemAudio = ref(null);
const hasStarted = ref(false);
const isSidebarVisible = ref(false);
const isHoveringTrigger = ref(false);
const sunIntensity = ref(8.0);
const isFlatMode = ref(false);

const presets = {
  'Landscape': 0,
  'Artistic': Math.PI / 6.5,
};

const applyPreset = (name) => {
  currentPreset.value = name;
  const targetRotation = presets[name];

  // Smoothly animate the rotation
  gsap.to(flagMesh.rotation, {
    z: targetRotation,
    duration: 1.2,
    ease: "power2.inOut",
    onUpdate: () => {
      resizeFlagToFillScreen();
    }
  });
};

// Audio Reactive Refs
const isPlaying = ref(false);
const isMuted = ref(false);
const volume = ref(1);
const currentTime = ref(0);
const duration = ref(0);
const isSeeking = ref(false);

// Animation Reactive Refs
const waveSpeed = ref(3);
const waveStrength = ref(0.15);
const waveDensity = ref(10);

// --- AUDIO LOGIC ---
const startExperience = () => {
  if (!anthemAudio.value) return;
  if (!hasStarted.value) {
    hasStarted.value = true;
  }

  togglePlay();
};

const togglePlay = () => {
  if (!anthemAudio.value) return;
  if (anthemAudio.value.paused) {
    anthemAudio.value.play().then(() => isPlaying.value = true).catch(e => console.error(e));
  } else {
    anthemAudio.value.pause();
    isPlaying.value = false;
  }
};

const restartAudio = () => {
  if (!anthemAudio.value) return;
  anthemAudio.value.currentTime = 0;
  if (!isPlaying.value) togglePlay();
};

const updateVolume = () => {
  if (anthemAudio.value) {
    anthemAudio.value.volume = volume.value;
    // If user touches volume slider, automatically unmute
    if (isMuted.value) {
      isMuted.value = false;
      anthemAudio.value.muted = false;
    }
  }
};

const onMetadataLoaded = () => {
  duration.value = anthemAudio.value.duration;
};

const onTimeUpdate = () => {
  if (!isSeeking.value && anthemAudio.value) {
    currentTime.value = anthemAudio.value.currentTime;
  }
};

const seekAudio = () => {
  if (anthemAudio.value) {
    anthemAudio.value.currentTime = currentTime.value;
  }
};

const formatTime = (seconds) => {
  if (!seconds || isNaN(seconds)) return "0:00";
  const m = Math.floor(seconds / 60);
  const s = Math.floor(seconds % 60).toString().padStart(2, '0');
  return `${m}:${s}`;
};

// --- KEYBOARD LOGIC ---
const handleKeyboardControls = (event) => {
  if (!anthemAudio.value) return;
  const audio = anthemAudio.value;

  switch (event.code) {
    // -- Lighting Modes ---
    case 'KeyS':
      isFlatMode.value = false;
      break;
      case 'KeyO':
      isFlatMode.value = true;
      break;
    // --- NEW CAMERA SHORTCUTS ---
    case 'KeyL':
      applyPreset('Landscape');
      break;
    case 'KeyA':
      applyPreset('Artistic');
      break;

    case 'Space':
      event.preventDefault();
      if (audio.paused) {
        audio.play().then(() => isPlaying.value = true).catch(() => console.log("Blocked"));
      } else {
        audio.pause();
        isPlaying.value = false;
      }
      break;
    case 'KeyM':
      audio.muted = !audio.muted;
      isMuted.value = audio.muted; // Sync Vue state
      break;
    case 'ArrowUp':
      event.preventDefault();
      audio.volume = Math.min(1.0, audio.volume + 0.1);
      volume.value = audio.volume; 
      if(isMuted.value) { audio.muted = false; isMuted.value = false; }
      break;
    case 'ArrowDown':
      event.preventDefault();
      audio.volume = Math.max(0.0, audio.volume - 0.1);
      volume.value = audio.volume; 
      if(isMuted.value) { audio.muted = false; isMuted.value = false; }
      break;
    case 'KeyR':
      audio.currentTime = 0;
      if (audio.paused) {
        audio.play().then(() => isPlaying.value = true).catch(() => console.log("Blocked"));
      }
      break;
  }
};

// --- THREE.JS LOGIC ---
let scene, camera, renderer, flagMesh, sunLight, animationId, currentScale = 1, standardMaterial, flatMaterial;

onMounted(() => {
  window.addEventListener('keydown', handleKeyboardControls);

  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
  camera.position.z = 5;

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.outputColorSpace = THREE.SRGBColorSpace; 
  container.value.appendChild(renderer.domElement);

  const geometry = new THREE.PlaneGeometry(1, 1, 128, 128);

  const textureLoader = new THREE.TextureLoader();
  textureLoader.crossOrigin = 'anonymous';
  const texture = textureLoader.load('/cambodia-flag.png');
  texture.colorSpace = THREE.SRGBColorSpace;
  texture.anisotropy = renderer.capabilities.getMaxAnisotropy();

  standardMaterial = new THREE.MeshStandardMaterial({
    map: texture,
    side: THREE.DoubleSide,
    roughness: 0.7,
    metalness: 0.1,
  });

  flatMaterial = new THREE.MeshBasicMaterial({
    map: texture,
    side: THREE.DoubleSide,
  });

  flagMesh = new THREE.Mesh(geometry, standardMaterial);
  scene.add(flagMesh);

  sunLight = new THREE.DirectionalLight(0xffeedd, sunIntensity.value); 
    sunLight.position.set(5, 5, 3);
    scene.add(sunLight);

  resizeFlagToFillScreen();
  window.addEventListener('resize', onWindowResize);
  
  animate(0);
});

watch(isFlatMode, (isFlat) => {
  if (flagMesh) {
    flagMesh.material = isFlat ? flatMaterial : standardMaterial;
  }
});

watch(sunIntensity, (newIntensity) => {
  if (sunLight) {
    sunLight.intensity = newIntensity;
  }
});

// --- DETERMINISTIC PHYSICS ---
const updateFlagPhysics = (exactTimeInSeconds) => {
  const scaledTime = exactTimeInSeconds * waveSpeed.value; 

  const densitySoftener = Math.max(1, waveDensity.value / 6);
  const safeStrength = waveStrength.value / densitySoftener;

  const positions = flagMesh.geometry.attributes.position;
  const uvs = flagMesh.geometry.attributes.uv;

  for (let i = 0; i < positions.count; i++) {
    const uvX = uvs.getX(i); 
    const uvY = uvs.getY(i); 
    
    // We apply the displacement based on the forced time, not real time
    const zDisplacement = Math.sin((uvX * waveDensity.value) + (uvY * 2) - scaledTime) * safeStrength;
    positions.setZ(i, zDisplacement);
  }

  positions.needsUpdate = true;
  flagMesh.geometry.computeVertexNormals(); 
};



const resizeFlagToFillScreen = () => {
  if (!camera || !flagMesh) return;
  
  const vFOV = (camera.fov * Math.PI) / 180;
  const visibleHeight = 2 * Math.tan(vFOV / 2) * camera.position.z;
  const visibleWidth = visibleHeight * camera.aspect;

  const targetWidth = visibleWidth * 1.2;
  const targetHeight = visibleHeight * 1.15;

  // THE MAGIC FIX: Scale the actual geometry vertices, not the mesh scale!
  const positions = flagMesh.geometry.attributes.position;
  const uvs = flagMesh.geometry.attributes.uv;

  for (let i = 0; i < positions.count; i++) {
    const uvX = uvs.getX(i);
    const uvY = uvs.getY(i);
    
    // We subtract 0.5 to keep the flag perfectly centered around 0,0
    positions.setXY(i, (uvX - 0.5) * targetWidth, (uvY - 0.5) * targetHeight);
  }

  positions.needsUpdate = true;
  
  // Lock the mesh scale to 1 so the normal matrix never distorts!
  flagMesh.scale.set(1, 1, 1);
};

const onWindowResize = () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
  resizeFlagToFillScreen();
};

const animate = (timestamp) => {
  animationId = requestAnimationFrame(animate);

  const liveTime = timestamp / 1000; 

  updateFlagPhysics(liveTime);

  renderer.render(scene, camera);
};

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeyboardControls);
  cancelAnimationFrame(animationId);
  window.removeEventListener('resize', onWindowResize);
  if (renderer) renderer.dispose();
  if (flagMesh) {
    flagMesh.geometry.dispose();
    flagMesh.material.dispose();
  }
});

const toggleMute = () => {
  if (!anthemAudio.value) return;
  // Flip the audio element's mute state
  anthemAudio.value.muted = !anthemAudio.value.muted;
  // Sync our Vue reactive variable with the audio element
  isMuted.value = anthemAudio.value.muted;
};
</script>

<template>
  <div class="app-wrapper">
    <div class="canvas-container" ref="container" @click="startExperience">
      <audio 
        ref="anthemAudio"
        :src="currentTrackPath"
        loop 
        @timeupdate="onTimeUpdate" 
        @loadedmetadata="onMetadataLoaded"
      >
      </audio>
    </div>

    <div class="toggle-trigger-zone" @mouseenter="isHoveringTrigger = true" @mouseleave="isHoveringTrigger = false">
    <button 
        class="floating-toggle-btn" 
        :class="{ 'visible': !isSidebarVisible && isHoveringTrigger }" 
        @click.stop="isSidebarVisible = true"
    >
        <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <line x1="4" y1="21" x2="4" y2="14"></line><line x1="4" y1="10" x2="4" y2="3"></line>
        <line x1="12" y1="21" x2="12" y2="12"></line><line x1="12" y1="8" x2="12" y2="3"></line>
        <line x1="20" y1="21" x2="20" y2="16"></line><line x1="20" y1="12" x2="20" y2="3"></line>
        <line x1="1" y1="14" x2="7" y2="14"></line><line x1="9" y1="8" x2="15" y2="8"></line><line x1="17" y1="16" x2="23" y2="16"></line>
        </svg>
    </button>
</div>

    <div class="sidebar" :class="{ 'active': isSidebarVisible }" @click.stop>
      <div class="sidebar-header">
        <h2>Control Panel</h2>
        <button class="close-btn" @click="isSidebarVisible = false">
          <font-awesome-icon icon="xmark" />
        </button>
      </div>

      <div class="sidebar-content">
        <div class="control-group">
          <h3>
            <font-awesome-icon icon="play-circle" /> National Anthem Playback
          </h3>

          <div class="audio-selector-wrapper">
            <div class="slider-header">
              <label><font-awesome-icon icon="music" /> Select Audio</label>
            </div>
            <div class="select-wrapper">
              <select v-model="currentTrackPath" @change="changeTrack">
                <option v-for="track in tracks" :key="track.path" :value="track.path">
                  {{ track.name }}
                </option>
              </select>
              <font-awesome-icon icon="chevron-down" class="select-arrow" />
            </div>
          </div>

          <div class="btn-row">
            <button class="action-btn" @click="togglePlay">
              <font-awesome-icon :icon="isPlaying ? 'pause' : 'play'" /> 
               {{ isPlaying ? 'Pause' : 'Play' }}
              <span class="kbd-hint">Space</span>
            </button>
            <button class="action-btn" @click="restartAudio">
              <font-awesome-icon icon="sync" /> Restart 
              <span class="kbd-hint">R</span>
            </button>
          </div>

          <div class="slider-wrapper">
            <div class="slider-header">
              <label>
                <font-awesome-icon :icon="isMuted ? 'volume-xmark' : 'volume-high'" /> Volume
                <span class="kbd-hint">↑</span><span class="kbd-hint">↓</span><span class="kbd-hint">M</span>
              </label>
              
              <div class="mute-toggle-wrapper">
                <button class="mute-toggle-btn" @click="toggleMute" title="Click to toggle mute">
                  <div class="mute-toggle-inner">
                    <span :class="{ 'text-muted-alert': isMuted }">
                      {{ isMuted ? 'Muted' : Math.round(volume * 100) + '%' }}
                    </span>
                  </div>
                </button>
              </div>
            </div>

            <div class="ios-volume-wrapper" :style="{ opacity: isMuted ? 0.5 : 1 }">
              
              <span class="ios-overlay-symbol minus">−</span>
              
              <input 
                type="range" 
                min="0" 
                max="1" 
                step="0.01" 
                v-model.number="volume" 
                @input="updateVolume" 
                :disabled="isMuted"
                class="ios-slider"
                :style="{ background: `linear-gradient(to right, #ffffff ${volume * 100}%, rgba(255, 255, 255, 0.15) ${volume * 100}%)` }"
              />
              
              <span class="ios-overlay-symbol plus">+</span>
              
            </div>
          </div>

          <div class="premium-player-progress">
            <div class="progress-track-container">
              <input 
                type="range" 
                min="0" 
                :max="duration" 
                step="0.1" 
                v-model.number="currentTime" 
                @mousedown="isSeeking = true" 
                @mouseup="isSeeking = false" 
                @input="seekAudio"
                class="premium-progress-slider"
                :style="{ background: `linear-gradient(to right, #ffffff ${duration ? (currentTime / duration) * 100 : 0}%, rgba(255, 255, 255, 0.15) ${duration ? (currentTime / duration) * 100 : 0}%)` }"
              />
            </div>
            
            <div class="time-footer">
              <span class="time-current">{{ formatTime(currentTime) }}</span>
              <span class="time-duration">{{ formatTime(duration) }}</span>
            </div>
          </div>
        </div>

        <hr class="divider" />

        <div class="control-group">
          <h3><font-awesome-icon icon="camera" /> View Presets</h3>
          <div class="preset-grid">
            <button 
              v-for="(angle, name) in presets" 
              :key="name"
              class="preset-btn"
              :class="{ 'active': currentPreset === name }"
              @click="applyPreset(name)"
            >
              {{ name }}
              <span v-if="name === 'Landscape'" class="kbd-hint">L</span>
              <span v-if="name === 'Artistic'" class="kbd-hint">A</span>
            </button>
          </div>
        </div>

        <hr class="divider" />

        <div class="control-group">
          <h3>
            <font-awesome-icon icon="sliders" /> Environment
          </h3>

          <div class="slider-wrapper" style="margin-bottom: 12px;">
            <div class="slider-header" style="margin-bottom: 8px;">
              <label><font-awesome-icon icon="lightbulb" /> Lighting Mode</label>
            </div>
            <div class="preset-grid">
              <button 
                class="preset-btn" 
                :class="{ 'active': !isFlatMode }" 
                @click="isFlatMode = false"
              >
                Sunlight
                <span class="kbd-hint">S</span>
              </button>
              <button 
                class="preset-btn" 
                :class="{ 'active': isFlatMode }" 
                @click="isFlatMode = true"
              >
                Original
                <span class="kbd-hint">O</span>
              </button>
            </div>
          </div>

          <div 
            class="slider-wrapper" 
            :style="{ 
              opacity: isFlatMode ? 0.4 : 1, 
              pointerEvents: isFlatMode ? 'none' : 'auto',
              transition: 'opacity 0.3s ease'
            }"
          >
            <div class="slider-header">
              <label><font-awesome-icon icon="sun" /> Sun Intensity</label>
              <span>{{ sunIntensity.toFixed(1) }}</span>
            </div>
            <input 
              type="range" min="6" max="20" step="0.1" 
              v-model.number="sunIntensity" 
              class="env-pill-slider"
              :style="{ background: `linear-gradient(to right, #ffffff ${((sunIntensity - 6) / 14) * 100}%, rgba(255, 255, 255, 0.1) ${((sunIntensity - 6) / 14) * 100}%)` }"
            />
          </div>
          
          <div class="slider-wrapper">
            <div class="slider-header">
              <label><font-awesome-icon icon="wind" /> Wind Speed</label>
              <span>{{ waveSpeed }}x</span>
            </div>
            <input 
              type="range" min="0" max="10" step="0.1" 
              v-model.number="waveSpeed" 
              class="env-pill-slider"
              :style="{ background: `linear-gradient(to right, #ffffff ${(waveSpeed / 10) * 100}%, rgba(255, 255, 255, 0.1) ${(waveSpeed / 10) * 100}%)` }"
            />
          </div>

          <div class="slider-wrapper">
            <div class="slider-header">
              <label><font-awesome-icon icon="expand-arrows-alt" /> Wave Strength</label>
              <span>{{ waveStrength.toFixed(2) }}</span>
            </div>
            <input 
              type="range" min="0" max="0.5" step="0.01" 
              v-model.number="waveStrength" 
              class="env-pill-slider"
              :style="{ background: `linear-gradient(to right, #ffffff ${(waveStrength / 0.5) * 100}%, rgba(255, 255, 255, 0.1) ${(waveStrength / 0.5) * 100}%)` }"
            />
          </div>

          <div class="slider-wrapper">
            <div class="slider-header">
              <label><font-awesome-icon icon="water" /> Ripple Density</label>
              <span>{{ waveDensity }}</span>
            </div>
            <input 
              type="range" min="1" max="30" step="1" 
              v-model.number="waveDensity" 
              class="env-pill-slider"
              :style="{ background: `linear-gradient(to right, #ffffff ${((waveDensity - 1) / 29) * 100}%, rgba(255, 255, 255, 0.1) ${((waveDensity - 1) / 29) * 100}%)` }"
            />
          </div>
        </div>

        <hr class="divider" />

        <div class="control-group">
          <h3 class="header-with-tag">
            <span><font-awesome-icon icon="video" /> Export</span>
            <span class="beta-badge">BETA</span>
          </h3>
          
          <button 
            class="action-btn" 
            @click="isModalOpen = true"
          >
            <font-awesome-icon icon="download" />
            Save Video
          </button>
        </div>
      </div>

      <div class="developer-credit">
        Developed by <strong>Le Boritheareach</strong>
      </div>
    </div>

    <VideoExportModal 
      :is-open="isModalOpen" 
      :is-processing="isExporting"
      :progress="exportProgress"
      :max-audio-duration="duration"
      :status-message="exportStatus"
      @close="isModalOpen = false"
      @cancel="handleCancelExport"  @start-render="handleExportStart"
    />
  </div>
</template>



<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600&display=swap');

.app-wrapper {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  font-family: 'Inter', sans-serif;
  /* We removed the background-image from here so we can animate it on a dedicated layer */
}

/* --- THE ANIMATED CINEMATIC SKY --- */
.app-wrapper::before {
  content: "";
  position: absolute;
  /* Make the background slightly larger than the screen so we have room to move it */
  top: -5%;
  left: -5%;
  width: 110%; 
  height: 110%;
  
  background-image: url('/sky-view.png'); 
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;

  z-index: -1; 

  animation: slowSkyPan 40s linear infinite alternate;
}

/* GPU-Accelerated Animation Keyframes */
@keyframes slowSkyPan {
  0% {
    transform: translate(0, 0) scale(1);
  }
  100% {
    /* Slowly drifts up and left while zooming in 5% */
    transform: translate(-2%, -2%) scale(1.05);
  }
}

.canvas-container {
  width: 100%;
  height: 100%;
  cursor: pointer;
}

/* --- TOGGLE TRIGGER ZONE --- */
.toggle-trigger-zone {
  position: absolute;
  top: 0;
  right: 0;
  width: 100px; /* Area the user needs to hover over */
  height: 100px;
  z-index: 50;
  display: flex;
  justify-content: center;
  align-items: center;
}

.floating-toggle-btn {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(8px);
  border: 1px solid rgba(255, 255, 255, 0.2);
  color: white;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  
  /* Initial State: Completely Hidden */
  opacity: 0;
  transform: scale(0.8);
  pointer-events: none; 
  transition: all 0.4s cubic-bezier(0.23, 1, 0.32, 1);
}

/* Visible State: Triggered by Hover */
.floating-toggle-btn.visible {
  opacity: 1;
  transform: scale(1);
  pointer-events: auto;
}

.floating-toggle-btn:hover {
  background: rgba(255, 255, 255, 0.2);
  transform: scale(1.1);
}

.floating-toggle-btn svg {
  width: 20px;
  height: 20px;
}

.floating-toggle-btn.hidden {
  opacity: 0;
  transform: scale(0.8) rotate(90deg);
  pointer-events: none;
}

/* --- PREMIUM SIDEBAR --- */
.sidebar {
  position: absolute;
  top: 0;
  right: 0;
  width: 360px;
  height: 100vh;
  background: rgba(15, 15, 15, 0.65);
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
  border-left: 1px solid rgba(255, 255, 255, 0.1);
  color: #fff;
  display: flex;
  flex-direction: column;
  z-index: 100;
  transform: translateX(100%);
  transition: transform 0.4s cubic-bezier(0.25, 1, 0.5, 1);
  box-shadow: -10px 0 30px rgba(0, 0, 0, 0.5);
}

.sidebar.active {
  transform: translateX(0);
}

.sidebar-header {
  padding: 24px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.sidebar-header h2 {
  margin: 0;
  font-size: 1.25rem;
  font-weight: 600;
  letter-spacing: 0.5px;
}

.close-btn {
  background: transparent;
  border: none;
  color: rgba(255, 255, 255, 0.6);
  font-size: 1.2rem;
  cursor: pointer;
  transition: color 0.2s;
  padding: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.close-btn:hover {
  color: white;
}

.sidebar-content {
  flex: 1;
  padding: 24px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 16px;
  scrollbar-width: thin;
  scrollbar-color: rgba(255, 255, 255, 0.2) transparent;
}

/* For Chrome, Edge, and Safari */
.sidebar-content::-webkit-scrollbar {
  width: 6px; /* Thin and modern */
}

.sidebar-content::-webkit-scrollbar-track {
  background: transparent; /* Makes the track invisible */
}

.sidebar-content::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.2); /* Semi-transparent white */
  border-radius: 10px; /* Fully rounded pill shape */
  border: 1px solid rgba(255, 255, 255, 0.05); /* Very subtle border */
}

.sidebar-content::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.35); /* Brighter when interacting */
}

h3 {
  margin: 0 0 16px 0;
  font-size: 0.85rem;
  font-weight: 600;
  text-transform: uppercase;
  color: rgba(255, 255, 255, 0.5);
  letter-spacing: 1.5px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.control-group {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

/* --- KEYBOARD HINTS --- */
.kbd-hint {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 18px;
  padding: 2px 5px;
  margin-left: 6px;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
  font-size: 0.65rem;
  font-weight: 600;
  color: rgba(255, 255, 255, 0.8);
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 4px;
  box-shadow: 0 2px 0 rgba(0, 0, 0, 0.2);
  vertical-align: middle;
}

label .kbd-hint {
  margin-left: 4px;
}

.text-muted {
  color: #ff5555 !important;
  font-weight: 600;
}

.btn-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}

.action-btn {
  flex: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 10px;
  background: rgba(255, 255, 255, 0.1);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.15);
  border-radius: 8px;
  font-size: 0.9rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  gap: 8px; /* Space between icon and text */
}

.action-btn:hover {
  background: rgba(255, 255, 255, 0.2);
  border-color: rgba(255, 255, 255, 0.3);
}

.action-btn:active {
  transform: scale(0.97);
}

/* --- CUSTOM RANGE SLIDERS --- */
.slider-wrapper {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.slider-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.85rem;
  gap: 8px;
}

.slider-header label {
  color: rgba(255, 255, 255, 0.85);
  font-weight: 500;
  display: flex;
  align-items: center;
  gap: 8px; /* Space between icon and label */
  flex-wrap: wrap; /* Wrap label text if needed, keeping icon together */
}

.slider-header span {
  color: rgba(255, 255, 255, 0.5);
  font-variant-numeric: tabular-nums;
  transition: color 0.2s;
}

.divider {
  border: 0;
  height: 1px;
  background: rgba(255, 255, 255, 0.1);
  margin: 0;
}

/* --- DEVELOPER CREDIT --- */
.developer-credit {
  padding: 20px 24px;
  background: rgba(0, 0, 0, 0.2);
  border-top: 1px solid rgba(255, 255, 255, 0.05);
  font-size: 0.8rem;
  color: rgba(255, 255, 255, 0.6);
  text-align: center;
}

.developer-credit strong {
  color: white;
  font-weight: 500;
  letter-spacing: 0.5px;
}

/* --- PREMIUM LAYERED TOGGLE BUTTON --- */
.mute-toggle-wrapper {
  /* bg-gradient-to-b from-stone-300/40 to-transparent p-[4px] rounded-[16px] */
  background: linear-gradient(to bottom, rgba(214, 211, 209, 0.4), transparent);
  padding: 4px;
  border-radius: 16px;
  display: inline-flex;
}

.mute-toggle-btn {
  /* p-[4px] rounded-[12px] bg-gradient-to-b from-white to-stone-200/40 shadow-[0_1px_3px_...] */
  padding: 4px;
  border-radius: 12px;
  background: linear-gradient(to bottom, #ffffff, rgba(231, 229, 228, 0.4));
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.5);
  border: none;
  cursor: pointer;
  outline: none;
  transition: transform 0.1s ease, box-shadow 0.1s ease;
}

/* active:shadow-[0_0px_1px_...] active:scale-[0.995] */
.mute-toggle-btn:active {
  box-shadow: 0 0px 1px rgba(0, 0, 0, 0.5);
  transform: scale(0.95); /* Exaggerated slightly from 0.995 so it feels "clicky" */
}

.mute-toggle-inner {
  /* bg-gradient-to-b from-stone-200/40 to-white/80 rounded-[8px] px-2 py-2 */
  background: linear-gradient(to bottom, rgba(231, 229, 228, 0.4), rgba(255, 255, 255, 0.8));
  border-radius: 8px;
  padding: 4px 10px; /* Adjusted slightly from py-2 px-2 to look better as a label */
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 60px; /* Keeps the button from jumping in size between 'Muted' and '100%' */
}

.mute-toggle-inner span {
  /* font-semibold */
  font-weight: 600;
  font-size: 0.8rem;
  color: #333; /* Dark text to contrast the white/stone background */
  transition: color 0.2s ease;
}

/* Updated alert text color to look good on the light background */
.text-muted-alert {
  color: #dc2626 !important; /* A slightly darker red */
}

/* --- iOS STYLE HORIZONTAL SLIDER --- */
.ios-volume-wrapper {
  position: relative;
  width: 100%;
  margin-top: 8px;
}

/* New CSS for the floating symbols */
.ios-overlay-symbol {
  position: absolute;
  top: 50%;
  transform: translateY(-60%);
  font-size: 1.25rem;
  font-weight: 500;
  color: #fff;
  z-index: 10;
  pointer-events: none; 
  mix-blend-mode: difference; 
}

.ios-overlay-symbol.minus {
  left: 14px;
}

.ios-overlay-symbol.plus {
  right: 14px;
}

/* 1. The Main Input Container */
.ios-slider {
  -webkit-appearance: none;
  appearance: none; /* Kills standard browser styling */
  width: 100%;
  height: 36px;
  border-radius: 12px;
  outline: none;
  margin: 0;
  cursor: pointer;
  box-shadow: inset 0 1px 3px rgba(0, 0, 0, 0.2);
  /* The inline style handles the background gradient */
}

/* 2. WebKit (Chrome/Safari/Edge) Track */
.ios-slider::-webkit-slider-runnable-track {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 100%;
  background: transparent !important; /* Forces the track to be invisible */
  border: none;
}

/* 3. WebKit (Chrome/Safari/Edge) Thumb */
.ios-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  height: 36px;
  width: 10px; /* Needs a slight width to be draggable */
  background: transparent !important; /* Forces invisible thumb */
  border: none !important;
  box-shadow: none !important; /* Removes the ghost ring */
}

/* 4. Firefox Track */
.ios-slider::-moz-range-track {
  width: 100%;
  height: 100%;
  background: transparent !important; 
  border: none;
}

/* 5. Firefox Hidden Progress Line (This is often the culprit!) */
.ios-slider::-moz-range-progress {
  background: transparent !important;
  border: none;
}

/* 6. Firefox Thumb */
.ios-slider::-moz-range-thumb {
  height: 36px;
  width: 10px; 
  background: transparent !important;
  border: none !important;
  box-shadow: none !important; /* Removes the ghost ring */
}

/* 7. Focus States (Removes blue outlines when clicked) */
.ios-slider:focus {
  outline: none;
}
.ios-slider:focus::-webkit-slider-runnable-track {
  background: transparent;
}

/* --- PREMIUM PROGRESS SLIDER (Spotify/Apple Music Style) --- */
.premium-player-progress {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-top: 12px;
}

.progress-track-container {
  position: relative;
  height: 16px; /* Invisible larger touch target so it's easy to grab */
  display: flex;
  align-items: center;
  cursor: pointer;
}

.premium-progress-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 4px; /* Ultra-thin sleek line */
  border-radius: 2px;
  outline: none;
  margin: 0;
  transition: height 0.2s ease;
}

/* 1. Track expands slightly when the user hovers over the container */
.progress-track-container:hover .premium-progress-slider {
  height: 6px; 
}

/* 2. Reveal the dragging thumb ONLY when hovering over the container */
.progress-track-container:hover .premium-progress-slider::-webkit-slider-thumb {
  transform: scale(1);
}
.progress-track-container:hover .premium-progress-slider::-moz-range-thumb {
  transform: scale(1);
}

/* WebKit Thumb Styling */
.premium-progress-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: #ffffff;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.5);
  cursor: pointer;
  transform: scale(0); /* Hidden by default */
  /* Bouncy spring animation */
  transition: transform 0.2s cubic-bezier(0.175, 0.885, 0.32, 1.275); 
}

/* Firefox Thumb Styling */
.premium-progress-slider::-moz-range-thumb {
  width: 12px;
  height: 12px;
  border: none;
  border-radius: 50%;
  background: #ffffff;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.5);
  cursor: pointer;
  transform: scale(0); 
  transition: transform 0.2s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

/* Absolute Reset for Native Tracks */
.premium-progress-slider::-webkit-slider-runnable-track {
  -webkit-appearance: none;
  background: transparent !important;
  border: none;
}
.premium-progress-slider::-moz-range-track {
  background: transparent !important;
  border: none;
}
.premium-progress-slider::-moz-range-progress {
  background: transparent !important;
  border: none;
}

/* Time Footer Layout */
.time-footer {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: rgba(255, 255, 255, 0.5);
  font-variant-numeric: tabular-nums;
  font-weight: 500;
  letter-spacing: 0.5px;
}

/* --- ENV PILL SLIDERS --- */
.env-pill-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 20px; /* Slightly thinner than volume for visual hierarchy */
  border-radius: 10px;
  outline: none;
  margin: 8px 0 0 0;
  cursor: pointer;
  box-shadow: inset 0 1px 2px rgba(0, 0, 0, 0.3);
  transition: opacity 0.2s;
}

/* Chrome/Safari Thumb */
.env-pill-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 0;
  height: 0;
  background: transparent;
  border: none;
}

/* Firefox Thumb */
.env-pill-slider::-moz-range-thumb {
  width: 0;
  height: 0;
  background: transparent;
  border: none;
}

/* Reset Tracks */
.env-pill-slider::-webkit-slider-runnable-track {
  background: transparent !important;
  border: none;
}
.env-pill-slider::-moz-range-track {
  background: transparent !important;
  border: none;
}

.preset-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 8px;
  background: rgba(255, 255, 255, 0.05);
  padding: 6px;
  border-radius: 12px;
}

.preset-btn {
  background: transparent;
  border: none;
  color: rgba(255, 255, 255, 0.6);
  padding: 8px;
  border-radius: 8px;
  font-size: 0.8rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
}

.preset-btn.active {
  background: white;
  color: black;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.4);
}

.preset-btn:hover:not(.active) {
  background: rgba(255, 255, 255, 0.1);
  color: white;
}

.preset-btn.active .kbd-hint {
  color: rgba(0, 0, 0, 0.8);
  background: rgba(0, 0, 0, 0.05); 
  border-color: rgba(0, 0, 0, 0.2); 
  box-shadow: 0 1px 0 rgba(255, 255, 255, 0.5); 
}

.header-with-tag {
  display: flex;
  align-items: center;
  justify-content: space-between; 
  width: 100%;
}

.beta-badge {
  font-size: 0.6rem;
  font-weight: 900;
  letter-spacing: 0.05em;
  background: rgba(255, 255, 255, 0.1); 
  color: rgba(255, 255, 255, 0.9);
  padding: 2px 6px;
  border-radius: 4px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  text-transform: uppercase;
  margin-left: 8px;
  line-height: 1;
  animation: pulse-subtle 2s infinite;
}


@keyframes pulse-subtle {
  0% { opacity: 0.6; }
  50% { opacity: 1; }
  100% { opacity: 0.6; }
}

/* Reusing the select-wrapper logic for consistency */
.select-wrapper {
  position: relative;
  width: 100%;
  margin-top: 8px;
}

.select-wrapper select {
  width: 100%;
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 8px;
  color: white;
  padding: 10px 15px;
  appearance: none;
  font-size: 0.9rem;
  cursor: pointer;
  outline: none;
  transition: all 0.3s ease;
}

.select-wrapper select:focus {
  background: rgba(255, 255, 255, 0.15);
  border-color: rgba(255, 255, 255, 0.4);
}

.select-wrapper select option {
  background: #2a2a2a; /* Dark background for dropdown options */
  color: white;
}

.select-arrow {
  position: absolute;
  right: 15px;
  top: 50%;
  transform: translateY(-50%);
  pointer-events: none;
  opacity: 0.6;
  font-size: 0.8rem;
}
</style>