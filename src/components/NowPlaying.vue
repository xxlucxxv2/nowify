<template>
  <div id="app">
    <div
      v-if="player.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div class="now-playing__cover">
        <img
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
        />
      </div>
      <div class="now-playing__details">
        <h1 class="now-playing__track" v-text="player.trackTitle"></h1>
        <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
        <div class="now-playing__progress-bar">
          <div class="now-playing__progress" :style="{ width: progressPercent + '%' }"></div>
        </div>
      </div>
    </div>
    <div v-else class="now-playing" :class="getNowPlayingClass()">
      <h1 class="now-playing__idle-heading">No music is playing 😔</h1>
    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant'
import props from '@/utils/props.js'

export default {
  name: 'NowPlaying',

  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player
  },

  data() {
    return {
      pollPlaying: '',
      tickTimer: null,
      playerResponse: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: [],
      progressMs: 0,
      durationMs: 0
    }
  },

  computed: {
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    },
    progressPercent() {
      return this.durationMs > 0
        ? Math.min((this.progressMs / this.durationMs) * 100, 100)
        : 0
    }
  },

  mounted() {
    this.setDataInterval()
  },

  beforeUnmount() {
    clearInterval(this.pollPlaying)
    clearInterval(this.tickTimer)
  },

  methods: {
    async getNowPlaying() {
      let data = {}

      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        if (response.status === 204) {
          data = this.getEmptyPlayer()
          this.playerData = data
          this.$nextTick(() => this.$emit('spotifyTrackUpdated', data))
          return
        }

        if (!response.ok) throw new Error(`Status: ${response.status}`)

        data = await response.json()
        this.playerResponse = data

        // ⏱ Real timing info
        if (data.progress_ms && data.item?.duration_ms) {
          const newProgress = data.progress_ms
          const newDuration = data.item.duration_ms

          // Only update if there's a real change or large drift
          if (Math.abs(this.progressMs - newProgress) > 2000) {
            this.progressMs = newProgress
          }

          this.durationMs = newDuration

          // Restart local ticker
          clearInterval(this.tickTimer)
          this.tickTimer = setInterval(() => {
            if (this.progressMs < this.durationMs) {
              this.progressMs += 1000
            }
          }, 1000)
        }
      } catch (err) {
        this.handleExpiredToken()
        data = this.getEmptyPlayer()
        this.playerData = data
        this.$nextTick(() => this.$emit('spotifyTrackUpdated', data))
      }
    },

    getNowPlayingClass() {
      return this.player.playing ? 'now-playing--active' : 'now-playing--idle'
    },

    getAlbumColours() {
      if (!this.player.trackAlbum?.image) return

      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(this.handleAlbumPalette)
    },

    handleNowPlaying() {
      const err = this.playerResponse.error?.status
      if (err === 401 || err === 400) return this.handleExpiredToken()

      if (!this.playerResponse.is_playing) {
        clearInterval(this.tickTimer)
        this.playerData = this.getEmptyPlayer()
        return
      }

      if (this.playerResponse.item?.id === this.playerData.trackId) return

      this.playerData = {
        playing: this.playerResponse.is_playing,
        trackArtists: this.playerResponse.item.artists.map(a => a.name),
        trackTitle: this.playerResponse.item.name,
        trackId: this.playerResponse.item.id,
        trackAlbum: {
          title: this.playerResponse.item.album.name,
          image: this.playerResponse.item.album.images[0].url
        }
      }
    },

    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(this.getNowPlaying, 2500)
    },

    handleAlbumPalette(palette) {
      let albumColours = Object.keys(palette)
        .filter(k => palette[k])
        .map(k => ({
          text: palette[k].getTitleTextColor(),
          background: palette[k].getHex()
        }))

      this.swatches = albumColours
      this.colourPalette = albumColours[Math.floor(Math.random() * albumColours.length)]

      this.$nextTick(this.setAppColours)
    },

    setAppColours() {
      document.documentElement.style.setProperty('--color-text-primary', this.colourPalette.text)
      document.documentElement.style.setProperty('--colour-background-now-playing', this.colourPalette.background)
    },

    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: ''
      }
    },

    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      clearInterval(this.tickTimer)
      this.$emit('requestRefreshToken')
    }
  },

  watch: {
    auth(newVal) {
      if (newVal.status === false) clearInterval(this.pollPlaying)
    },

    playerResponse() {
      this.handleNowPlaying()
    },

    playerData() {
      this.$emit('spotifyTrackUpdated', this.playerData)
      this.$nextTick(this.getAlbumColours)
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
