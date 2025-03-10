<template>
  <div class="w-full layout-wrapper bg-bg text-white">
    <app-appbar />
    <div id="content" class="overflow-hidden relative" :class="isPlayerOpen ? 'playerOpen' : ''">
      <Nuxt :key="currentLang" />
    </div>
    <app-audio-player-container ref="streamContainer" />
    <modals-libraries-modal />
    <modals-playlists-add-create-modal />
    <modals-select-local-folder-modal />
    <modals-rssfeeds-rss-feed-modal />
    <app-side-drawer :key="currentLang" />
    <readers-reader />
  </div>
</template>

<script>
import { CapacitorHttp } from '@capacitor/core'

export default {
  data() {
    return {
      inittingLibraries: false,
      hasMounted: false,
      disconnectTime: 0,
      timeLostFocus: 0,
      currentLang: null
    }
  },
  watch: {
    networkConnected: {
      handler(newVal, oldVal) {
        if (!this.hasMounted) {
          // watcher runs before mount, handling libraries/connection should be handled in mount
          return
        }
        if (newVal) {
          console.log(`[default] network connected changed ${oldVal} -> ${newVal}`)
          if (!this.user) {
            this.attemptConnection()
          } else if (!this.currentLibraryId) {
            this.initLibraries()
          } else {
            var timeSinceDisconnect = Date.now() - this.disconnectTime
            if (timeSinceDisconnect > 5000) {
              console.log('Time since disconnect was', timeSinceDisconnect, 'sync with server')
              setTimeout(() => {
                this.syncLocalSessions()
              }, 4000)
            }
          }
        } else {
          console.log(`[default] lost network connection`)
          this.disconnectTime = Date.now()
        }
      }
    }
  },
  computed: {
    isPlayerOpen() {
      return this.$store.getters['getIsPlayerOpen']
    },
    routeName() {
      return this.$route.name
    },
    networkConnected() {
      return this.$store.state.networkConnected
    },
    user() {
      return this.$store.state.user.user
    },
    currentLibraryId() {
      return this.$store.state.libraries.currentLibraryId
    },
    attemptingConnection: {
      get() {
        return this.$store.state.attemptingConnection
      },
      set(val) {
        this.$store.commit('setAttemptingConnection', val)
      }
    }
  },
  methods: {
    initialStream(stream) {
      if (this.$refs.streamContainer?.audioPlayerReady) {
        this.$refs.streamContainer.streamOpen(stream)
      }
    },
    async loadSavedSettings() {
      const userSavedServerSettings = await this.$localStore.getServerSettings()
      if (userSavedServerSettings) {
        this.$store.commit('setServerSettings', userSavedServerSettings)
      }

      await this.$store.dispatch('user/loadUserSettings')
    },
    async postRequest(url, data, headers, connectTimeout = 30000) {
      const options = {
        url,
        headers,
        data,
        connectTimeout
      }
      const response = await CapacitorHttp.post(options)
      console.log('[default] POST request response', response)
      if (response.status >= 400) {
        throw new Error(response.data)
      } else {
        return response.data
      }
    },
    async attemptConnection() {
      console.warn('[default] attemptConnection')
      if (!this.networkConnected) {
        console.warn('[default] No network connection')
        return
      }
      if (this.attemptingConnection) {
        return
      }
      this.attemptingConnection = true

      const deviceData = await this.$db.getDeviceData()
      let serverConfig = null
      if (deviceData) {
        this.$store.commit('globals/setHapticFeedback', deviceData.deviceSettings?.hapticFeedback)

        if (deviceData.lastServerConnectionConfigId && deviceData.serverConnectionConfigs.length) {
          serverConfig = deviceData.serverConnectionConfigs.find((scc) => scc.id == deviceData.lastServerConnectionConfigId)
        }
      }

      if (!serverConfig) {
        // No last server config set
        this.attemptingConnection = false
        return
      }

      console.log(`[default] Got server config, attempt authorize ${serverConfig.address}`)

      const authRes = await this.postRequest(`${serverConfig.address}/api/authorize`, null, { Authorization: `Bearer ${serverConfig.token}` }, 10000).catch((error) => {
        console.error('[default] Server auth failed', error)
        return false
      })
      if (!authRes) {
        this.attemptingConnection = false
        return
      }

      const { user, userDefaultLibraryId, serverSettings, ereaderDevices } = authRes
      this.$store.commit('setServerSettings', serverSettings)
      this.$store.commit('libraries/setEReaderDevices', ereaderDevices)

      // Set library - Use last library if set and available fallback to default user library
      const lastLibraryId = await this.$localStore.getLastLibraryId()
      if (lastLibraryId && (!user.librariesAccessible.length || user.librariesAccessible.includes(lastLibraryId))) {
        this.$store.commit('libraries/setCurrentLibrary', lastLibraryId)
      } else if (userDefaultLibraryId) {
        this.$store.commit('libraries/setCurrentLibrary', userDefaultLibraryId)
      }
      const serverConnectionConfig = await this.$db.setServerConnectionConfig(serverConfig)

      this.$store.commit('user/setUser', user)
      this.$store.commit('user/setServerConnectionConfig', serverConnectionConfig)

      this.$socket.connect(serverConnectionConfig.address, serverConnectionConfig.token)

      console.log('[default] Successful connection on last saved connection config', JSON.stringify(serverConnectionConfig))
      await this.initLibraries()
      this.attemptingConnection = false
    },
    itemRemoved(libraryItem) {
      if (this.$route.name.startsWith('item')) {
        if (this.$route.params.id === libraryItem.id) {
          this.$router.replace(`/bookshelf`)
        }
      }
    },
    userLoggedOut() {
      // Only cancels stream if streamining not playing downloaded
      this.$eventBus.$emit('close-stream')
    },
    socketConnectionFailed(err) {
      this.$toast.error('Socket connection error: ' + err.message)
    },
    async initLibraries() {
      if (this.inittingLibraries) {
        return
      }
      this.inittingLibraries = true
      await this.$store.dispatch('libraries/load')
      console.log(`[default] initLibraries loaded ${this.currentLibraryId}`)
      await this.$store.dispatch('libraries/fetch', this.currentLibraryId)
      this.$eventBus.$emit('library-changed')
      this.inittingLibraries = false
    },
    async syncLocalSessions() {
      if (!this.user) {
        console.log('[default] No need to sync local sessions - not connected to server')
        return
      }

      console.log('[default] Calling syncLocalSessions')
      const response = await this.$db.syncLocalSessionsWithServer()
      if (response?.error) {
        console.error('[default] Failed to sync local sessions', response.error)
      } else {
        console.log('[default] Successfully synced local sessions')
        // Reload local media progresses
        await this.$store.dispatch('globals/loadLocalMediaProgress')
      }
    },
    userUpdated(user) {
      // console.log('[default] userUpdated:', JSON.stringify(user))
      if (this.user && this.user.id == user.id) {
        this.$store.commit('user/setUser', user)
      }
    },
    async userMediaProgressUpdated(payload) {
      const prog = payload.data // MediaProgress
      console.log(`[default] userMediaProgressUpdate checking for local media progress ${payload.id}`)

      // Update local media progress if exists
      const localProg = await this.$db.getLocalMediaProgressForServerItem({ libraryItemId: prog.libraryItemId, episodeId: prog.episodeId })

      let newLocalMediaProgress = null
      if (localProg && localProg.lastUpdate < prog.lastUpdate) {
        if (localProg.currentTime == prog.currentTime && localProg.isFinished == prog.isFinished) {
          console.log('[default] syncing progress server lastUpdate > local lastUpdate but currentTime and isFinished is equal')
          return
        } else {
          console.log(`[default] syncing progress server lastUpdate > local lastUpdate. server currentTime=${prog.currentTime} local currentTime=${localProg.currentTime} | server/local isFinished=${prog.isFinished}/${localProg.isFinished}`)
        }

        // Check if this media item is currently open in the player, paused, and this progress update is coming from a different session
        const isMediaOpenInPlayer = this.$store.getters['getIsMediaStreaming'](prog.libraryItemId, prog.episodeId)
        if (isMediaOpenInPlayer && this.$store.getters['getCurrentPlaybackSessionId'] !== payload.sessionId && !this.$store.state.playerIsPlaying) {
          console.log('[default] userMediaProgressUpdated for current open media item', payload.data.currentTime)
          this.$eventBus.$emit('playback-time-update', payload.data.currentTime)
        }

        // Server progress is more up-to-date
        console.log(`[default] syncing progress from server with local item for "${prog.libraryItemId}" ${prog.episodeId ? `episode ${prog.episodeId}` : ''} | server lastUpdate=${prog.lastUpdate} > local lastUpdate=${localProg.lastUpdate}`)
        const payload = {
          localMediaProgressId: localProg.id,
          mediaProgress: prog
        }
        newLocalMediaProgress = await this.$db.syncServerMediaProgressWithLocalMediaProgress(payload)
      } else if (!localProg) {
        // Check if local library item exists
        //   local media progress may not exist yet if it hasn't been played
        const localLibraryItem = await this.$db.getLocalLibraryItemByLId(prog.libraryItemId)
        if (localLibraryItem) {
          if (prog.episodeId) {
            // If episode check if local episode exists
            const lliEpisodes = localLibraryItem.media.episodes || []
            const localEpisode = lliEpisodes.find((ep) => ep.serverEpisodeId === prog.episodeId)
            if (localEpisode) {
              // Add new local media progress
              const payload = {
                localLibraryItemId: localLibraryItem.id,
                localEpisodeId: localEpisode.id,
                mediaProgress: prog
              }
              newLocalMediaProgress = await this.$db.syncServerMediaProgressWithLocalMediaProgress(payload)
            }
          } else {
            // Add new local media progress
            const payload = {
              localLibraryItemId: localLibraryItem.id,
              mediaProgress: prog
            }
            newLocalMediaProgress = await this.$db.syncServerMediaProgressWithLocalMediaProgress(payload)
          }
        } else {
          console.log(`[default] userMediaProgressUpdate no local media progress or lli found for this server item ${prog.id}`)
        }
      }

      if (newLocalMediaProgress?.id) {
        console.log(`[default] local media progress updated for ${newLocalMediaProgress.id}`)
        this.$store.commit('globals/updateLocalMediaProgress', newLocalMediaProgress)
      }
    },
    async visibilityChanged() {
      if (document.visibilityState === 'visible') {
        const elapsedTimeOutOfFocus = Date.now() - this.timeLostFocus
        console.log(`✅ [default] device visibility: has focus (${elapsedTimeOutOfFocus}ms out of focus)`)
        // If device out of focus for more than 30s then reload local media progress
        if (elapsedTimeOutOfFocus > 30000) {
          console.log(`✅ [default] device visibility: reloading local media progress`)
          // Reload local media progresses
          await this.$store.dispatch('globals/loadLocalMediaProgress')
        }
        if (document.visibilityState === 'visible') {
          this.$eventBus.$emit('device-focus-update', true)
        }
      } else {
        console.log('⛔️ [default] device visibility: does NOT have focus')
        this.timeLostFocus = Date.now()
        this.$eventBus.$emit('device-focus-update', false)
      }
    },
    changeLanguage(code) {
      console.log('Changed lang', code)
      this.currentLang = code
      document.documentElement.lang = code
    }
  },
  async mounted() {
    this.$eventBus.$on('change-lang', this.changeLanguage)
    document.addEventListener('visibilitychange', this.visibilityChanged)

    this.$socket.on('user_updated', this.userUpdated)
    this.$socket.on('user_media_progress_updated', this.userMediaProgressUpdated)

    if (this.$store.state.isFirstLoad) {
      this.$store.commit('setIsFirstLoad', false)

      const deviceData = await this.$db.getDeviceData()
      this.$store.commit('setDeviceData', deviceData)

      this.$setOrientationLock(this.$store.getters['getOrientationLockSetting'])

      await this.$store.dispatch('setupNetworkListener')

      await this.$store.dispatch('globals/loadLocalMediaProgress')

      if (this.$store.state.user.serverConnectionConfig) {
        await this.initLibraries()
      } else {
        await this.attemptConnection()
      }

      console.log(`[default] finished connection attempt or already connected ${!!this.user}`)
      await this.syncLocalSessions()

      this.loadSavedSettings()
      this.hasMounted = true

      console.log('[default] fully initialized')
      this.$eventBus.$emit('abs-ui-ready')
    }
  },
  beforeDestroy() {
    this.$eventBus.$off('change-lang', this.changeLanguage)
    document.removeEventListener('visibilitychange', this.visibilityChanged)
    this.$socket.off('user_updated', this.userUpdated)
    this.$socket.off('user_media_progress_updated', this.userMediaProgressUpdated)
  }
}
</script>
