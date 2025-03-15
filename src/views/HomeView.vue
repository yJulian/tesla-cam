<template>
  <div>
    <button id="main-button" v-if="!folderSelected" @click="selectFolder">Ordner auswählen</button>
    
    <div v-if="folderSelected" id="map" style="height: 100vh; width: 100vw;"></div>
    
    <div v-if="selectedFolder && selectedFolder.fragments.length > 0" class="video-grid">
      <div id="big" class="video-container big">
        <video ref="frontVideo" width="100%" height="100%" controls @play="playAll" @pause="pauseAll" @seeked="progress" @ended="ended">
          <source :src="selectedFragment('front')" type="video/mp4">
          Your browser does not support the video tag.
        </video>
      </div>
      <div class="video-container left" @click="swap('left')">
        <video ref="leftRepeaterVideo" width="100%" height="100%">
          <source :src="selectedFragment('left_repeater')" type="video/mp4">
          Your browser does not support the video tag.
        </video>
      </div>
      <div class="video-container right">
        <video ref="rightRepeaterVideo" width="100%" height="100%">
          <source :src="selectedFragment('right_repeater')" type="video/mp4">
          Your browser does not support the video tag.
        </video>
      </div>
      <div class="video-container mid">
        <video ref="backVideo" width="100%" height="100%">
          <source :src="selectedFragment('back')" type="video/mp4">
          Your browser does not support the video tag.
        </video>
      </div>
      <div class="controls">
        <button @click="previousFragment">Vorheriges Fragment</button>
        <button @click="nextFragment">Nächstes Fragment</button>
        <button @click="close">Close</button>
      </div>
      <div class="time">
        {{ currentTime }} | {{ activationTime }}
        <br>
        {{ selectedFolder.fragments[currentFragmentIndex].timestamp }}
      </div>
    </div>
  </div>
</template>

<script>
import L from 'leaflet';
import 'leaflet/dist/images/marker-icon-2x.png';
import 'leaflet/dist/leaflet.css';

const defaultIcon = L.icon({
  iconUrl: "./leaflet/marker-icon.png",
  shadowUrl: "./leaflet/marker-shadow.png",
});
L.Marker.prototype.options.icon = defaultIcon;

export default {
  data() {
    return {
      folders: [],
      folderSelected: false,
      selectedFolder: null,
      map: null,
      currentFragmentIndex: 0,
      videoTime: 0,
    };
  },
  computed: {
    currentTime() {
    
      try {
      // Berechne die Zeit des Videos in Millisekunden
      const videoTimeInMillis = this.videoTime * 1000;
      
      // Addiere die Basiszeit
      const baseDate = new Date(this.selectedFolder.eventData.timestamp);
      const combinedDate = new Date(baseDate.getTime() + videoTimeInMillis);
      // Hole Stunden, Minuten und Sekunden
      const hours = combinedDate.getHours();
      const minutes = combinedDate.getUTCMinutes();
      const seconds = combinedDate.getUTCSeconds();
      // Formatierte Zeit im Format hh:mm:ss
      return `${this.padToTwoDigits(hours)}:${this.padToTwoDigits(minutes)}:${this.padToTwoDigits(seconds)}`;
    
      } catch {
        return "00:00:00";
      }
      
    },
    activationTime() {
      try {     
      // Addiere die Basiszeit
      const baseDate = new Date(this.selectedFolder.eventData.timestamp);
      // Hole Stunden, Minuten und Sekunden
      const hours = baseDate.getHours();
      const minutes = baseDate.getUTCMinutes();
      const seconds = baseDate.getUTCSeconds();
      // Formatierte Zeit im Format hh:mm:ss
      return `${this.padToTwoDigits(hours)}:${this.padToTwoDigits(minutes)}:${this.padToTwoDigits(seconds)}`;
    
      } catch {
        return "00:00:00";
      }
    },
  },
  methods: {
    padToTwoDigits(number) {
      return number < 10 ? '0' + number : number;
    },
    updateTime() {
      if (this.$refs.frontVideo) {
        // Update die Videozeit in Sekunden
        this.videoTime = this.$refs.frontVideo.currentTime;
      }
    },
    startUpdatingTime() {
      // Setze ein Interval, um die Zeit kontinuierlich zu aktualisieren
      this.intervalId = setInterval(() => {
        this.updateTime();
      }, 1000); // Update jede Sekunde
    },
    stopUpdatingTime() {
      // Stoppe das Interval, wenn es nicht mehr benötigt wird
      clearInterval(this.intervalId);
    },
    async selectFolder() {
      try {
        const directoryHandle = await window.showDirectoryPicker();
        this.folders = []; // Clear previous data

        for await (const [name, handle] of directoryHandle.entries()) {
          if (handle.kind === 'directory') {
            this.processFolder(handle);
          }
        }

        this.folderSelected = true; // Show the map

        await this.$nextTick();
        this.initMap();
      } catch (err) {
        console.error('Fehler beim Zugriff auf das Dateisystem:', err);
      }
    },
    async processFolder(folderHandle) {
      let folderData = {
        name: folderHandle.name,
        eventData: null,
        fragments: [],
      };

      for await (const [name, handle] of folderHandle.entries()) {
        if (handle.kind === 'file' && name === 'event.json') {
          const file = await handle.getFile();
          const content = await file.text();
          folderData.eventData = JSON.parse(content);
        }

        if (handle.kind === 'file' && name.endsWith('.mp4')) {
          const match = name.match(/^(\d{4}-\d{2}-\d{2})_(\d{2}-\d{2}-\d{2})-(\w+)\.mp4$/);
          
          if (!match) {
            console.warn(`Unerwartetes Dateiformat für ${name}`);
            continue;
          }

          const [_, date, time, camera] = match;
          folderData.fragments.push({
            url: URL.createObjectURL(await handle.getFile()),
            camera: camera, 
            timestamp: new Date(`${date}T${time.replace(/-/g, ':')}`),
          });
        }
      }

      folderData.fragments.sort((a, b) => a.timestamp - b.timestamp);

      if (folderData.eventData) {
        this.folders.push(folderData);
        this.addToMap(folderData);
      }
    },
    initMap() {
      if (!this.map) {
        this.map = L.map('map').setView([51.1657, 10.4515], 6); // Default view over Germany
        L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
          attribution: '&copy; OpenStreetMap contributors',
        }).addTo(this.map);

        this.folders.forEach(folder => {
          if (folder.eventData) {
            const { est_lat, est_lon } = folder.eventData;
            const marker = L.marker([est_lat, est_lon])
              .addTo(this.map)
              .bindPopup(`<b>${folder.name}</b><br>${folder.eventData.city}`);

            marker.on('click', () => {
              this.selectedFolder = folder;
              this.currentFragmentIndex = 0;
              this.updateVideoSources();
            });
          }
        });
      }
    },
    addToMap(folder) {
      if (folder.eventData) {
            const { est_lat, est_lon } = folder.eventData;
            const marker = L.marker([est_lat, est_lon])
              .addTo(this.map)
              .bindPopup(`<b>${folder.name}</b><br>${folder.eventData.city}`);

            marker.on('click', () => {
              this.selectedFolder = folder;
              this.currentFragmentIndex = 0;
              this.updateVideoSources();
            });
      }
    },
    selectedFragment(camera) {
      try {
        const fragment = this.selectedFolder.fragments.find(
          (frag) => frag.camera === camera && frag.timestamp.getTime() === this.selectedFolder.fragments[this.currentFragmentIndex].timestamp.getTime()
        );
        return fragment ? fragment.url : '';
      } catch {
        this.close();
        return '';
      }
    },
    playAll() {
      try {
        this.$refs.frontVideo.play();
        this.$refs.leftRepeaterVideo.play();
        this.$refs.rightRepeaterVideo.play();
        this.$refs.backVideo.play();
      } catch {
        this.close();
      }
    },
    pauseAll() {
      try {
        this.$refs.frontVideo.pause();
        this.$refs.leftRepeaterVideo.pause();
        this.$refs.rightRepeaterVideo.pause();
        this.$refs.backVideo.pause();
      } catch {
        this.close();
      }
    },
    ended() {
      this.nextFragment();
    },
    nextFragment() {
      if (this.currentFragmentIndex < this.selectedFolder.fragments.length - 1) {
        this.currentFragmentIndex+=4;
        this.updateVideoSources();
      }
    },
    previousFragment() {
      if (this.currentFragmentIndex >= 4) {
        this.currentFragmentIndex-=4;
        this.updateVideoSources();
      }
    },
    progress(event) {
      this.$refs.leftRepeaterVideo.currentTime = event.target.currentTime;
      this.$refs.rightRepeaterVideo.currentTime = event.target.currentTime;
      this.$refs.backVideo.currentTime = event.target.currentTime;
    },
    updateVideoSources() {
      this.$refs.frontVideo.src = this.selectedFragment('front');
      this.$refs.leftRepeaterVideo.src = this.selectedFragment('left_repeater');
      this.$refs.rightRepeaterVideo.src = this.selectedFragment('right_repeater');
      this.$refs.backVideo.src = this.selectedFragment('back');
      this.playAll();
    },
    close() {
      this.selectedFolder = null;
    },
    swap(dom) {
      console.log(dom);
    }
  },
  mounted() {
    this.startUpdatingTime();
  },
  beforeDestroy() {
    this.stopUpdatingTime();
  },
};
</script>

<style>
#map {
  height: 100vh;
  width: 100vw;
  position: absolute;
  top: 0;
  left: 0;
}

.video-grid {
  height: 100vh;
  width: 100vw;
  position: fixed;
  top: 0;
  left: 0;
  background-color: black;
  z-index: 1000;
}

.video-container {
  padding: 10px;
  box-sizing: border-box;
}

.front {
  grid-column: 1 / span 2;
  grid-row: 1 / span 1;
  width: 100%;
  height: 100%;
}

.back {
  grid-column: 1 / span 2;
  grid-row: 2 / span 1;
  width: 100%;
  height: 100%;
}

.left-repeater {
  grid-column: 1 / span 1;
  grid-row: 1 / span 2;
  width: 50%;
  height: 50%;
}

.right-repeater {
  grid-column: 2 / span 1;
  grid-row: 1 / span 2;
  width: 50%;
  height: 50%;
}

.controls {
  position: absolute;
  top: 10px;
  left: 50%;
  transform: translate(-50%, 0);
  z-index: 1001;
}

.time {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 1001;
}

.big {
  width: 100%;
  height: 100%;
}

.left {
  position: fixed;
  top: 60vh;
  left: 2vw;
  height: 35vh;
}

.mid {
  position: fixed;
  top: 60vh;
  left: 50%;
  transform: translate(-50%, 0);
  height: 35vh;
}

.right {
  position: fixed;
  top: 60vh;
  right: 2vw;
  height: 35vh;
}

#main-button {
  margin: 5px;
  border-radius: 1rem;
  padding: 0.5rem 1rem;
  background-color: tomato;
}

button {
  margin: 5px;
  border-radius: 1rem;
  padding: 0.3rem 0.6rem;
  background-color: thistle;
}
</style>
