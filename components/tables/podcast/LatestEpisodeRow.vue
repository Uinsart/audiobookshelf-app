<template>
  <div class="w-full py-4 overflow-hidden relative border-b border-white border-opacity-10" @click.stop="goToEpisodePage">
    <div v-if="episode" class="w-full px-1">
      <div class="flex mb-2">
        <div class="w-10 min-w-10">
          <covers-preview-cover :src="$store.getters['globals/getLibraryItemCoverSrcById'](libraryItemId)" :width="40" :book-cover-aspect-ratio="bookCoverAspectRatio" :show-resolution="false" class="md:hidden" />
        </div>
        <div class="flex-grow px-2">
          <div class="-mt-0.5 mb-0.5" @click.stop>
            <nuxt-link :to="`/item/${libraryItemId}`" class="text-sm text-gray-200 underline">{{ podcast.metadata.title }}</nuxt-link>
          </div>
          <p v-if="publishedAt" class="text-xs text-gray-300">{{ $dateDistanceFromNow(publishedAt) }}</p>
        </div>
      </div>

      <p class="text-sm font-semibold">
        {{ title }}
      </p>

      <p class="text-sm text-gray-200 episode-subtitle mt-1.5 mb-0.5" v-html="subtitle" />

      <div v-if="episodeNumber || season || episodeType" class="flex pt-2 items-center -mx-0.5">
        <div v-if="episodeNumber" class="px-2 pt-px pb-0.5 mx-0.5 bg-primary bg-opacity-50 rounded-full text-xs font-light text-gray-200">{{ $strings.LabelEpisode }} #{{ episodeNumber }}</div>
        <div v-if="season" class="px-2 pt-px pb-0.5 mx-0.5 bg-primary bg-opacity-50 rounded-full text-xs font-light text-gray-200">{{ $strings.LabelSeason }} #{{ season }}</div>
        <div v-if="episodeType" class="px-2 pt-px pb-0.5 mx-0.5 bg-primary bg-opacity-50 rounded-full text-xs font-light text-gray-200 capitalize">
          {{ episodeType }}
        </div>
      </div>

      <div class="flex items-center pt-2">
        <div class="h-8 px-4 border border-white border-opacity-20 hover:bg-white hover:bg-opacity-10 rounded-full flex items-center justify-center cursor-pointer" :class="userIsFinished ? 'text-white text-opacity-40' : ''" @click.stop="playClick">
          <span class="material-icons" :class="streamIsPlaying ? '' : 'text-success'">{{ streamIsPlaying ? 'pause' : 'play_arrow' }}</span>
          <p class="pl-2 pr-1 text-sm font-semibold">{{ timeRemaining }}</p>
        </div>

        <ui-read-icon-btn :disabled="isProcessingReadUpdate" :is-read="userIsFinished" borderless class="mx-1 mt-0.5" @click="toggleFinished" />

        <button v-if="!isLocal" class="mx-1.5 mt-1.5" @click.stop="addToPlaylist">
          <span class="material-icons text-2xl">playlist_add</span>
        </button>

        <div v-if="userCanDownload">
          <span v-if="isLocal" class="material-icons-outlined px-2 text-success text-lg">audio_file</span>
          <span v-else-if="!localEpisode" class="material-icons mx-1.5 mt-2 text-xl" :class="downloadItem ? 'animate-bounce text-warning text-opacity-75' : ''" @click.stop="downloadClick">{{ downloadItem ? 'downloading' : 'download' }}</span>
          <span v-else class="material-icons px-2 text-success text-xl">download_done</span>
        </div>

        <div class="flex-grow" />
      </div>
    </div>

    <div v-if="processing" class="absolute top-0 left-0 w-full h-full bg-black bg-opacity-30 flex items-center justify-center">
      <widgets-loading-spinner size="la-lg" />
    </div>

    <div v-if="!userIsFinished" class="absolute bottom-0 left-0 h-0.5 bg-warning" :style="{ width: itemProgressPercent * 100 + '%' }" />
  </div>
</template>

<script>
import { Dialog } from '@capacitor/dialog'
import { AbsFileSystem, AbsDownloader } from '@/plugins/capacitor'

export default {
  props: {
    libraryItemId: String,
    episode: {
      type: Object,
      default: () => {}
    },
    localLibraryItemId: String,
    localEpisode: {
      type: Object,
      default: () => {}
    },
    isLocal: Boolean
  },
  data() {
    return {
      isProcessingReadUpdate: false,
      processing: false
    }
  },
  computed: {
    bookCoverAspectRatio() {
      return this.$store.getters['libraries/getBookCoverAspectRatio']
    },
    isIos() {
      return this.$platform === 'ios'
    },
    mediaType() {
      return 'podcast'
    },
    userCanDownload() {
      return this.$store.getters['user/getUserCanDownload']
    },
    audioFile() {
      return this.episode.audioFile
    },
    title() {
      return this.episode.title || ''
    },
    subtitle() {
      return this.episode.subtitle || this.episode.description || ''
    },
    episodeNumber() {
      return this.episode.episode
    },
    season() {
      return this.episode.season
    },
    episodeType() {
      if (this.episode.episodeType === 'full') return null // only show Trailer/Bonus
      return this.episode.episodeType
    },
    duration() {
      return this.$secondsToTimestamp(this.episode.duration)
    },
    isStreaming() {
      return this.$store.getters['getIsMediaStreaming'](this.libraryItemId, this.episode.id)
    },
    streamIsPlaying() {
      return this.$store.state.playerIsPlaying && this.isStreaming
    },
    itemProgress() {
      if (this.isLocal) return this.$store.getters['globals/getLocalMediaProgressById'](this.libraryItemId, this.episode.id)
      return this.$store.getters['user/getUserMediaProgress'](this.libraryItemId, this.episode.id)
    },
    localMediaProgress() {
      if (this.isLocal) return this.$store.getters['globals/getLocalMediaProgressById'](this.libraryItemId, this.episode.id)
      else if (this.localLibraryItemId && this.localEpisode) {
        return this.$store.getters['globals/getLocalMediaProgressById'](this.localLibraryItemId, this.localEpisode.id)
      } else {
        return null
      }
    },
    itemProgressPercent() {
      return this.itemProgress ? this.itemProgress.progress : 0
    },
    userIsFinished() {
      return this.itemProgress ? !!this.itemProgress.isFinished : false
    },
    timeRemaining() {
      if (this.streamIsPlaying) return 'Playing'
      if (!this.itemProgressPercent) return this.$elapsedPretty(this.episode.duration)
      if (this.userIsFinished) return 'Finished'
      var remaining = Math.floor(this.itemProgress.duration - this.itemProgress.currentTime)
      return `${this.$elapsedPretty(remaining)} left`
    },
    publishedAt() {
      return this.episode.publishedAt
    },
    downloadItem() {
      return this.$store.getters['globals/getDownloadItem'](this.libraryItemId, this.episode.id)
    },
    localEpisodeId() {
      return this.localEpisode ? this.localEpisode.id : null
    },
    podcast() {
      return this.episode.podcast || {}
    }
  },
  methods: {
    goToEpisodePage() {
      this.$router.push(`/item/${this.libraryItemId}/${this.episode.id}`)
    },
    addToPlaylist() {
      this.$emit('addToPlaylist', this.episode)
    },
    async selectFolder() {
      var folderObj = await AbsFileSystem.selectFolder({ mediaType: this.mediaType })
      if (folderObj.error) {
        return this.$toast.error(`Error: ${folderObj.error || 'Unknown Error'}`)
      }
      return folderObj
    },
    async downloadClick() {
      if (this.downloadItem) return
      await this.$hapticsImpact()
      if (this.isIos) {
        // no local folders on iOS
        this.startDownload()
      } else {
        this.download()
      }
    },
    async download(selectedLocalFolder = null) {
      let localFolder = selectedLocalFolder
      if (!localFolder) {
        const localFolders = (await this.$db.getLocalFolders()) || []
        console.log('Local folders loaded', localFolders.length)
        const foldersWithMediaType = localFolders.filter((lf) => {
          console.log('Checking local folder', lf.mediaType)
          return lf.mediaType == this.mediaType
        })
        console.log('Folders with media type', this.mediaType, foldersWithMediaType.length)
        const internalStorageFolder = foldersWithMediaType.find((f) => f.id === `internal-${this.mediaType}`)
        if (!foldersWithMediaType.length) {
          localFolder = {
            id: `internal-${this.mediaType}`,
            name: 'Internal App Storage',
            mediaType: this.mediaType
          }
        } else if (foldersWithMediaType.length === 1 && internalStorageFolder) {
          localFolder = internalStorageFolder
        } else {
          this.$store.commit('globals/showSelectLocalFolderModal', {
            mediaType: this.mediaType,
            callback: (folder) => {
              this.download(folder)
            }
          })
          return
        }
      }

      console.log('Local folder', JSON.stringify(localFolder))

      var startDownloadMessage = `Start download for "${this.title}" to folder ${localFolder.name}?`
      const { value } = await Dialog.confirm({
        title: 'Confirm',
        message: startDownloadMessage
      })
      if (value) {
        this.startDownload(localFolder)
      }
    },
    async startDownload(localFolder) {
      var payload = {
        libraryItemId: this.libraryItemId,
        episodeId: this.episode.id
      }
      if (localFolder) {
        payload.localFolderId = localFolder.id
      }
      var downloadRes = await AbsDownloader.downloadLibraryItem(payload)
      if (downloadRes && downloadRes.error) {
        var errorMsg = downloadRes.error || 'Unknown error'
        console.error('Download error', errorMsg)
        this.$toast.error(errorMsg)
      }
    },
    async playClick() {
      await this.$hapticsImpact()
      if (this.streamIsPlaying) {
        this.$eventBus.$emit('pause-item')
      } else {
        if (this.localEpisode && this.localLibraryItemId) {
          console.log('Play local episode', this.localEpisode.id, this.localLibraryItemId)

          this.$eventBus.$emit('play-item', {
            libraryItemId: this.localLibraryItemId,
            episodeId: this.localEpisode.id,
            serverLibraryItemId: this.libraryItemId,
            serverEpisodeId: this.episode.id
          })
        } else {
          this.$eventBus.$emit('play-item', {
            libraryItemId: this.libraryItemId,
            episodeId: this.episode.id
          })
        }
      }
    },
    async toggleFinished() {
      await this.$hapticsImpact()

      this.isProcessingReadUpdate = true
      if (this.isLocal || this.localEpisode) {
        const isFinished = !this.userIsFinished
        const localLibraryItemId = this.isLocal ? this.libraryItemId : this.localLibraryItemId
        const localEpisodeId = this.isLocal ? this.episode.id : this.localEpisode.id
        const payload = await this.$db.updateLocalMediaProgressFinished({ localLibraryItemId, localEpisodeId, isFinished })
        console.log('toggleFinished payload', JSON.stringify(payload))
        if (payload?.error) {
          this.$toast.error(payload?.error || 'Unknown error')
        } else {
          const localMediaProgress = payload.localMediaProgress
          console.log('toggleFinished localMediaProgress', JSON.stringify(localMediaProgress))
          if (localMediaProgress) {
            this.$store.commit('globals/updateLocalMediaProgress', localMediaProgress)
          }
        }
        this.isProcessingReadUpdate = false
      } else {
        const updatePayload = {
          isFinished: !this.userIsFinished
        }
        this.$nativeHttp
          .patch(`/api/me/progress/${this.libraryItemId}/${this.episode.id}`, updatePayload)
          .catch((error) => {
            console.error('Failed', error)
            this.$toast.error(`Failed to mark as ${updatePayload.isFinished ? 'Finished' : 'Not Finished'}`)
          })
          .finally(() => {
            this.isProcessingReadUpdate = false
          })
      }
    }
  }
}
</script>
