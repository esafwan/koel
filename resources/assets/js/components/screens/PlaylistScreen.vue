<template>
  <section v-if="playlist" id="playlistWrapper">
    <ScreenHeader :layout="songs.length === 0 ? 'collapsed' : headerLayout" :disabled="loading">
      {{ playlist.name }}
      <ControlsToggle v-if="songs.length" v-model="showingControls"/>

      <template v-slot:thumbnail>
        <ThumbnailStack :thumbnails="thumbnails"/>
      </template>

      <template v-if="songs.length" v-slot:meta>
        <span>{{ pluralize(songs, 'song') }}</span>
        <span>{{ duration }}</span>
        <a
          v-if="allowDownload"
          href
          role="button"
          title="Download all songs in playlist"
          @click.prevent="download"
        >
          Download All
        </a>
      </template>

      <template v-slot:controls>
        <SongListControls
          v-if="!isPhone || showingControls"
          :config="controlsConfig"
          @deletePlaylist="destroy"
          @playAll="playAll"
          @playSelected="playSelected"
          @refresh="fetchSongs(true)"
        />
      </template>
    </ScreenHeader>

    <SongListSkeleton v-show="loading"/>
    <SongList
      v-if="!loading && songs.length"
      ref="songList"
      @sort="sort"
      @press:delete="removeSelected"
      @press:enter="onPressEnter"
      @scroll-breakpoint="onScrollBreakpoint"
    />

    <ScreenEmptyState v-if="!songs.length && !loading">
      <template v-slot:icon>
        <icon :icon="faFile"/>
      </template>

      <template v-if="playlist?.is_smart">
        No songs match the playlist's
        <a @click.prevent="editPlaylist">criteria</a>.
      </template>
      <template v-else>
        The playlist is currently empty.
        <span class="d-block secondary">
          Drag songs into its name in the sidebar or use the &quot;Add To…&quot; button to fill it up.
        </span>
      </template>
    </ScreenEmptyState>
  </section>
</template>

<script lang="ts" setup>
import { faFile } from '@fortawesome/free-regular-svg-icons'
import { differenceBy } from 'lodash'
import { ref, toRef, watch } from 'vue'
import { eventBus, pluralize, requireInjection } from '@/utils'
import { commonStore, playlistStore, songStore } from '@/stores'
import { downloadService } from '@/services'
import { usePlaylistManagement, useSongList } from '@/composables'
import { MessageToasterKey, RouterKey } from '@/symbols'

import ScreenHeader from '@/components/ui/ScreenHeader.vue'
import ScreenEmptyState from '@/components/ui/ScreenEmptyState.vue'
import SongListSkeleton from '@/components/ui/skeletons/SongListSkeleton.vue'

const toaster = requireInjection(MessageToasterKey)
const router = requireInjection(RouterKey)

const playlistId = ref<number>()
const playlist = ref<Playlist>()
const loading = ref(false)

const controlsConfig: Partial<SongListControlsConfig> = {
  deletePlaylist: true,
  refresh: true
}

const {
  SongList,
  SongListControls,
  ControlsToggle,
  ThumbnailStack,
  headerLayout,
  songs,
  songList,
  duration,
  thumbnails,
  selectedSongs,
  showingControls,
  isPhone,
  onPressEnter,
  playAll,
  playSelected,
  onScrollBreakpoint,
  sort
} = useSongList(ref<Song[]>([]), 'Playlist')

const { removeSongsFromPlaylist } = usePlaylistManagement()

const allowDownload = toRef(commonStore.state, 'allow_download')

const destroy = () => eventBus.emit('PLAYLIST_DELETE', playlist.value)
const download = () => downloadService.fromPlaylist(playlist.value!)
const editPlaylist = () => eventBus.emit('MODAL_SHOW_EDIT_PLAYLIST_FORM', playlist.value)

const removeSelected = async () => await removeSongsFromPlaylist(playlist.value!, selectedSongs.value)

const fetchSongs = async (refresh = false) => {
  loading.value = true
  songs.value = await songStore.fetchForPlaylist(playlist.value!, refresh)
  loading.value = false
  sort()
}

watch(playlistId, async (id) => {
  if (!id) return

  playlist.value = playlistStore.byId(id)
  playlist.value ? await fetchSongs() : await router.triggerNotFound()
})

router.onRouteChanged(route => route.screen === 'Playlist' && (playlistId.value = parseInt(route.params!.id)))

eventBus.on({
  PLAYLIST_UPDATED: async (updated: Playlist) => updated.id === playlistId.value && await fetchSongs(),
  PLAYLIST_SONGS_REMOVED: async (playlist: Playlist, removed: Song[]) => {
    if (playlist.id !== playlistId.value) return
    songs.value = differenceBy(songs.value, removed, 'id')
  }
})
</script>
