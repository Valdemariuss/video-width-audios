<template>
  <div class="media-player">
    <!-- Загрузка файлов -->
    <div class="file-inputs">
      <div class="input-group">
        <label>Видео (MP4 без звука):</label>
        <input 
          type="file" 
          accept="video/mp4" 
          @change="handleVideoUpload"
        >
      </div>
      
      <div class="input-group">
        <label>Аудио дорожки (MP3):</label>
        <input 
          type="file" 
          accept="audio/mp3" 
          multiple
          @change="handleAudioUpload"
        >
      </div>
    </div>

    <!-- Видео элемент (без звука) -->
    <video 
      ref="videoPlayer"
      :src="videoUrl"
      @timeupdate="handleTimeUpdate"
      @seeked="handleSeek"
      @ended="handleVideoEnded"
      @error="handleVideoError"
      crossorigin="anonymous"
      muted
      playsinline
      v-if="videoUrl"
    ></video>
    
    <!-- Элементы управления -->
    <div class="controls" v-if="videoUrl">
      <button @click="togglePlay" class="play-button" :class="{ playing: isPlaying }">
        <span v-if="isPlaying">❚❚</span>
        <span v-else>▶</span>
      </button>
      
      <div class="time-controls">
        <span class="current-time">{{ formatTime(currentTime) }}</span>
        
        <input 
          type="range" 
          class="progress-bar"
          v-model="currentTime"
          :max="Math.floor(duration)"
          :style="{'--progress': (currentTime / duration) * 100 + '%'}"
          @input="handleSeek"
          @mousedown="isSeeking = true"
          @mouseup="isSeeking = false"
        >
        
        <span class="duration">{{ formatTime(duration) }}</span>
      </div>
      
      <button 
        @click="toggleMute" 
        class="mute-button"
        :title="isMuted ? 'Включить звук' : 'Выключить звук'"
      >
        {{ isMuted ? '🔇' : '🔊' }}
      </button>
      
      <input 
        type="range" 
        class="volume-slider"
        v-model="volume"
        min="0"
        max="1"
        step="0.01"
        @input="handleVolumeChange"
      >
    </div>
    
    <!-- Аудиодорожки -->
    <div class="audio-tracks" v-if="audioTracks.length > 0">
      <div v-for="(track, index) in audioTracks" :key="index" class="track">
        <label>
          <input 
            type="checkbox" 
            v-model="track.active"
            @change="() => toggleTrack(track)"
          >
          {{ track.name || `Дорожка ${index + 1}` }}
          <span class="volume-badge" v-if="track.active">🔊 {{ Math.round(volume * 100) }}%</span>
        </label>
        <audio 
          :ref="el => { if (el) audioPlayers[index] = el }"
          :src="track.url"
          crossorigin="anonymous"
          @ended="handleAudioEnded"
          @error="handleAudioError"
        ></audio>
      </div>
    </div>
    
    <!-- Сообщение об ошибке -->
    <div class="error-message" v-if="error">
      {{ error }}
      <button @click="error = ''">×</button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      videoUrl: null,
      audioTracks: [],
      audioPlayers: [],
      videoPlayer: null,
      isPlaying: false,
      currentTime: 0,
      duration: 0,
      isSeeking: false,
      lastUpdateTime: 0,
      isMuted: false,
      volume: 0.8,
      error: ''
    }
  },
  
  methods: {
    handleVideoUpload(event) {
      const file = event.target.files[0];
      if (!file) return;
      
      this.resetPlayer();
      this.videoUrl = URL.createObjectURL(file);
      this.$nextTick(() => {
        this.videoPlayer = this.$refs.videoPlayer;
        this.initPlayers();
      });
    },
    
    handleAudioUpload(event) {
      const files = Array.from(event.target.files);
      if (files.length === 0) return;
      
      this.audioTracks = files.map((file, index) => ({
        name: file.name.replace(/\.[^/.]+$/, ""),
        url: URL.createObjectURL(file),
        active: true,
        file: file
      }));
      
      this.$nextTick(() => {
        if (this.videoUrl) {
          this.initPlayers();
        }
      });
    },
    
    resetPlayer() {
      try {
        this.pauseAll();
        
        // Освобождаем ресурсы
        if (this.videoUrl) {
          URL.revokeObjectURL(this.videoUrl);
        }
        this.audioTracks.forEach(track => {
          if (track.url) {
            URL.revokeObjectURL(track.url);
          }
        });
        
        // Сбрасываем состояние
        this.videoUrl = null;
        this.audioTracks = [];
        this.audioPlayers = [];
        this.currentTime = 0;
        this.duration = 0;
        this.isPlaying = false;
        this.error = '';
      } catch (e) {
        console.error('Ошибка при сбросе плеера:', e);
      }
    },
    
    initPlayers() {
      if (!this.videoPlayer) return;
      
      // Убедимся, что видео без звука
      this.videoPlayer.muted = true;
      
      this.videoPlayer.addEventListener('loadedmetadata', () => {
        this.duration = this.videoPlayer.duration;
        
        // Установим громкость для всех аудио дорожек
        this.audioPlayers.forEach(player => {
          if (player) {
            player.volume = this.volume;
            player.addEventListener('loadedmetadata', () => {
              if (player.duration > this.duration) {
                console.warn('Аудио дорожка длиннее видео!');
              }
            });
          }
        });
      });
      
      this.videoPlayer.addEventListener('canplay', () => {
        this.playAll().catch(e => {
          this.error = 'Автовоспроизведение заблокировано. Нажмите Play для запуска.';
          console.log('Автовоспроизведение заблокировано:', e.message);
        });
      });
    },
    
    async playAll() {
      try {
        if (!this.videoPlayer) return;
        
        await this.videoPlayer.play().catch(e => {
          throw new Error('Ошибка воспроизведения видео: ' + e.message);
        });
        
        // Синхронизируем и запускаем активные аудиодорожки
        const audioPlayPromises = this.audioPlayers.map((player, index) => {
          if (player && this.audioTracks[index]?.active) {
            player.currentTime = this.videoPlayer.currentTime;
            return player.play().catch(e => {
              console.error('Ошибка воспроизведения аудио:', e);
              return Promise.resolve(); // Продолжаем, даже если одна дорожка не воспроизвелась
            });
          }
          return Promise.resolve();
        });
        
        await Promise.all(audioPlayPromises);
        this.isPlaying = true;
        this.error = '';
      } catch (error) {
        this.error = error.message;
        console.error('Ошибка воспроизведения:', error);
        this.pauseAll();
        throw error;
      }
    },
    
    pauseAll() {
      try {
        if (this.videoPlayer) {
          this.videoPlayer.pause();
        }
        
        this.audioPlayers.forEach(player => {
          if (player) {
            player.pause();
          }
        });
        
        this.isPlaying = false;
      } catch (e) {
        console.error('Ошибка при паузе:', e);
      }
    },
    
    syncAllPlayers() {
      if (!this.isPlaying || !this.videoPlayer || this.isSeeking) return;
      
      const videoTime = this.videoPlayer.currentTime;
      const now = performance.now();
      
      if (now - this.lastUpdateTime > 100) {
        this.audioPlayers.forEach((player, index) => {
          if (player && this.audioTracks[index]?.active) {
            // Остановка аудио, если видео закончилось
            if (videoTime >= this.duration) {
              player.pause();
              return;
            }
            
            if (Math.abs(player.currentTime - videoTime) > 0.1) {
              player.currentTime = videoTime;
            }
          }
        });
        
        this.lastUpdateTime = now;
      }
    },
    
    handleTimeUpdate() {
      if (this.videoPlayer) {
        this.currentTime = this.videoPlayer.currentTime;
        this.syncAllPlayers();
        
        // Остановка при достижении конца видео
        if (this.currentTime >= this.duration) {
          this.pauseAll();
        }
      }
    },
    
    handleVideoEnded() {
      this.pauseAll();
      this.currentTime = this.duration;
    },
    
    handleAudioEnded() {
      // Если какая-то аудиодорожка закончилась раньше видео
      if (this.isPlaying && this.videoPlayer && this.videoPlayer.currentTime < this.duration) {
        this.syncAllPlayers();
      }
    },
    
    handleVideoError() {
      this.error = 'Ошибка загрузки видео';
      console.error('Ошибка видео элемента:', this.videoPlayer.error);
    },
    
    handleAudioError(event) {
      this.error = 'Ошибка загрузки аудио дорожки';
      console.error('Ошибка аудио элемента:', event.target.error);
    },
    
    handleSeek() {
      if (this.isSeeking || !this.videoPlayer) return;
      
      this.isSeeking = true;
      const seekTime = parseFloat(this.currentTime);
      
      this.videoPlayer.currentTime = seekTime;
      
      this.audioPlayers.forEach((player, index) => {
        if (player && this.audioTracks[index]?.active) {
          player.currentTime = seekTime;
        }
      });
      
      setTimeout(() => {
        if (this.isPlaying) {
          this.playAll().catch(e => {
            this.error = 'Ошибка при перемотке: ' + e.message;
            console.error('Ошибка при перемотке:', e);
          });
        }
        this.isSeeking = false;
      }, 50);
    },
    
    toggleTrack(track) {
      if (!this.videoPlayer) return;
      
      const index = this.audioTracks.indexOf(track);
      const player = this.audioPlayers[index];
      
      if (!player) return;
      
      if (track.active) {
        player.currentTime = this.videoPlayer.currentTime;
        if (this.isPlaying && this.videoPlayer.currentTime < this.duration) {
          player.play().catch(e => {
            this.error = 'Ошибка воспроизведения аудио дорожки';
            console.error('Ошибка воспроизведения аудио:', e);
          });
        }
      } else {
        player.pause();
      }
    },
    
    togglePlay() {
      if (this.isPlaying) {
        this.pauseAll();
      } else {
        this.playAll().catch(e => {
          this.error = 'Ошибка воспроизведения: ' + e.message;
          console.error('Ошибка воспроизведения:', e);
        });
      }
    },
    
    toggleMute() {
      this.isMuted = !this.isMuted;
      this.audioPlayers.forEach(player => {
        if (player) {
          player.muted = this.isMuted;
        }
      });
    },
    
    handleVolumeChange() {
      this.audioPlayers.forEach(player => {
        if (player) {
          player.volume = this.volume;
        }
      });
      this.isMuted = this.volume === 0;
    },
    
    formatTime(seconds) {
      if (isNaN(seconds)) return "0:00";
      const minutes = Math.floor(seconds / 60);
      seconds = Math.floor(seconds % 60);
      return `${minutes}:${seconds < 10 ? '0' : ''}${seconds}`;
    }
  },
  
  beforeUnmount() {
    this.resetPlayer();
  }
}
</script>

<style>
.media-player {
  max-width: 800px;
  margin: 0 auto;
  font-family: Arial, sans-serif;
  position: relative;
}

.file-inputs {
  margin-bottom: 20px;
  padding: 15px;
  background: #f5f5f5;
  border-radius: 8px;
}

.input-group {
  margin-bottom: 10px;
}

.input-group label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

input[type="file"] {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  background: white;
}

video {
  width: 100%;
  background: #000;
  border-radius: 8px;
  margin-top: 10px;
  display: block;
}

.controls {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: rgba(0, 0, 0, 0.7);
  border-radius: 8px;
  color: white;
  margin-top: 5px;
}

.play-button {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: #ff5722;
  color: white;
  border: none;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 12px;
}

.play-button.playing {
  background: #4CAF50;
}

.time-controls {
  flex-grow: 1;
  display: flex;
  align-items: center;
  gap: 8px;
}

.progress-bar {
  flex-grow: 1;
  height: 4px;
  background: linear-gradient(to right, #ff5722 var(--progress), #555 var(--progress));
  border-radius: 2px;
  cursor: pointer;
  -webkit-appearance: none;
}

.progress-bar::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: white;
  cursor: pointer;
}

.current-time, .duration {
  font-size: 12px;
  min-width: 40px;
  text-align: center;
}

.mute-button {
  background: transparent;
  border: none;
  color: white;
  cursor: pointer;
  font-size: 16px;
  padding: 0 5px;
}

.volume-slider {
  width: 80px;
  height: 4px;
}

.audio-tracks {
  margin-top: 20px;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 10px;
  padding: 15px;
  background: #f5f5f5;
  border-radius: 8px;
}

.track {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px;
  background: white;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

.track label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  width: 100%;
  font-size: 14px;
}

.track input[type="checkbox"] {
  margin: 0;
}

.volume-badge {
  font-size: 12px;
  opacity: 0.7;
}

.error-message {
  margin-top: 10px;
  padding: 10px;
  background: #ffebee;
  color: #c62828;
  border-radius: 4px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.error-message button {
  background: none;
  border: none;
  font-size: 16px;
  cursor: pointer;
  color: #c62828;
}
</style>