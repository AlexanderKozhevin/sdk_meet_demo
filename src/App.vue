<template>
  <div>
    <div class="preview" v-if="!joined">
      <h1 class="mb-4 mt-4">Enter your name</h1>
      <input class="p-2 border bg-gray-100 rounded-xl" v-model="name" placeholder="Name" type="text" />

      <!-- <button class="enter-room-button" @click="joinRoom">
        Join Room
      </button> -->
      <button type="button" @click="joinRoom()"  class="ml-2 bg-indigo-50 text-indigo-700 inline-block rounded-xl font-medium leading-none py-3.5 px-3 focus:outline-none bg-indigo-50 text-indigo-700">
          Preview
        </button>
    </div>
    <div class="wrapper" v-else>

      <div class="peers">
        <div class="peer rounded-xl border" :key="id" v-for="(peer, id) in peers">
          <peer :ref="`peer${peer.id}`" :id="peer.id" :peer="peer" :tracks="tracks" :name="peer.name" :room="room" />
        </div>
      </div>



      <div class="controls">
        <button class="control" @click="toggleMic">
          {{ micEnabled ? "disable mic" : "enable mic" }}
        </button>
        <button class="control" @click="toggleWebcam">
          {{ camEnabled ? "disable cam" : "enable cam" }}
        </button>
        <button class="control" @click="toggleShare">
          {{ shareEnabled ? "disable share" : "enable share" }}
        </button>
      </div>


    </div>
  </div>
</template>

<script lang="ts">
import sdk from "alpha-meet-js-sdk";
import Peer from "./components/Peer.vue";
export default {
  name: "App",
  components: {
    Peer,
  },
  computed: {
    shareEnabled() {
      return this.tracks.find(t => t.type === 'share' && t.isLocal);
    },
    camEnabled() {
      return this.tracks.find(t => t.type === 'video' && t.isLocal);
    },
    micEnabled() {
      return this.tracks.find(t => t.type === 'audio' && t.isLocal);
    }
  },
  data() {
    return {
      peers: [],
      consumers: [],
      videoDevices: [],
      audioDevices: [],
      selectedVideoDeviceId: 'default',
      joined: false,
      room: null,
      name: '',
      tracks: [],
    };
  },
  async created() {
    const devices = await navigator.mediaDevices.enumerateDevices();
    this.videoDevices = devices.filter(d => d.kind === 'videoinput');
    this.audioDevices = devices.filter(d => d.kind === 'audioinput');
  },

  methods: {
    async joinRoom() {
      this.room = new sdk.Room("serv0poqni01m");
      this.room.connect(this.name, {
        queryParams: {
          webinar: true,
        },
        hostname: "webrtc2.edgelive.ru",
      });
      this.room.on(sdk.ClientEventTypes.RoomConnected, async () => {
        this.peers = [...this.room.allPeers];
      })
        .on(sdk.ClientEventTypes.NewPeer, ({ peer }) => {
          console.log(peer, 'peer')
          this.peers = [...this.peers, peer];
        })
        .on(sdk.ClientEventTypes.PeerClosed, ({ peerId }) => {
          this.peers = this.peers.filter((p) => p.id !== peerId);
          this.consumers = this.consumers.filter(c => c.peerId !== peerId);
        })
        .on(sdk.ClientEventTypes.NewTrack, ({ track }) => {
          this.tracks = [...this.tracks, track];
          this.attachTrack(track);
        })
        .on(sdk.ClientEventTypes.TrackEnded, ({ id }) => {
          const track = this.tracks.find(t => t.id === id);
          const peer = this.peers.find(p => track.peerId === p.id);
          if (peer) this.detachTrack(track);
          this.tracks = this.tracks.filter(t => t.id !== id);
        })

      this.joined = true;
    },
    async enableMic(deviceId = 'default'): Promise<void> {
      await this.room.localPeer.enableMic(deviceId);
    },
    async enableCam(deviceId = 'default'): Promise<void> {
      this.selectedVideoDeviceId = deviceId;
      await this.room.localPeer.enableWebcam(deviceId);
    },
    async toggleMic() {
      if (this.micEnabled) {
        await this.room.localPeer.disableMic();
      } else {
        this.enableMic();
      }
    },
    attachTrack(track) {
      if (track.kind === 'video') {
        const stream = new MediaStream();
        stream.addTrack(track);
        const videoElem = this.$refs[`peer${track.peerId}`][0].$refs[`videoElem${track.peerId}`]
        videoElem.srcObject = stream;
        videoElem.play();
      } else {
        const stream = new MediaStream();
        stream.addTrack(track);
        const audioElem = this.$refs[`peer${track.peerId}`][0].$refs[`audioElem${track.peerId}`]
        audioElem.srcObject = stream;
        audioElem.play();
      }
    },
    detachTrack(track) {
      if (track.kind === 'video') {
        const videoElem = this.$refs[`peer${track.peerId}`][0].$refs[`videoElem${track.peerId}`]
        videoElem.srcObject = null;
      } else {
        const audioElem = this.$refs[`peer${track.peerId}`][0].$refs[`audioElem${track.peerId}`]
        audioElem.srcObject = null;
      }
    },
    async toggleWebcam() {
      if (!this.camEnabled) {
        await this.enableCam(this.selectedVideoDeviceId);
      }
      else {
        await this.room.localPeer.disableWebcam();
      }
    },
    async toggleShare() {
      if (!this.shareEnabled) {
        await this.room.localPeer.enableShare();
      }
      else {
        await this.room.localPeer.disableShare();
      }
    }
  },
};
</script>

<style>
#app {
  text-align: center;
}

.peers {
  justify-content: center;
  display: flex;
  flex-wrap: wrap;
  height: 80vh;
  overflow: scroll;
}

.peer {
  display: flex;
  margin: 20px;
  width: 300px;
  height: 300px;
  border: 1px solid;
  flex-direction: column;
  justify-content: center;
}

video {
  width: 100%;
}

.controls {
  display: flex;
  justify-content: center;
  margin-top: 20px;
}

.control {
  margin: 20px;
  background-color: initial;
  border: 1px solid #75aeca;
  border-radius: 3px;
  color: #548eaa;
  cursor: pointer;
  display: inline-block;
  font-size: .8125rem;
  height: 36px;
  line-height: 2.25rem;
  margin-top: 18px;
  padding: 0 13px;
  text-decoration: none;
}

.control:hover {
  background-color: #7aa1bd;
  border: 1px solid transparent;
  color: #fff;
}

.peer-name {
  margin-top: 20px;
  font: bold 1.5rem/1.5rem "Helvetica Neue", Helvetica, Arial, sans-serif;
}

.disabled {
  display: none;
}

.video-device {
  cursor: pointer;
}
</style>
