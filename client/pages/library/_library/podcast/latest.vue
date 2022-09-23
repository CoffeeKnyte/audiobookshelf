<template>
  <div class="page" :class="streamLibraryItem ? 'streaming' : ''">
    <app-book-shelf-toolbar page="recent-episodes" />

    <div id="bookshelf" class="w-full overflow-y-auto px-2 py-6 sm:px-4 md:p-12 relative">
      <div class="w-full max-w-3xl mx-auto py-4">
        <p class="text-xl mb-2 font-semibold">Latest episodes</p>
        <p v-if="!recentEpisodes.length && !processing" class="text-center text-xl">No podcasts found</p>
        <template v-for="(episode, index) in episodesMapped">
          <div :key="episode.id" class="flex py-5 cursor-pointer relative" @click.stop="clickEpisode(episode)">
            <covers-preview-cover :src="$store.getters['globals/getLibraryItemCoverSrcById'](episode.libraryItemId)" :width="96" :book-cover-aspect-ratio="bookCoverAspectRatio" :show-resolution="false" />
            <div class="flex-grow pl-4 max-w-2xl">
              <nuxt-link :to="`/item/${episode.libraryItemId}`" class="text-sm text-gray-200 hover:underline">{{ episode.podcast.metadata.title }}</nuxt-link>

              <p class="text-xs text-gray-300 mb-1">{{ $dateDistanceFromNow(episode.publishedAt) }}</p>

              <p class="font-semibold mb-2">{{ episode.title }}</p>

              <p class="text-sm text-gray-200 mb-4">{{ episode.subtitle }}</p>

              <button class="h-8 px-4 border border-white border-opacity-20 hover:bg-white hover:bg-opacity-10 rounded-full flex items-center justify-center cursor-pointer focus:outline-none" :class="episode.progress && episode.progress.isFinished ? 'text-white text-opacity-40' : ''" @click.stop="playClick(episode)">
                <span v-if="episodeIdStreaming === episode.id" class="material-icons" :class="streamIsPlaying ? '' : 'text-success'">{{ streamIsPlaying ? 'pause' : 'play_arrow' }}</span>
                <span v-else class="material-icons text-success">play_arrow</span>
                <p class="pl-2 pr-1 text-sm font-semibold">{{ getButtonText(episode) }}</p>
              </button>
            </div>

             <div v-if="episode.progress" class="absolute bottom-0 left-0 h-0.5 pointer-events-none bg-warning" :style="{ width: episode.progress.progress * 100 + '%' }" />
          </div>
          <div :key="index" v-if="index !== recentEpisodes.length" class="w-full h-px bg-white bg-opacity-10" />
        </template>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  async asyncData({ params, query, store, app, redirect }) {
    var libraryId = params.library
    var libraryData = await store.dispatch('libraries/fetch', libraryId)
    if (!libraryData) {
      return redirect('/oops?message=Library not found')
    }

    // Redirect book libraries
    const library = libraryData.library
    if (library.mediaType === 'book') {
      return redirect(`/library/${libraryId}`)
    }

    return {
      libraryId
    }
  },
  data() {
    return {
      recentEpisodes: [],
      totalEpisodes: 0,
      currentPage: 0,
      processing: false,
      openingItem: false
    }
  },
  computed: {
    streamLibraryItem() {
      return this.$store.state.streamLibraryItem
    },
    bookCoverAspectRatio() {
      return this.$store.getters['libraries/getBookCoverAspectRatio']
    },
    libraryItemIdStreaming() {
      return this.$store.getters['getLibraryItemIdStreaming']
    },
    episodeIdStreaming() {
      return this.$store.state.streamEpisodeId
    },
    streamIsPlaying() {
      return this.$store.state.streamIsPlaying
    },
    episodesMapped() {
      return this.recentEpisodes.map((ep) => {
        return {
          ...ep,
          progress: this.$store.getters['user/getUserMediaProgress'](ep.libraryItemId, ep.id)
        }
      })
    }
  },
  methods: {
    async clickEpisode(episode) {
      if (this.openingItem) return
      this.openingItem = true
      const fullLibraryItem = await this.$axios.$get(`/api/items/${episode.libraryItemId}`).catch((error) => {
        var errMsg = error.response ? error.response.data || '' : ''
        this.$toast.error(errMsg || 'Failed to get library item')
        return null
      })
      this.openingItem = false
      if (!fullLibraryItem) return

      this.$store.commit('setSelectedLibraryItem', fullLibraryItem)
      this.$store.commit('globals/setSelectedEpisode', episode)
      this.$store.commit('globals/setShowViewPodcastEpisodeModal', true)
    },
    getButtonText(episode) {
      if (this.episodeIdStreaming === episode.id) return this.streamIsPlaying ? 'Streaming' : 'Play'
      if (!episode.progress) return this.$elapsedPretty(episode.duration)
      if (episode.progress.isFinished) return 'Finished'
      var remaining = Math.floor(episode.progress.duration - episode.progress.currentTime)
      return `${this.$elapsedPretty(remaining)} left`
    },
    playClick(episodeToPlay) {
      if (episodeToPlay.id === this.episodeIdStreaming && this.streamIsPlaying) {
        return this.$eventBus.$emit('pause-item')
      }

      // Queue up more recent items
      const queueItems = []
      const episodeIndex = this.episodesMapped.findIndex((e) => e.id === episodeToPlay.id)
      const indexFromBack = this.episodesMapped.length - episodeIndex - 1
      for (let i = this.episodesMapped.length - 1 - indexFromBack; i >= 0; i--) {
        const episode = this.episodesMapped[i]
        if (!episode.progress || !episode.isFinished) {
          queueItems.push({
            libraryItemId: episode.libraryItemId,
            episodeId: episode.id,
            title: episode.title,
            subtitle: episode.podcast.metadata.title,
            caption: episode.publishedAt ? `Published ${this.$formatDate(episode.publishedAt, 'MMM do, yyyy')}` : 'Unknown publish date',
            duration: episode.duration || null,
            coverPath: episode.podcast.coverPath || null
          })
        }
      }

      this.$eventBus.$emit('play-item', {
        libraryItemId: episodeToPlay.libraryItemId,
        episodeId: episodeToPlay.id,
        queueItems
      })
    },
    async loadRecentEpisodes(page = 0) {
      this.processing = true
      const episodePayload = await this.$axios.$get(`/api/libraries/${this.libraryId}/recent-episodes?limit=25&page=${page}`).catch((error) => {
        console.error('Failed to get recent episodes', error)
        this.$toast.error('Failed to get recent episodes')
        return null
      })
      this.processing = false
      console.log('Episodes', episodePayload)
      this.recentEpisodes = episodePayload.episodes || []
      this.totalEpisodes = episodePayload.total
      this.currentPage = page
    }
  },
  mounted() {
    this.loadRecentEpisodes()
  }
}
</script>