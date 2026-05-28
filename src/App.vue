<template>
  <div class="app">
    <header>
      <div class="logo">
        <svg viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
          <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm-2 14.5v-9l6 4.5-6 4.5z" fill="#ff0000"/>
        </svg>
        <span>TubeSave</span>
      </div>
      <p class="tagline">Download YouTube music &amp; videos</p>
    </header>

    <main>
      <div class="search-card">
        <div class="input-row">
          <input
            v-model="url"
            type="text"
            placeholder="Paste YouTube URL here..."
            @paste="onPaste"
            @keydown.enter="fetchInfo"
            :disabled="loading || downloading"
          />
          <button class="btn-fetch" @click="fetchInfo" :disabled="!url || loading || downloading">
            <span v-if="loading" class="spinner"></span>
            <span v-else>Search</span>
          </button>
        </div>
        <p v-if="error" class="error-msg">{{ error }}</p>
      </div>

      <transition name="slide-up">
        <div v-if="videoInfo" class="info-card">
          <img :src="videoInfo.thumbnail" :alt="videoInfo.title" class="thumbnail" />
          <div class="info-body">
            <h2 class="video-title">{{ videoInfo.title }}</h2>
            <p class="meta">
              <span>{{ videoInfo.uploader }}</span>
              <span v-if="videoInfo.duration"> · {{ formatDuration(videoInfo.duration) }}</span>
            </p>

            <div class="format-toggle">
              <button :class="['fmt-btn', { active: format === 'audio' }]" @click="format = 'audio'">
                🎵 MP3 Audio
              </button>
              <button :class="['fmt-btn', { active: format === 'video' }]" @click="format = 'video'">
                🎬 MP4 Video
              </button>
            </div>

            <div v-if="downloading" class="progress-section">
              <div class="progress-bar">
                <div class="progress-fill" :style="{ width: progress + '%' }"></div>
              </div>
              <p class="progress-label">
                {{ progress < 100 ? `Downloading... ${progress.toFixed(0)}%` : 'Finalizing...' }}
              </p>
            </div>

            <button v-else class="btn-download" @click="startDownload" :disabled="downloading">
              Download {{ format === 'audio' ? 'MP3' : 'MP4' }}
            </button>
          </div>
        </div>
      </transition>
    </main>

    <footer>
      <p>TubeSave · For personal use only · Respect copyright</p>
    </footer>
  </div>
</template>

<script setup>
import { ref } from 'vue';

const API = import.meta.env.VITE_API_URL;

const url = ref('');
const format = ref('audio');
const loading = ref(false);
const downloading = ref(false);
const progress = ref(0);
const error = ref('');
const videoInfo = ref(null);

function onPaste(e) {
  const text = e.clipboardData?.getData('text') || '';
  if (text.includes('youtube.com') || text.includes('youtu.be')) {
    url.value = text;
    setTimeout(fetchInfo, 100);
  }
}

async function fetchInfo() {
  if (!url.value) return;
  error.value = '';
  videoInfo.value = null;
  loading.value = true;

  try {
    const res = await fetch(`${API}/api/info`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ url: url.value }),
    });
    const data = await res.json();
    if (!res.ok) throw new Error(data.error || 'Failed to fetch info');
    videoInfo.value = data;
  } catch (err) {
    error.value = err.message;
  } finally {
    loading.value = false;
  }
}

function startDownload() {
  downloading.value = true;
  progress.value = 0;

  const params = new URLSearchParams({ url: url.value, format: format.value });
  const sse = new EventSource(`${API}/api/progress?${params}`);

  sse.onmessage = (e) => {
    const msg = JSON.parse(e.data);
    if (msg.type === 'progress') {
      progress.value = msg.percent;
    } else if (msg.type === 'done') {
      progress.value = 100;
      sse.close();
      const a = document.createElement('a');
      a.href = `${API}/api/file/${encodeURIComponent(msg.file)}`;
      a.download = msg.filename;
      a.click();
      setTimeout(() => { downloading.value = false; }, 1000);
    } else if (msg.type === 'error') {
      error.value = msg.message;
      sse.close();
      downloading.value = false;
    }
  };

  sse.onerror = () => {
    error.value = 'Connection to server lost.';
    sse.close();
    downloading.value = false;
  };
}

function formatDuration(secs) {
  if (!secs) return '';
  const m = Math.floor(secs / 60);
  const s = secs % 60;
  return `${m}:${String(s).padStart(2, '0')}`;
}
</script>

<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: #0f0f0f;
  color: #f1f1f1;
  min-height: 100vh;
}

.app {
  display: flex;
  flex-direction: column;
  min-height: 100vh;
  max-width: 680px;
  margin: 0 auto;
  padding: 0 16px;
}

header { padding: 48px 0 32px; text-align: center; }

.logo {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  margin-bottom: 8px;
}

.logo svg { width: 40px; height: 40px; }
.logo span { font-size: 2rem; font-weight: 700; letter-spacing: -0.5px; }
.tagline { color: #aaa; font-size: 0.95rem; }

main { flex: 1; }

.search-card {
  background: #1a1a1a;
  border: 1px solid #2a2a2a;
  border-radius: 14px;
  padding: 20px;
  margin-bottom: 20px;
}

.input-row { display: flex; gap: 10px; }

input {
  flex: 1;
  background: #111;
  border: 1.5px solid #333;
  border-radius: 8px;
  padding: 12px 16px;
  font-size: 0.95rem;
  color: #f1f1f1;
  outline: none;
  transition: border-color 0.2s;
}

input:focus { border-color: #ff4444; }
input::placeholder { color: #555; }
input:disabled { opacity: 0.5; }

.btn-fetch {
  background: #ff0000;
  color: #fff;
  border: none;
  border-radius: 8px;
  padding: 12px 24px;
  font-size: 0.95rem;
  font-weight: 600;
  cursor: pointer;
  min-width: 90px;
  transition: background 0.2s, opacity 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
}

.btn-fetch:hover:not(:disabled) { background: #cc0000; }
.btn-fetch:disabled { opacity: 0.5; cursor: not-allowed; }

.spinner {
  width: 18px; height: 18px;
  border: 2px solid rgba(255,255,255,0.3);
  border-top-color: #fff;
  border-radius: 50%;
  animation: spin 0.7s linear infinite;
  display: inline-block;
}

@keyframes spin { to { transform: rotate(360deg); } }

.error-msg { color: #ff6b6b; font-size: 0.85rem; margin-top: 10px; }

.info-card {
  background: #1a1a1a;
  border: 1px solid #2a2a2a;
  border-radius: 14px;
  overflow: hidden;
}

.thumbnail { width: 100%; max-height: 280px; object-fit: cover; display: block; }
.info-body { padding: 20px; }

.video-title { font-size: 1.05rem; font-weight: 600; line-height: 1.4; margin-bottom: 6px; }
.meta { color: #888; font-size: 0.85rem; margin-bottom: 20px; }

.format-toggle { display: flex; gap: 10px; margin-bottom: 20px; }

.fmt-btn {
  flex: 1;
  padding: 10px;
  border-radius: 8px;
  border: 1.5px solid #333;
  background: transparent;
  color: #aaa;
  font-size: 0.9rem;
  cursor: pointer;
  transition: all 0.2s;
}

.fmt-btn.active {
  border-color: #ff0000;
  background: rgba(255, 0, 0, 0.1);
  color: #ff4444;
  font-weight: 600;
}

.fmt-btn:hover:not(.active) { border-color: #555; color: #f1f1f1; }

.btn-download {
  width: 100%;
  background: #ff0000;
  color: #fff;
  border: none;
  border-radius: 10px;
  padding: 14px;
  font-size: 1rem;
  font-weight: 700;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-download:hover:not(:disabled) { background: #cc0000; }
.btn-download:disabled { opacity: 0.5; cursor: not-allowed; }

.progress-section { margin-top: 4px; }

.progress-bar {
  background: #2a2a2a;
  border-radius: 6px;
  height: 8px;
  overflow: hidden;
  margin-bottom: 8px;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #ff0000, #ff6666);
  border-radius: 6px;
  transition: width 0.3s ease;
}

.progress-label { color: #888; font-size: 0.85rem; text-align: center; }

footer { text-align: center; padding: 24px 0; color: #444; font-size: 0.8rem; }

.slide-up-enter-active { transition: all 0.3s ease; }
.slide-up-enter-from { opacity: 0; transform: translateY(16px); }
</style>
