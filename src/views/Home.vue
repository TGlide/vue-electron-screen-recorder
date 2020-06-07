<template>
  <div class="home">
    <section class="section">
      <div class="container is-fluid">
        <h1 class="title has-text-centered">
          ⚡ VuElectron Screen Recorder ⚡
        </h1>

        <div class="video-container">
          <video ref="videoSource"></video>
        </div>

        <div class="record-buttons">
          <button
            class="button"
            :class="{ 'is-primary': !recording, 'is-danger': recording }"
            :disabled="selectedVideo === undefined"
            @click="handleStart"
          >
            <span v-if="!recording">Start</span>
            <span v-else>Recording</span>
          </button>
          <button
            class="button is-warning"
            @click="handleStop"
            :disabled="selectedVideo === undefined"
          >
            Stop
          </button>
        </div>

        <hr />

        <div id="video-sources">
          <b-dropdown aria-role="list">
            <button
              class="button is-text"
              slot="trigger"
              @click="getVideoSources"
            >
              <span v-if="selectedVideo === undefined">Select a source</span>
              <span v-else>{{ selectedVideo.name }}</span>
              <!-- <b-icon :icon="active ? 'chevron-up' : 'chevron-down'"></b-icon> -->
            </button>

            <b-dropdown-item
              aria-role="listitem"
              v-for="source of videoSources"
              :key="source.id"
              @click="() => selectSource(source)"
              >{{ source.name }}</b-dropdown-item
            >
          </b-dropdown>
        </div>
      </div>
    </section>
  </div>
</template>

<script>
import { desktopCapturer, remote } from 'electron';

const fs = require('fs');

const { dialog } = remote;

export default {
  name: 'Home',
  data() {
    return {
      videoSources: [],
      selectedVideo: undefined,
      mediaRecorder: undefined,
      recordedChunks: [],
      recording: false
    };
  },
  methods: {
    async getVideoSources() {
      this.videoSources = await desktopCapturer.getSources({
        types: ['window', 'screen']
      });
    },

    async selectSource(source) {
      this.selectedVideo = source;

      const constraints = {
        audio: false,
        video: {
          mandatory: {
            chromeMediaSource: 'desktop',
            chromeMediaSourceId: source.id
          }
        }
      };

      const stream = await navigator.mediaDevices.getUserMedia(constraints);

      // Preview Source
      this.$refs['videoSource'].srcObject = stream;
      this.$refs['videoSource'].play();

      // Create the Media Recorder
      const options = { mimeType: 'video/webm; codecs=vp9' };
      this.mediaRecorder = new MediaRecorder(stream, options);

      // Event Handlers
      this.mediaRecorder.ondataavailable = this.handleDataAvailable;
      this.mediaRecorder.onstop = this.handleMediaStop;
    },

    handleStart() {
      if (this.recording || this.selectedVideo === undefined) return;
      this.mediaRecorder.start();
      this.recording = true;
    },

    handleStop() {
      if (!this.recording) return;
      this.mediaRecorder.stop();
      this.recording = false;
    },

    // Captures all recorded chunks
    handleDataAvailable(e) {
      this.recordedChunks.push(e.data);
    },

    // Saves the video file on stop
    async handleMediaStop() {
      const blob = new Blob(this.recordedChunks, {
        type: 'video/webm; codecs=vp9'
      });

      const buffer = Buffer.from(await blob.arrayBuffer());

      const { filePath } = await dialog.showSaveDialog({
        buttonLabel: 'Save video',
        defaultPath: `vid-${Date.now()}.webm`
      });

      if (filePath) {
        fs.writeFileSync(filePath, buffer);
      }

      this.recordedChunks = [];
    }
  }
};
</script>

<style lang="scss" scoped>
@import '@/assets/styles/theme.scss';

.video-container {
  display: flex;
  justify-content: center;

  max-height: 85vh;

  margin: 1rem auto;
  padding: 0.5rem;
  border: 0.25rem rgba($black, 0.75) dashed;
  border-radius: 1rem;
}

.record-buttons {
  margin: 0 auto;

  display: flex;
  justify-content: center;

  .button {
    margin: 0 0.1rem;
  }
}

#video-sources {
  margin: 0 auto;

  display: flex;
  justify-content: center;
}
</style>
