<template>
  <footer id="mainFooter" @contextmenu.prevent="requestContextMenu">
    <AudioPlayer/>

    <div class="wrapper">
      <SongInfo/>
      <PlaybackControls/>
      <ExtraControls/>
    </div>
  </footer>
</template>

<script lang="ts" setup>
import { nextTick, ref, watch } from 'vue'
import { eventBus, isAudioContextSupported, requireInjection } from '@/utils'
import { CurrentSongKey } from '@/symbols'
import { preferenceStore } from '@/stores'
import { audioService, playbackService, volumeManager } from '@/services'

import AudioPlayer from '@/components/layout/app-footer/AudioPlayer.vue'
import SongInfo from '@/components/layout/app-footer/FooterSongInfo.vue'
import ExtraControls from '@/components/layout/app-footer/FooterExtraControls.vue'
import PlaybackControls from '@/components/layout/app-footer/FooterPlaybackControls.vue'

const song = requireInjection(CurrentSongKey, ref(null))

const requestContextMenu = (event: MouseEvent) => {
  song.value && eventBus.emit('SONG_CONTEXT_MENU_REQUESTED', event, song.value)
}

const initPlaybackRelatedServices = async () => {
  const plyrWrapper = document.querySelector<HTMLElement>('.plyr')
  const volumeInput = document.querySelector<HTMLInputElement>('#volumeInput')

  if (!plyrWrapper || !volumeInput) {
    await nextTick()
    await initPlaybackRelatedServices()
    return
  }

  playbackService.init(plyrWrapper)
  volumeManager.init(volumeInput)
  isAudioContextSupported && audioService.init(playbackService.player.media)
}

watch(preferenceStore.initialized, async initialized => {
  if (!initialized) return
  await initPlaybackRelatedServices()
}, { immediate: true })
</script>

<style lang="scss" scoped>
footer {
  background: var(--color-bg-secondary);
  height: var(--footer-height);
  display: flex;
  box-shadow: 0 0 30px 20px rgba(0, 0, 0, .2);
  flex-direction: column;
  position: relative;
  z-index: 1;

  .wrapper {
    display: flex;
    flex: 1;
  }
}
</style>
