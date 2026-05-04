<template>
  <Transition name="fade">
    <div v-if="isOpen" class="modal-overlay" @click.self="closeModal">
      <div class="modal-card" :class="{ 'processing': isProcessing }">
        
        <div class="modal-header">
          <h2 class="header-with-badge">
            <span><font-awesome-icon icon="video" /> Export Video Options</span>
            <span class="beta-badge">BETA</span>
          </h2>
          <button v-if="!isProcessing" class="close-btn" @click="closeModal">
            <font-awesome-icon icon="xmark" />
          </button>
        </div>

        <div class="modal-body">
          <div v-if="!isProcessing" class="config-view">
            
            <div class="tier-toggle">
              <button 
                :class="{ active: tier === 'free' }" 
                @click="tier = 'free'"
              >Free</button>
              <button 
                :class="{ 'active-premium': tier === 'premium' }" 
                @click="tier = 'premium'"
              >
                <font-awesome-icon icon="star" /> Premium
              </button>
            </div>

            <div class="settings-grid">
              <!-- Export Format -->
              <div class="setting-item">
                <label>Export Format</label>
                <div class="chip-group">
                  <button 
                    :class="{ active: format === 'webm' }"
                    @click="format = 'webm'"
                  >WebM</button>
                  <button 
                    :class="{ active: format === 'mp4' }"
                    :disabled="tier === 'free'"
                    @click="format = 'mp4'"
                  >MP4</button>
                </div>
              </div>

              <!-- Resolution -->
              <div class="setting-item">
                <label>Resolution</label>
                <div class="select-wrapper">
                  <select v-model="selectedRes" :disabled="tier === 'free'">
                    <option v-for="res in availableResolutions" :key="res.label" :value="res">
                      {{ res.label }} — {{ res.w }}x{{ res.h }} {{ tier === 'free' && res.label !== '720p' ? '🔒' : '' }}
                    </option>
                  </select>
                  <font-awesome-icon icon="chevron-down" class="select-arrow" />
                </div>
              </div>

              <!-- FPS -->
              <div class="setting-item">
                <label>Frame Rate (FPS)</label>
                <div class="chip-group">
                  <button 
                    v-for="f in [30, 60]" 
                    :key="f"
                    :class="{ active: fps === f }"
                    :disabled="tier === 'free' && f === 60"
                    @click="fps = f"
                  >{{ f }} FPS</button>
                </div>
              </div>

              <!-- Conditional Duration Section -->
              <div class="setting-item">
                <label>Duration</label>
                
                <!-- Option A: Track is long enough to have a "Short" version -->
                <div v-if="maxAudioDuration > 51" class="chip-group">
                  <button 
                    :class="{ active: duration === 51 }"
                    @click="duration = 51"
                  >Short (51s)</button>
                  <button 
                    :class="{ active: duration > 51 }"
                    :disabled="tier === 'free'"
                    @click="duration = Math.floor(maxAudioDuration)"
                  >Full ({{ Math.floor(maxAudioDuration) }}s) </button>
                </div>

                <!-- Option B: Track is naturally short (e.g. 30s) -->
                <div v-else class="full-length-badge">
                  <font-awesome-icon icon="circle-check" />
                  Full Track ({{ Math.floor(maxAudioDuration) }}s)
                </div>
              </div>
            </div>

            <div class="warning-box">
                <font-awesome-icon icon="triangle-exclamation" class="warning-icon" />
                <div class="warning-content">
                    <strong>Technical Note:</strong> Keep this tab active to prevent pausing. Video encoding is hardware intensive.
                </div>
            </div>

            <div class="start-btn-container">
              <button class="start-btn" @click="handleStart">
                Start Rendering
              </button>
              <span class="beta-disclaimer">Experimental browser-based encoding</span>
            </div>
          </div>

          <!-- Progress View -->
          <div v-else class="progress-view">
            <div class="loader-spinner"></div>
            <h3>Creating Cinematic Sequence...</h3>
            
            <div class="progress-container">
              <div class="progress-bar">
                <div class="progress-fill" :style="{ width: progress + '%' }"></div>
              </div>
              <div class="progress-stats">
                <span>{{ Math.round(progress) }}%</span>
                <span>{{ estimatedTime }} remaining</span>
              </div>
            </div>
            
            <p class="status-msg">{{ statusMessage }}</p>

            <button class="cancel-btn" @click="$emit('cancel')">
                <font-awesome-icon icon="xmark" /> Cancel Process
            </button>
          </div>
        </div>
      </div>
    </div>
  </Transition>
</template>

<script setup>
import { ref, computed, watch } from 'vue';

const props = defineProps({
  isOpen: Boolean,
  isProcessing: Boolean,
  progress: Number,
  maxAudioDuration: Number,
  statusMessage: String
});

const emit = defineEmits(['close', 'start-render', 'cancel']);

const tier = ref('free');
const format = ref('webm');
const fps = ref(30);
const duration = ref(51);
const selectedRes = ref({ label: '720p', w: 1280, h: 720 });
const startTime = ref(null);

const availableResolutions = [
  { label: '720p', w: 1280, h: 720 },
  { label: '1080p', w: 1920, h: 1080 },
  { label: '1440p', w: 2560, h: 1440 },
  { label: '4K', w: 3180, h: 2160 },
];

// Logic to force Free limits
watch(tier, (newTier) => {
  if (newTier === 'free') {
    selectedRes.value = availableResolutions[0];
    fps.value = 30;
    format.value = 'webm';
    // Only force 51s if the track is actually longer than 51s
    if (props.maxAudioDuration > 51) {
      duration.value = 51;
    } else {
      duration.value = props.maxAudioDuration;
    }
  }
});

watch(() => props.maxAudioDuration, (newMax) => {
  if (newMax <= 51 || (tier.value === 'premium' && duration.value > 51)) {
    duration.value = newMax;
  } else if (tier.value === 'free') {
    duration.value = 51;
  }
}, { immediate: true });

watch(() => props.isProcessing, (newVal) => {
  if (newVal) {
    startTime.value = Date.now();
  } else {
    startTime.value = null;
  }
});

const formatTimeRemaining = (seconds) => {
  if (!isFinite(seconds) || seconds <= 0) return "Calculating...";
  
  const h = Math.floor(seconds / 3600);
  const m = Math.floor((seconds % 3600) / 60);
  const s = Math.floor(seconds % 60);

  if (h > 0) return `${h}h ${m}m`;
  if (m > 0) return `${m}m ${s}s`;
  return `${s}s`;
};

const estimatedTime = computed(() => {
  // 1. Initial State
  if (!props.isProcessing || props.progress <= 1) return "Calculating...";
  
  // 2. Finalizing State
  if (props.progress >= 98) return "Almost done...";

  // 3. Dynamic Calculation
  if (startTime.value) {
    const elapsedMs = Date.now() - startTime.value;
    // Calculate total estimated time based on current velocity
    const totalEstimatedMs = (elapsedMs / props.progress) * 100;
    const remainingMs = totalEstimatedMs - elapsedMs;
    
    return formatTimeRemaining(remainingMs / 1000);
  }

  return "This is taking longer than expected...";
});

const closeModal = () => {
  if (!props.isProcessing) emit('close');
};

const handleStart = () => {
  emit('start-render', {
    width: selectedRes.value.w,
    height: selectedRes.value.h,
    fps: fps.value,
    duration: Math.min(duration.value, props.maxAudioDuration),
    format: format.value
  });
};
</script>

<style scoped>

.full-length-badge {
  display: flex;
  align-items: center;
  gap: 8px;
  background: rgba(255, 255, 255, 0.1);
  color: #4ade80;
  padding: 10px 15px;
  border-radius: 8px;
  font-size: 0.9rem;
  font-weight: 600;
  border: 1px solid rgba(74, 222, 128, 0.2);
}

.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.85);
  backdrop-filter: blur(12px);
  z-index: 3000;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
}

.modal-card {
  background: #1a1a1a;
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 28px;
  width: 100%;
  max-width: 460px;
  box-shadow: 0 30px 60px rgba(0,0,0,0.6);
}

.modal-header {
  padding: 20px 24px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.05);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.modal-header h2 { font-size: 1.1rem; margin: 0; color: #fff; font-weight: 500; }

.modal-body { padding: 24px; }

.tier-toggle {
  display: flex;
  background: rgba(255,255,255,0.05);
  padding: 4px;
  border-radius: 14px;
  margin-bottom: 32px;
}

.tier-toggle button {
  flex: 1;
  padding: 12px;
  border: none;
  background: transparent;
  color: rgba(255,255,255,0.4);
  border-radius: 10px;
  cursor: pointer;
  font-weight: 600;
  transition: 0.2s;
}

.tier-toggle button.active {
  background: rgba(255,255,255,0.1);
  color: #fff;
}

.active-premium {
  background: linear-gradient(135deg, #fde047 0%, #eab308 100%) !important;
  color: #451a03 !important;
  box-shadow: 0 4px 15px rgba(234, 179, 8, 0.3);
}

.select-wrapper {
  position: relative;
  display: flex;
  align-items: center;
}

select {
  width: 100%;
  background: rgba(255,255,255,0.05);
  border: 1px solid rgba(255,255,255,0.1);
  padding: 14px;
  border-radius: 12px;
  color: #fff;
  appearance: none;
  font-size: 0.95rem;
  cursor: pointer;
  transition: border-color 0.2s;
}

select:focus { border-color: rgba(255,255,255,0.3); outline: none; }
select:disabled { opacity: 0.5; cursor: not-allowed; }

.select-arrow {
  position: absolute;
  right: 16px;
  pointer-events: none;
  font-size: 0.8rem;
  color: rgba(255,255,255,0.3);
}

.chip-group {
  display: flex;
  gap: 12px;
}

.chip-group button {
  flex: 1;
  padding: 12px;
  background: rgba(255,255,255,0.05);
  border: 1px solid rgba(255,255,255,0.1);
  border-radius: 12px;
  color: #fff;
  cursor: pointer;
  font-size: 0.9rem;
  transition: 0.2s;
}

.chip-group button.active {
  background: #fff;
  color: #000;
  border-color: #fff;
}

.chip-group button:disabled { opacity: 0.4; cursor: not-allowed; }

.settings-grid { display: flex; flex-direction: column; gap: 24px; margin-bottom: 32px; }
.setting-item { display: flex; flex-direction: column; gap: 10px; }
.setting-item label { font-size: 0.75rem; color: rgba(255,255,255,0.4); text-transform: uppercase; letter-spacing: 1.5px; font-weight: 600; }

.warning-box {
  background: rgba(255, 255, 255, 0.03);
  border-left: 3px solid #eab308;
  border-top: 1px solid rgba(255, 255, 255, 0.05);
  border-right: 1px solid rgba(255, 255, 255, 0.05);
  border-bottom: 1px solid rgba(255, 255, 255, 0.05);
  padding: 12px 14px;
  border-radius: 12px;
  margin-bottom: 24px;
  display: flex;
  align-items: flex-start;
  gap: 12px;
  color: rgba(255, 255, 255, 0.7);
  font-size: 0.78rem;
  line-height: 1.4;
}

.warning-icon { color: #eab308; font-size: 0.9rem; margin-top: 2px; }
.warning-content strong { color: #fde047; font-weight: 600; text-transform: uppercase; letter-spacing: 0.5px; margin-right: 4px; }

.start-btn {
  width: 100%;
  padding: 18px;
  background: #fff;
  color: #000;
  border: none;
  border-radius: 16px;
  font-weight: 700;
  font-size: 1rem;
  cursor: pointer;
  transition: transform 0.1s;
}

.start-btn:active { transform: scale(0.98); }
.close-btn { background: none; border: none; color: rgba(255,255,255,0.3); cursor: pointer; font-size: 1.2rem; }
.progress-view { text-align: center; padding: 20px 0; }
.loader-spinner {
  width: 40px; height: 40px; border: 3px solid rgba(255,255,255,0.1); border-top-color: #fff;
  border-radius: 50%; margin: 0 auto 20px; animation: spin 1s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }
.progress-bar { height: 6px; background: rgba(255,255,255,0.1); border-radius: 10px; margin: 24px 0 12px; overflow: hidden; }
.progress-fill { height: 100%; background: #fff; transition: width 0.4s ease; }
.progress-stats { display: flex; justify-content: space-between; font-size: 0.8rem; color: rgba(255,255,255,0.4); }
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }

/* Forces the dropdown list to follow the dark theme */
select option {
  background-color: #1a1a1a; /* Matches modal background */
  color: #ffffff;            /* Forces white text */
  padding: 10px;             /* Note: some browsers ignore padding on options */
}

/* Optional: Improves the look on hover in some browsers */
select option:hover,
select option:focus,
select option:active {
  background-color: #333333 !important;
}

/* Fix for Firefox/Mobile where options might still be stubborn */
select:-moz-focusring {
  color: transparent;
  text-shadow: 0 0 0 #fff;
}

/* --- PROGRESS VIEW CONTRAST FIX --- */
.progress-view { 
  text-align: center; 
  padding: 30px 0; 
}

.progress-view h3 {
  color: #ffffff; /* Pure white for the main title */
  font-size: 1.15rem;
  font-weight: 500;
  margin-bottom: 8px;
}

.progress-stats { 
  display: flex; 
  justify-content: space-between; 
  margin-top: 10px;
}

.progress-stats span {
  /* Increased opacity from 0.4 to 0.85 for readability */
  color: rgba(255, 255, 255, 0.85); 
  font-size: 0.85rem;
  font-weight: 500;
}

.status-msg {
  /* Slightly muted but clearly visible */
  color: rgba(255, 255, 255, 0.6); 
  font-size: 0.9rem;
  margin-top: 24px;
  font-family: 'Inter', monospace; /* Monospace looks great for "Frame X of Y" */
}

.loader-spinner {
  width: 44px; 
  height: 44px; 
  border: 3px solid rgba(255, 255, 255, 0.1); 
  border-top-color: #ffffff; /* Bright white spinner head */
  border-radius: 50%; 
  margin: 0 auto 24px; 
  animation: spin 1s linear infinite;
}

.progress-bar { 
  height: 8px; 
  background: rgba(255, 255, 255, 0.1); 
  border-radius: 10px; 
  margin-top: 24px; 
  overflow: hidden; 
}

.progress-fill { 
  height: 100%; 
  background: #ffffff; /* Bright white bar */
  box-shadow: 0 0 15px rgba(255, 255, 255, 0.3); /* Subtle glow to show it's active */
  transition: width 0.4s ease; 
}

.cancel-btn {
  margin-top: 32px;
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  color: rgba(255, 255, 255, 0.5);
  padding: 10px 20px;
  border-radius: 10px;
  font-size: 0.8rem;
  cursor: pointer;
  transition: all 0.2s;
}

.cancel-btn:hover {
  background: rgba(220, 38, 38, 0.15); /* Subtle red tint */
  border-color: rgba(220, 38, 38, 0.3);
  color: #fca5a5;
}

/* Layout for the Header */
.header-with-badge {
  display: flex;
  align-items: center;
  gap: 10px;
}

/* The BETA Badge Style */
.beta-badge {
  font-size: 0.65rem;
  font-weight: 800;
  letter-spacing: 0.05em;
  background: rgba(255, 255, 255, 0.15); /* Glass effect */
  color: #fff;
  padding: 2px 8px;
  border-radius: 6px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  text-transform: uppercase;
  line-height: 1.4;
}

/* Start Button disclaimer */
.start-btn-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  margin-top: 10px;
}

.beta-disclaimer {
  font-size: 0.75rem;
  color: rgba(255, 255, 255, 0.5);
  font-style: italic;
}

/* Ensure the modal-card header is flex-start to accommodate the badge */
.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
</style>