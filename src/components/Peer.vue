<template>
  <video playsInline :ref="`videoElem${id}`" class="peer-video rounded" :class="{ 'disabled': !videoTrack && !shareTrack }"
    autoPlay :muted="true" :controls="false" />
  <audio :ref="`audioElem${id}`" autoPlay :controls="false" />
  <div class="peer-name text-gray-500">{{ name }}</div>
</template>
<script lang="ts">
export default {
  name: "Peer",
  computed: {
    videoTrack() {
      return this.tracks.find(t => t.type === 'video' && t.peerId === this.id);
    },
    shareTrack() {
      return this.tracks.find(t => t.type === 'share' && t.peerId === this.id);
    }
  },
  methods: {
    del() {
      this.room.localPeer.removeUser(this.id);
    },
    disableTrack(kind: 'audio' | 'video' | 'share') {
      this.room.localPeer.disablePeerTrack(this.id, kind);
    }
  },
  watch: {
    videoTrack(newTrack) {
      console.log(newTrack, 'newTrack');
    }
  },
  props: {
    tracks: Array,
    id: String,
    peer: Object,
    name: String,
    room: Object,
  }
}
</script>
