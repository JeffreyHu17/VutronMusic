<template>
  <div class="play-page" :class="{ 'no-lyric': noLyric && show === 'lyric' }" :data-theme="theme">
    <button-icon
      v-show="show === 'lyric'"
      class="player-button theme-button"
      @click="setThemeModal = !setThemeModal"
    >
      <SvgIcon icon-class="theme" />
    </button-icon>
    <button-icon class="player-button close-button" @click="showLyrics = !showLyrics">
      <SvgIcon icon-class="arrow-down" />
    </button-icon>
    <div class="lyrics-container">
      <div
        v-if="['blur', 'dynamic'].includes(general.lyricBackground)"
        class="lyrics-background"
        :class="{ 'dynamic-background': general.lyricBackground === 'dynamic' }"
        :style="{ '--cover-url': `url(${pic})` }"
      />
      <div
        v-else-if="general.lyricBackground === 'true'"
        class="graditent-background"
        :style="{ background }"
      />
    </div>
    <div class="left-side">
      <div>
        <div class="cover" :class="{ rotate: currentTheme.name === '旋转封面' }">
          <img ref="imgRef" :src="pic" loading="lazy" />
          <div class="shadow" :style="{ backgroundImage: `url(${pic})` }"></div>
        </div>
        <div class="controls">
          <div class="top-part">
            <div class="track-info">
              <div
                :class="['title', { haslist: hasListSource() }]"
                :title="source"
                @click="hasListSource() && goToList()"
              >
                <span>{{ currentTrack?.name }}</span>
              </div>
              <div class="subtitle">
                <router-link
                  v-if="artist.matched !== false"
                  :to="`/artist/${artist.id}`"
                  @click="showLyrics = !showLyrics"
                  >{{ artist.name }}
                </router-link>
                <span v-else>{{ artist.name }}</span>
                <span>
                  -
                  <router-link
                    v-if="album.matched !== false"
                    :to="`/album/${album.id}`"
                    @click="showLyrics = !showLyrics"
                    >{{ album.name }}
                  </router-link>
                  <span v-else>{{ album.name }}</span>
                </span>
              </div>
            </div>
            <div class="top-right">
              <div class="buttons">
                <div class="transPro" @click="switchTransitionMode">
                  <label
                    v-show="lyrics.tlyric.length"
                    :class="{ active: nTranslationMode === 'tlyric' }"
                    >译</label
                  >
                  <label v-if="lyrics.tlyric.length && lyrics.rlyric.length" class="m-label"
                    >|</label
                  >
                  <label
                    v-show="lyrics.rlyric.length"
                    :class="{ active: nTranslationMode === 'rlyric' }"
                    >音</label
                  >
                </div>
                <button-icon
                  :title="
                    currentTrack?.type === 'local' && currentTrack?.matched === false
                      ? $t('player.noAllowCauseLocal')
                      : $t('player.like')
                  "
                  class="button"
                  :class="{
                    disabled: currentTrack?.type === 'local' && currentTrack?.matched === false
                  }"
                  @click="likeTrack"
                >
                  <SvgIcon :icon-class="isLiked ? 'heart-solid' : 'heart'" />
                </button-icon>
                <button-icon
                  class="button"
                  :title="
                    heartDisabled && show === 'lyric'
                      ? $t('player.noAllowCauseLocal')
                      : show === 'lyric'
                        ? $t('contextMenu.showComment')
                        : $t('contextMenu.showLyric')
                  "
                  :class="{ disabled: currentTrack?.matched === false && show === 'lyric' }"
                  @click="switchRightPage(show === 'lyric' ? 'comment' : 'lyric')"
                >
                  <SvgIcon class="comment" :icon-class="show === 'lyric' ? 'comment' : 'lyric'" />
                </button-icon>
                <button-icon class="button" @click="addTrackToPlaylist"
                  ><SvgIcon icon-class="plus"
                /></button-icon>
                <button-icon class="button" @click="showContextMenu">
                  <SvgIcon icon-class="options" />
                </button-icon>
              </div>
            </div>
          </div>
          <div class="progress-bar">
            <div class="time">{{ formatTime(position) || '0:00' }}</div>
            <div class="slider">
              <vue-slider
                v-model="position"
                :min="0"
                :max="currentTrackDuration"
                :interval="1"
                :duration="0.5"
                :dot-size="12"
                :height="4"
                :marks="marks"
                :use-keyboard="false"
                :drag-on-click="false"
                :step-style="{
                  display: 'block',
                  height: '8px',
                  width: '8px',
                  transform: 'translateY(-2px)',
                  backgroundColor: 'var(--color-text)',
                  opacy: 0.8
                }"
                :rail-style="{ backgroundColor: 'rgba(128, 128, 128, 0.18)' }"
                :process-style="{ backgroundColor: 'var(--color-text)', opacity: 0.8 }"
                :dot-style="{ display: 'none' }"
                tooltip="none"
                :lazy="false"
                :silent="true"
              ></vue-slider>
            </div>
            <div class="time">{{ formatTime(currentTrackDuration) }}</div>
          </div>
          <div class="media-controls">
            <button-icon
              :class="{ active: repeatMode !== 'off' }"
              :title="repeatMode === 'one' ? $t('player.repeatTrack') : $t('player.repeat')"
              @click="switchRepeatMode"
            >
              <svg-icon v-show="repeatMode !== 'one'" icon-class="repeat" />
              <svg-icon v-show="repeatMode === 'one'" icon-class="repeat-1" />
            </button-icon>
            <div class="middle">
              <button-icon v-show="!isPersonalFM" :title="$t('player.previous')" @click="playPrev"
                ><svg-icon icon-class="previous"
              /></button-icon>
              <button-icon
                v-show="isPersonalFM"
                :title="$t('player.moveToFMTrash')"
                @click="moveToFMTrash"
                ><svg-icon icon-class="thumbs-down"
              /></button-icon>
              <button-icon
                id="play"
                :title="$t(playing ? 'player.play' : 'player.pause')"
                @click="playOrPause"
                ><svg-icon :icon-class="playing ? 'pause' : 'play'"
              /></button-icon>
              <button-icon :title="$t('player.next')" @click="_playNextTrack(isPersonalFM)"
                ><svg-icon icon-class="next"
              /></button-icon>
            </div>
            <button-icon :class="{ active: shuffle }" @click="shuffle = !shuffle"
              ><svg-icon icon-class="shuffle"
            /></button-icon>
          </div>
          <div class="progress-bar">
            <div class="time"
              ><button-icon> <svg-icon icon-class="volume-half" /></button-icon
            ></div>
            <div class="slider">
              <VueSlider
                v-model="volume"
                :min="0"
                :max="1"
                :interval="0.01"
                :duration="0.5"
                :dot-size="12"
                :height="4"
                :use-keyboard="false"
                :drag-on-click="false"
                :tooltip-formatter="Math.round(volume * 100).toString()"
                :rail-style="{ backgroundColor: 'rgba(128, 128, 128, 0.18)' }"
                :process-style="{ backgroundColor: 'var(--color-text)', opacity: 0.8 }"
                :dot-style="{
                  display: 'none',
                  backgroundColor: 'var(--color-text)',
                  boxshadow: '0.5px 0.5px 2px 1px rgba(0, 0, 0, 0.18)'
                }"
                tooltip="none"
                :silent="true"
              />
            </div>
            <div class="time">
              <button-icon> <svg-icon icon-class="volume" /></button-icon
            ></div>
          </div>
        </div>
      </div>
    </div>
    <div class="right-side" @mouseenter="hover = true" @mouseleave="hover = false">
      <LyricPage v-if="show === 'lyric'" :text-align="textAlign" :hover="hover" />
      <Comment v-else :id="currentTrack!.id" type="music" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, inject, nextTick, onMounted, ref, toRefs, watch } from 'vue'
import { storeToRefs } from 'pinia'
import { useRouter } from 'vue-router'
import ButtonIcon from './ButtonIcon.vue'
import SvgIcon from './SvgIcon.vue'
import VueSlider from 'vue-3-slider-component'
import LyricPage from './LyricPage.vue'
import Comment from './CommentPage.vue'
import { usePlayerStore } from '../store/player'
import ContextMenu from './ContextMenu.vue'
import { useSettingsStore, TranslationMode } from '../store/settings'
import { hasListSource, getListSourcePath } from '../utils/playlist'
import { useNormalStateStore } from '../store/state'
import { useStreamMusicStore } from '../store/streamingMusic'
import { useDataStore } from '../store/data'
import { Vibrant } from 'node-vibrant/browser'
import Color from 'color'

const router = useRouter()
const playPageContextMenu = inject('playPageContextMenu', ref<InstanceType<typeof ContextMenu>>())

const settingsStore = useSettingsStore()
const { normalLyric, general, playerTheme } = storeToRefs(settingsStore)
const { nTranslationMode, textAlign } = toRefs(normalLyric.value)

const playerStore = usePlayerStore()
const {
  currentTrack,
  seek,
  noLyric,
  lyrics,
  volume,
  currentTrackDuration,
  isPersonalFM,
  playing,
  shuffle,
  isLiked,
  source,
  chorus,
  repeatMode,
  pic
} = storeToRefs(playerStore)
const { playPrev, playOrPause, _playNextTrack, switchRepeatMode, moveToFMTrash } = playerStore

const { likeATrack } = useDataStore()
const { likeAStreamTrack } = useStreamMusicStore()

const stateStore = useNormalStateStore()
const { showLyrics, addTrackToPlaylistModal, setThemeModal } = storeToRefs(stateStore)

const show = ref('lyric')
const imgRef = ref<HTMLElement>()
const color = ref('')
const color2 = ref('')
let animation: Animation | null = null

const currentTheme = computed(() => playerTheme.value.common.find((theme) => theme.selected)!)

const theme = computed(() => {
  return general.value.lyricBackground === 'true' ? 'dark' : 'auto'
})

const background = computed(() => {
  return `linear-gradient(to top left, ${color.value}, ${color2.value})`
})

const heartDisabled = computed(() => {
  return currentTrack.value?.type !== 'online' && !currentTrack.value?.matched
})

const artist = computed(() => {
  return currentTrack.value?.artists ? currentTrack.value.artists[0] : currentTrack.value?.ar[0]
})

const album = computed(() => {
  return currentTrack.value?.album ?? currentTrack.value?.al
})

const marks = computed(() => {
  const result: Record<string, any> = {}
  if (chorus.value === 0) return result
  result[chorus.value.toString()] = { labelStyle: { display: 'none' } }
  return result
})

const tags = computed(() => {
  const lst = ['none']
  if (lyrics.value.tlyric.length) {
    lst.splice(1, 0, 'tlyric')
  }
  if (lyrics.value.rlyric.length) {
    lst.push('rlyric')
  }
  return lst as TranslationMode[]
})

const position = computed({
  get() {
    return seek.value
  },
  set(value) {
    const line = lyrics.value.lyric.find((l, index) => {
      const nextLine = lyrics.value.lyric[index + 1]
      if (nextLine) {
        return nextLine.start > value && l.start <= value
      } else {
        return value >= l.start && value < l.start + 10
      }
    })
    seek.value = line?.start ?? value
  }
})

const idx = ref(tags.value.indexOf(nTranslationMode.value))
const hover = ref(false)

watch(pic, () => {
  getColor()
})

watch(playing, (value) => {
  value ? animation?.play() : animation?.pause()
})

watch(
  currentTheme,
  async (value) => {
    if (value.name === '旋转封面') {
      await nextTick()
      if (!imgRef.value) return
      const effect = new KeyframeEffect(
        imgRef.value!,
        [{ transform: 'rotate(0deg)' }, { transform: 'rotate(360deg)' }],
        { duration: 10000, iterations: Infinity }
      )
      animation = new Animation(effect, document.timeline)
      playing.value ? animation?.play() : animation?.pause()
    } else {
      animation?.cancel()
      animation = null
    }
  },
  { immediate: true }
)

const getColor = () => {
  if (!pic.value) return
  Vibrant.from(pic.value)
    .getPalette()
    .then((palette) => {
      const swatch = palette.DarkMuted
      if (swatch) {
        const originColor = Color.rgb(swatch.rgb)
        color.value = originColor.darken(0.1).rgb().string()
        color2.value = originColor.lighten(0.28).rotate(-30).rgb().string()
      } else {
        console.log('未找到 DarkMuted 颜色')
      }
    })
}

const formatTime = (time: number) => {
  time = Math.round(time)
  const minutes = Math.floor(time / 60)
  const remainingSeconds = Math.ceil(time % 60)
  return `${minutes}:${remainingSeconds.toString().padStart(2, '0')}`
}

const likeTrack = () => {
  if (currentTrack.value?.type === 'stream') {
    const op = currentTrack.value.starred ? 'unstar' : 'star'
    likeAStreamTrack(op, currentTrack.value.id)
  }
  if (currentTrack.value?.matched) {
    likeATrack(currentTrack.value.id)
  }
}

const addTrackToPlaylist = () => {
  if (!currentTrack.value) return
  addTrackToPlaylistModal.value = {
    show: true,
    selectedTrackID: [currentTrack.value.id],
    type: currentTrack.value.type!
  }
}

const switchTransitionMode = () => {
  idx.value = (idx.value + 1) % tags.value.length
  const value = tags.value[idx.value] as TranslationMode
  nTranslationMode.value = value
}

const goToList = () => {
  const path = getListSourcePath()
  router.push(path)
  showLyrics.value = false
}

const showContextMenu = (e: MouseEvent): void => {
  playPageContextMenu.value?.openMenu(e)
}

const switchRightPage = (name: string) => {
  if (show.value === 'comment' || currentTrack.value?.matched) {
    show.value = name
  }
}
onMounted(() => {
  setTimeout(getColor)
})
</script>

<style scoped lang="scss">
.play-page {
  position: fixed;
  top: 0;
  left: 0;
  bottom: 0;
  right: 0;
  z-index: 100;
  color: var(--color-text);
  overflow: hidden;
  display: flex;
  background: var(--color-body-bg);
}

.play-page.no-lyric {
  .left-side {
    flex: 1;
    transition: all 0.35s;
    justify-content: center;
    transform: translateX(25vw);
    padding: 0 calc((50vw - min(50vh, 33.33vw)) / 6);
  }
}

.lyrics-container {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  overflow: hidden;
  z-index: -1;
}

.lyrics-background {
  --contrast-lyrics-background: 50%;
  --brightness-lyrics-background: 130%;
}

[data-theme='dark'] .lyrics-background {
  --contrast-lyrics-background: 105%;
  --brightness-lyrics-background: 60%;
}

.lyrics-background {
  position: absolute;
  height: 100vh;
  width: 100vw;
  filter: blur(50px) contrast(var(--contrast-lyrics-background))
    brightness(var(--brightness-lyrics-background));

  &::before,
  &::after {
    content: '';
    position: absolute;
    width: 140vw;
    height: 140vw;
    background-image: var(--cover-url);
    background-size: cover;
    opacity: 0.6;
    will-change: transform;
  }

  &::before {
    top: 0;
    right: 0;
    mix-blend-mode: luminosity;
  }

  &::after {
    bottom: 0;
    left: 0;
    animation-direction: reverse;
    animation-delay: 10s;
  }
}

.dynamic-background {
  &::before,
  &::after {
    animation: rotate 90s linear infinite;
    will-change: transform;
  }
}

.graditent-background {
  position: absolute;
  width: 100vw;
  height: 100vh;
}

.left-side {
  flex: 1;
  display: flex;
  justify-content: right;
  padding-right: calc((50vw - min(50vh, 33.33vw)) / 6);
  align-items: center;
  transition: all 0.35s;
  z-index: 10;

  .cover {
    position: relative;
    img {
      height: min(50vh, 33.33vw);
      width: min(50vh, 33.33vw);
      user-select: none;
      object-fit: cover;
      border-radius: 0.75rem;
    }
    .shadow {
      position: absolute;
      top: 12px;
      height: min(50vh, 33.33vw);
      width: min(50vh, 33.33vw);
      filter: blur(16px) opacity(0.6);
      transform: scale(0.92, 0.96);
      z-index: -1;
      background-size: cover;
      border-radius: 0.75em;
    }
  }

  .cover.rotate {
    img,
    .shadow {
      border-radius: 50%;
      will-change: transform;
    }
  }

  .controls {
    color: var(--color-text);
    max-width: min(50vh, 33.33vw);
    margin-top: 3vh;
    position: relative;

    .top-part {
      display: flex;
      justify-content: space-between;

      .title {
        // margin-top: 8px;
        font-size: 20px;
        font-weight: 600;
        opacity: 0.88;
        display: -webkit-box;
        -webkit-box-orient: vertical;
        -webkit-line-clamp: 1;
        line-clamp: 1;
        overflow: hidden;
      }
      .haslist {
        cursor: pointer;
        &:hover {
          text-decoration: underline;
        }
      }
      .subtitle {
        // margin-top: 4px;
        font-size: 14px;
        opacity: 0.7;
        display: -webkit-box;
        -webkit-box-orient: vertical;
        -webkit-line-clamp: 1;
        line-clamp: 1;
        overflow: hidden;
      }

      .top-right {
        display: flex;
        align-items: center;
        justify-content: space-between;

        .buttons {
          height: 34px;
          display: flex;
          justify-content: space-between;
          button {
            margin: 0 0 0 4px;
          }
          .svg-icon {
            height: 18px;
            width: 18px;
          }
          .comment {
            height: 22px;
            width: 22px;
          }
        }

        .transPro {
          cursor: pointer;
          line-height: 34px;
          padding: 0 6px;
          margin: 0 2px;
          display: flex;

          label {
            cursor: pointer;
            opacity: 0.5;
          }
          .active {
            opacity: 0.95;
          }
          .m-label {
            margin: 0 2px;
          }
        }
      }
    }
    :deep(.progress-bar) {
      margin-top: 18px;
      display: flex;
      align-items: center;
      justify-content: space-between;

      .slider {
        flex: 1;
        padding: 0 10px;
      }

      .time {
        font-size: 15px;
        font-weight: 600;
        opacity: 0.58;
        width: 34px;
      }
    }
    .media-controls {
      display: flex;
      justify-content: center;
      margin-top: 3vh;
      align-items: center;

      .svg-icon {
        opacity: 0.38;
        height: 14px;
        width: 14px;
      }

      .active .svg-icon {
        opacity: 0.88;
      }

      .middle {
        padding: 0 2.2vw;
        display: flex;
        align-items: center;

        button {
          margin: 0 1.2vw;
        }

        button#play .svg-icon {
          height: 28px;
          width: 28px;
          padding: 2px;
        }

        .svg-icon {
          opacity: 0.88;
          height: 22px;
          width: 22px;
        }
      }
    }
  }
}

.right-side {
  flex: 1;
  justify-self: center;
  padding: 0 calc((50vw - min(50vh, 33.33vw)) / 6);
}

.player-button {
  z-index: 300;
  border-radius: 0.75rem;
  height: 44px;
  width: 44px;
  display: flex;
  justify-content: center;
  align-items: center;
  opacity: 0.48;
  transition: 0.2s;
  -webkit-app-region: no-drag;

  .svg-icon {
    color: var(--color-text);
    height: 22px;
    width: 22px;
  }

  &:hover {
    background: var(--color-secondary-bg-for-transparent);
    opacity: 0.88;
  }
}

.theme-button {
  position: fixed;
  top: 24px;
  right: 74px;
}

.close-button {
  position: fixed;
  top: 24px;
  right: 24px;
}

.button-icon.disabled {
  cursor: default;
  opacity: 0.48;
  &:hover {
    background: none;
  }
  &:active {
    transform: unset;
  }
}

@keyframes rotate {
  from {
    transform: rotate(0deg) translateZ(0);
  } // 触发GPU加速
  to {
    transform: rotate(360deg) translateZ(0);
  }
}

@media (max-aspect-ratio: 10/9) {
  .left-side {
    display: none;
  }
  .right-side {
    max-width: 100%;
  }
  .play-page.no-lyric {
    .left-side {
      display: flex;
    }
  }
}
</style>
