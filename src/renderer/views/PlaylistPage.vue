<template>
  <div v-if="show" class="playlist">
    <div v-if="specialPlaylistInfo === undefined && !isLikedSongsPage" class="playlist-info">
      <Cover
        :id="playlist?.id"
        :image-url="playlist?.coverImgUrl"
        :show-play-button="true"
        :always-show-shadow="true"
        :click-cover-to-play="true"
        :fixed-size="288"
        :type="listType"
        :cover-hover="true"
        :play-button-size="18"
      />
      <div class="info">
        <div class="title" :title="playlist?.name"
          ><span v-if="playlist?.privacy === 10" class="lock-icon"
            ><svg-icon icon-class="lock" /></span
          >{{ playlist?.name }}</div
        >
        <div v-if="playlistType === 'local'" class="artist">
          离线歌单 {{ user.nickname ? `by ${user.nickname}` : `` }}
        </div>
        <div v-else-if="playlistType === 'stream'" class="artist">
          流媒体歌单 by {{ playlist.creator.nickname }}
        </div>
        <div v-else class="artist">
          歌单 by
          <span
            v-if="
              [5277771961, 5277965913, 5277969451, 5277778542, 5278068783].includes(playlist.id)
            "
            style="font-weight: 600"
            >Apple Music</span
          >
          <router-link :to="`/user/${playlist?.creator?.userId}`">{{
            playlist.creator.nickname
          }}</router-link>
        </div>
        <div class="date-and-count">
          {{ $t('library.playlist.updatedAt') }}
          {{ formatDate(playlist?.updateTime) }} · {{ playlist?.trackCount }}
          {{ $t('common.songs') }}
        </div>
        <div class="description" @click="toggleFullDescription">
          {{ playlist?.description }}
        </div>
        <div class="buttons">
          <ButtonTwoTone icon-class="play" @click="play">
            {{ $t('common.play') }}
          </ButtonTwoTone>
          <ButtonTwoTone
            v-if="playlistType === 'online'"
            icon-class="floor-comment"
            @click="openComment"
          >
            {{ '评论' }}
          </ButtonTwoTone>
          <ButtonTwoTone
            v-if="playlistType === 'online' && playlist?.creator?.userId !== user.userId"
            :icon-class="playlist.subscribed ? 'heart-solid' : 'heart'"
            :icon-button="true"
            :horizontal-padding="0"
            :color="playlist.subscribed ? 'var(--color-primary)' : 'grey'"
            :text-color="playlist.subscribed ? 'var(--color-primary)' : ''"
            :background-color="playlist.subscribed ? 'var(--color-secondary-bg)' : ''"
            @click="likePlaylist"
          >
          </ButtonTwoTone>
          <ButtonTwoTone
            v-if="playlistType !== 'online' || playlist?.creator?.userId === user.userId"
            icon-class="more"
            :icon-button="true"
            :horizontal-padding="0"
            color="grey"
            @click="openMenu"
          >
          </ButtonTwoTone>
          <SearchBox ref="pSearchBoxRef" :placeholder="$t('playlist.search')" />
        </div>
      </div>
    </div>

    <div v-if="specialPlaylistInfo !== undefined" class="special-playlist">
      <div class="title" :class="specialPlaylistInfo.gradient">
        {{ specialPlaylistInfo.name }}
      </div>
      <div class="subtitle">{{ playlist.englishTitle }} · {{ playlist.updateFrequency }} </div>

      <div class="buttons">
        <ButtonTwoTone class="play-button" icon-class="play" color="grey" @click="play">
          {{ $t('common.play') }}
        </ButtonTwoTone>
        <ButtonTwoTone color="grey" icon-class="floor-comment" @click="openComment">
          {{ '评论' }}
        </ButtonTwoTone>
        <ButtonTwoTone
          v-if="playlist.creator.userId !== user.userId"
          :icon-class="playlist.subscribed ? 'heart-solid' : 'heart'"
          :icon-button="true"
          :horizontal-padding="0"
          :color="playlist.subscribed ? 'var(--color-primary)' : 'grey'"
          :text-color="playlist.subscribed ? 'var(--color-primary)' : ''"
          :background-color="playlist.subscribed ? 'var(--color-secondary-bg)' : ''"
          @click="likePlaylist"
        >
        </ButtonTwoTone>
        <SearchBox ref="pSearchBoxRef" :placeholder="$t('playlist.search')" />
      </div>
    </div>

    <div v-if="isLikedSongsPage" class="special-playlist">
      <div v-show="$route.name === 'likedSongs'" class="title gradient-green">我喜欢的音乐</div>
      <div v-show="$route.name === 'streamLikedSongs'" class="title gradient-sky-blue"
        >我收藏的流媒体</div
      >
      <div class="buttons">
        <ButtonTwoTone class="play-button" icon-class="play" color="grey" @click="play">
          {{ $t('common.play') }}
        </ButtonTwoTone>
        <ButtonTwoTone
          v-if="$route.name === 'likedSongs'"
          class="play-button"
          icon-class="play"
          color="grey"
          @click="playIntelligenceList"
          >心动模式</ButtonTwoTone
        >
        <ButtonTwoTone
          v-if="$route.name === 'likedSongs'"
          color="grey"
          icon-class="floor-comment"
          @click="openComment"
        >
          {{ '评论' }}
        </ButtonTwoTone>
        <SearchBox ref="pSearchBoxRef" :placeholder="$t('playlist.search')" />
      </div>
    </div>

    <div>
      <TrackList
        :id="playlist?.id"
        :items="filterTracks"
        :type="typeMap[playlistType]"
        :colunm-number="1"
        :show-position="true"
        :load-more="loadMore"
        :extra-context-menu-item="isUserOwnPlaylist ? ['removeTrackFromPlaylist'] : []"
        :is-end="true"
      />
    </div>

    <Modal
      :show="showFullDescription"
      :close-fn="toggleFullDescription"
      :show-footer="false"
      :click-outside-hide="true"
      title="歌单介绍"
      style="white-space: pre-wrap"
      >{{ playlist.description }}</Modal
    >

    <ContextMenu ref="playlistMenu">
      <div
        v-if="playlistType !== 'online' || playlist?.creator?.userId === user.userId"
        class="item"
        @click="deleteAPlaylist"
        >{{ $t('contextMenu.deletePlaylist') }}</div
      >
      <div v-if="playlistType === 'online'" class="item" @click="copyUrl">{{
        $t('contextMenu.copyURL')
      }}</div>
      <div v-if="playlistType === 'online'" class="item" @click="openOnBrowser">{{
        $t('contextMenu.openOnBrowser')
      }}</div>
    </ContextMenu>
  </div>
  <div v-show="showComment" class="comment" @click="closeComment">
    <div></div>
    <div class="comment-container" @click.stop>
      <CommentPage v-if="showComment" :id="playlist.id" type="playlist" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref, provide, onMounted } from 'vue'
import { useDataStore } from '../store/data'
import { Playlist, useLocalMusicStore, Track } from '../store/localMusic'
import { useStreamMusicStore, StreamPlaylist } from '../store/streamingMusic'
import { useNormalStateStore } from '../store/state'
import { usePlayerStore } from '../store/player'
import { storeToRefs } from 'pinia'
import { useRoute, useRouter } from 'vue-router'
import { formatDate, openExternal } from '../utils'
import Cover from '../components/CoverBox.vue'
import CommentPage from '../components/CommentPage.vue'
import ButtonTwoTone from '../components/ButtonTwoTone.vue'
import TrackList from '../components/VirtualTrackList.vue'
import ContextMenu from '../components/ContextMenu.vue'
import SearchBox from '../components/SearchBox.vue'
import SvgIcon from '../components/SvgIcon.vue'
import Modal from '../components/BaseModal.vue'
import { isAccountLoggedIn } from '../utils/auth'
import { tricklingProgress } from '../utils/tricklingProgress'
import { useI18n } from 'vue-i18n'
import { getTrackDetail } from '../api/track'
import {
  getPlaylistDetail,
  subscribePlaylist,
  intelligencePlaylist,
  deletePlaylist
} from '../api/playlist'

const specialPlaylist = {
  2829816518: {
    name: '欧美私人订制',
    gradient: 'gradient-pink-purple-blue'
  },
  2890490211: {
    name: '助眠鸟鸣声',
    gradient: 'gradient-green'
  },
  5089855855: {
    name: '夜的胡思乱想',
    gradient: 'gradient-moonstone-blue'
  },
  2888212971: {
    name: '全球百大DJ',
    gradient: 'gradient-orange-red'
  },
  2829733864: {
    name: '睡眠伴侣',
    gradient: 'gradient-midnight-blue'
  },
  2829844572: {
    name: '洗澡时听的歌',
    gradient: 'gradient-yellow'
  },
  2920647537: {
    name: '还是会想你',
    gradient: 'gradient-dark-blue-midnight-blue'
  },
  2890501416: {
    name: '助眠白噪声',
    gradient: 'gradient-sky-blue'
  },
  5217150082: {
    name: '摇滚唱片行',
    gradient: 'gradient-yellow-red'
  },
  2829961453: {
    name: '古风音乐大赏',
    gradient: 'gradient-fog'
  },
  4923261701: {
    name: 'Trance',
    gradient: 'gradient-light-red-light-blue '
  },
  5212729721: {
    name: '欧美点唱机',
    gradient: 'gradient-indigo-pink-yellow'
  },
  3103434282: {
    name: '甜蜜少女心',
    gradient: 'gradient-pink'
  },
  2829896389: {
    name: '日系私人订制',
    gradient: 'gradient-yellow-pink'
  },
  2829779628: {
    name: '运动随身听',
    gradient: 'gradient-orange-red'
  },
  2860654884: {
    name: '独立女声精选',
    gradient: 'gradient-sharp-blue'
  },
  898150: {
    name: '浪漫婚礼专用',
    gradient: 'gradient-pink'
  },
  2638104052: {
    name: '牛奶泡泡浴',
    gradient: 'gradient-fog'
  },
  5317236517: {
    name: '后朋克精选',
    gradient: 'gradient-pink-purple-blue'
  },
  2821115454: {
    name: '一周原创发现',
    gradient: 'gradient-blue-purple'
  },
  2829883282: {
    name: '华语私人雷达',
    gradient: 'gradient-yellow-red'
  },
  3136952023: {
    name: '私人雷达',
    gradient: 'gradient-radar'
  }
}

const route = useRoute()
const router = useRouter()
const playlist = ref<{ [key: string]: any }>({
  id: 0,
  name: '',
  description: '',
  updateTime: 0,
  trackCount: 0,
  creator: { userId: '' },
  coverImgUrl: '',
  trackIds: []
})
const tracks = ref<Track[]>([])
const playlistMenu = ref()
const show = ref(false)
const lastLoadedTrackIndex = ref(9)
const showFullDescription = ref(false)
const showComment = ref(false)
const pSearchBoxRef = ref<InstanceType<typeof SearchBox>>()

const { user, likedSongPlaylistID } = storeToRefs(useDataStore())
const listType = computed(() => route.name!.toString())
const specialPlaylistInfo = computed(() => specialPlaylist[playlist.value?.id])
const keyword = computed(() => pSearchBoxRef.value?.keywords || '')
const filterTracks = computed(() => {
  return tracks.value.filter(
    (track) =>
      (track.name && track.name.toLowerCase().includes(keyword.value?.toLowerCase())) ||
      (track.alia || track.alias)?.find((al) =>
        al.toLowerCase().includes(keyword.value?.toLowerCase())
      ) ||
      (track.album?.name &&
        track.album.name.toLowerCase().includes(keyword.value?.toLowerCase())) ||
      (track.artists || track.ar).find(
        (ar) => ar.name && ar.name.toLowerCase().includes(keyword.value?.toLowerCase())
      )
  )
})

const { playlists, localTracks } = storeToRefs(useLocalMusicStore())
const { deleteLocalPlaylist } = useLocalMusicStore()

const streamMusic = useStreamMusicStore()
const { showToast } = useNormalStateStore()

const playerStore = usePlayerStore()
const { _shuffle } = storeToRefs(playerStore)
const { replacePlaylist } = playerStore

const { t } = useI18n()

const playlistType = computed(() => {
  if (route.name === 'localPlaylist') {
    return 'local'
  } else if (route.name === 'streamPlaylist') {
    return 'stream'
  } else if (route.name === 'streamLikedSongs') {
    return 'streamLiked'
  } else {
    return 'online'
  }
})

const typeMap = {
  local: 'localPlaylist',
  stream: 'streamPlaylist',
  streamLiked: 'streamLiked',
  online: 'playlist'
}

const isLikedSongsPage = computed(
  () => route.name === 'likedSongs' || route.name === 'streamLikedSongs'
)

const isUserOwnPlaylist = computed(() => {
  return (
    playlistType.value !== 'online' ||
    (playlist.value?.creator?.userId === user.value?.userId &&
      playlist.value?.id !== likedSongPlaylistID.value)
  )
})

const loadLocalData = (id: number) => {
  playlist.value = playlists.value.find((item) => item.id === id) as Playlist
  if (!playlist.value) {
    router.go(-1)
    return
  }
  const trackIDs = playlist.value.trackIds
  tracks.value = trackIDs
    .map((id) => localTracks.value.find((item) => item.id === id) as Track)
    .reverse()
  tricklingProgress.done()
  show.value = true
}

const loadStreamData = (id: string) => {
  playlist.value = streamMusic.playlists.find((p) => p.id === id) as StreamPlaylist
  if (!playlist.value) {
    router.go(-1)
    return
  }
  const trackIDs = playlist.value.trackIds
  tracks.value = trackIDs
    .map((id) => streamMusic.streamTracks.find((item) => item.id === id))
    .map((track) => {
      if (!playlist.value.trackItemIds) return track
      return { ...track, playlistItemId: playlist.value.trackItemIds[track.id] }
    })
  tricklingProgress.done()
  show.value = true
}

const loadStreamLiked = () => {
  tracks.value = streamMusic.streamLikedTracks
  tricklingProgress.done()
  show.value = true
}

const loadData = async (id: number) => {
  await getPlaylistDetail(id, true)
    .then((data: any) => {
      playlist.value = data.playlist
      tracks.value = data.playlist.tracks
      lastLoadedTrackIndex.value = data.playlist.tracks.length - 1
      tricklingProgress.done()
      show.value = true
    })
    .then(() => {
      if (playlist.value.trackCount > tracks.value.length) {
        const trackIDs = playlist.value.trackIds
          .slice(tracks.value.length, tracks.value.length + 500)
          .map((t) => t.id)
        getTrackDetail(trackIDs.join(',')).then((data: any) => {
          tracks.value.push(...data.songs)
        })
      }
    })
}

const loadMore = (Num: number = 500) => {
  if (playlist.value.trackCount > tracks.value.length) {
    const trackIDs = playlist.value.trackIds
      .slice(tracks.value.length, tracks.value.length + Num)
      .map((t) => t.id)
    getTrackDetail(trackIDs.join(',')).then((data: any) => {
      tracks.value.push(...data.songs)
      lastLoadedTrackIndex.value = tracks.value.length - 1
    })
  }
}

const likePlaylist = (toast = false) => {
  if (!isAccountLoggedIn()) {
    showToast(t('toast.needToLogin'))
    return
  }
  subscribePlaylist({ id: playlist.value.id, t: playlist.value.subscribed ? 2 : 1 }).then(
    (data) => {
      if (data.code === 200) {
        playlist.value.subscribed = !playlist.value.subscribed
        if (toast) {
          showToast(playlist.value.subscribed ? '已保存到音乐库' : '已从音乐库删除')
        }
      }
      getPlaylistDetail(playlist.value.id, true).then((data: any) => {
        playlist.value = data.playlist
      })
    }
  )
}

const play = () => {
  const trackIDs = tracks.value.map((t) => t.id)
  const idx = _shuffle.value ? Math.floor(Math.random() * trackIDs.length) : 0
  replacePlaylist(typeMap[playlistType.value], playlist.value.id || 0, trackIDs, idx)
}

const playIntelligenceList = () => {
  const randomId = Math.floor(Math.random() * tracks.value.length + 1)
  const songId = tracks.value[randomId].id
  intelligencePlaylist({ id: songId, pid: likedSongPlaylistID.value }).then((result) => {
    const trackIDs = result.data.map((t: any) => t.id)
    const idx = _shuffle.value ? Math.floor(Math.random() * trackIDs.length) : 0
    replacePlaylist('playlist', likedSongPlaylistID.value, trackIDs, idx)
  })
}

const toggleFullDescription = () => {
  showFullDescription.value = !showFullDescription.value
}

const openMenu = (e: MouseEvent) => {
  playlistMenu.value.openMenu(e)
}

const deleteAPlaylist = () => {
  if (playlistType.value === 'online' && !isAccountLoggedIn()) {
    showToast(t('toast.needToLogin'))
    return
  }

  if (confirm(`确定要删除歌单 ${playlist.value.name}？`)) {
    if (playlistType.value === 'local') {
      deleteLocalPlaylist(playlist.value.id).then((result) => {
        if (result) {
          show.value = false
          playlist.value = {
            id: 0,
            name: '',
            description: '',
            updateTime: 0,
            trackCount: 0,
            creator: { userId: '' },
            coverImgUrl: '',
            trackIds: []
          }
          showToast(t('toast.deleteSuccess'))
          router.go(-1)
        } else {
          showToast(t('toast.deleteFailed'))
        }
      })
    } else if (playlistType.value === 'stream') {
      window.mainApi
        ?.invoke('deleteStreamPlaylist', {
          id: playlist.value.id,
          platform: streamMusic.currentService.name
        })
        .then((result: boolean) => {
          if (result) {
            show.value = false
            playlist.value = {
              id: 0,
              name: '',
              description: '',
              updateTime: 0,
              trackCount: 0,
              creator: { userId: '' },
              coverImgUrl: '',
              trackIds: []
            }
            showToast(t('toast.deleteSuccess'))
            router.go(-1)
          } else {
            showToast(t('toast.deleteFailed'))
          }
        })
    } else {
      deletePlaylist(playlist.value.id).then((result) => {
        if (result.code === 200) {
          showToast(t('toast.deleteSuccess'))
          router.go(-1)
        } else {
          showToast(t('toast.deleteFailed'))
        }
      })
    }
  }
}

const copyUrl = () => {
  const url = `https://music.163.com/#/playlist?id=${playlist.value.id}`
  navigator.clipboard.writeText(url).then(() => {
    showToast(t('toast.copySuccess'))
  })
}

const openOnBrowser = () => {
  const url = `https://music.163.com/#/playlist?id=${playlist.value.id}`
  openExternal(url)
}

const openComment = () => {
  showComment.value = true
}

const closeComment = () => {
  showComment.value = false
}

const removeTrack = (idx: number) => {
  tracks.value.splice(idx, 1)
  if (playlistType.value === 'stream') {
    playlist.value.trackCount -= 1
  }
}

provide('removeTrack', removeTrack)

onMounted(() => {
  if (playlistType.value === 'local') {
    loadLocalData(Number(route.params.id))
  } else if (playlistType.value === 'stream') {
    loadStreamData(route.params.id as string)
  } else if (route.name === 'streamLikedSongs') {
    loadStreamLiked()
  } else if (route.name === 'likedSongs') {
    loadData(likedSongPlaylistID.value)
  } else {
    loadData(Number(route.params.id))
  }
  setTimeout(() => {
    if (!show.value) tricklingProgress.start()
  }, 1000)
})
</script>

<style lang="scss" scoped>
.playlist-info {
  display: flex;
  margin-bottom: 72px;
  position: relative;
  .info {
    display: flex;
    flex-direction: column;
    justify-content: center;
    flex: 1;
    margin-left: 56px;
    .title {
      font-size: 36px;
      font-weight: 700;
      white-space: pre-wrap;
      display: -webkit-box;
      -webkit-box-orient: vertical;
      -webkit-line-clamp: 1;
      line-clamp: 1;
      overflow: hidden;
      color: var(--color-text);

      .lock-icon {
        opacity: 0.28;
        color: var(--color-text);
        margin-right: 8px;
        .svg-icon {
          height: 26px;
          width: 26px;
        }
      }
    }
    .artist {
      font-size: 18px;
      opacity: 0.88;
      color: var(--color-text);
      margin-top: 24px;
    }
    .date-and-count {
      font-size: 14px;
      opacity: 0.68;
      color: var(--color-text);
      margin-top: 2px;
    }
    .description {
      white-space: pre-wrap;
      font-size: 14px;
      opacity: 0.68;
      color: var(--color-text);
      margin-top: 24px;
      display: -webkit-box;
      -webkit-box-orient: vertical;
      -webkit-line-clamp: 3;
      line-clamp: 3;
      overflow: hidden;
      cursor: pointer;
      &:hover {
        transition: opacity 0.3s;
        opacity: 0.88;
      }
    }
    .buttons {
      margin-top: 32px;
      display: flex;
      align-items: center;
      button {
        margin-right: 16px;
      }
    }
  }
}

.special-playlist {
  padding: 192px 0 128px 0;
  border-radius: 1.25em;
  text-align: center;

  @keyframes letterSpacing4 {
    from {
      letter-spacing: 0px;
    }

    to {
      letter-spacing: 4px;
    }
  }

  @keyframes letterSpacing1 {
    from {
      letter-spacing: 0px;
    }

    to {
      letter-spacing: 1px;
    }
  }

  .title {
    font-size: 84px;
    line-height: 1.05;
    font-weight: 700;
    text-transform: uppercase;

    letter-spacing: 4px;
    animation-duration: 0.8s;
    animation-name: letterSpacing4;
    -webkit-text-fill-color: transparent;
    background-clip: text;

    img {
      height: 78px;
      border-radius: 0.125em;
      margin-right: 24px;
    }
  }
  .subtitle {
    font-size: 18px;
    letter-spacing: 1px;
    margin: 28px 0 54px 0;
    animation-duration: 0.8s;
    animation-name: letterSpacing1;
    text-transform: uppercase;
    color: var(--color-text);
  }
  .buttons {
    margin-top: 32px;
    display: flex;
    justify-content: center;
    align-items: center;
    button {
      margin-right: 16px;
    }
  }
}

.gradient-test {
  background-image: linear-gradient(to left, #92fe9d 0%, #00c9ff 100%);
}

[data-theme='dark'] {
  .gradient-radar {
    background-image: linear-gradient(to left, #92fe9d 0%, #00c9ff 100%);
  }
}

.gradient-radar {
  background-image: linear-gradient(to left, #0ba360 0%, #3cba92 100%);
}

.gradient-blue-purple {
  background-image: linear-gradient(45deg, #89c4f5 0%, #6284ff 42%, #ff0000 100%);
}

.gradient-sharp-blue {
  background-image: linear-gradient(45deg, #00c6fb 0%, #005bea 100%);
}

.gradient-yellow-pink {
  background-image: linear-gradient(45deg, #f6d365 0%, #fda085 100%);
}

.gradient-pink {
  background-image: linear-gradient(45deg, #ee9ca7 0%, #ffdde1 100%);
}

.gradient-indigo-pink-yellow {
  background-image: linear-gradient(43deg, #4158d0 0%, #c850c0 46%, #ffcc70 100%);
}

.gradient-light-red-light-blue {
  background-image: linear-gradient(
    225deg,
    hsl(190, 30%, 50%) 0%,
    #081abb 38%,
    #ec3841 58%,
    hsl(13, 99%, 49%) 100%
  );
}

.gradient-fog {
  background:
    linear-gradient(-180deg, #bcc5ce 0%, #929ead 98%),
    radial-gradient(at top left, rgba(255, 255, 255, 0.3) 0%, rgba(0, 0, 0, 0.3) 100%);
  background-blend-mode: screen;
}

.gradient-red {
  background-image: linear-gradient(213deg, #ff0844 0%, #ffb199 100%);
}

.gradient-sky-blue {
  background-image: linear-gradient(147deg, #48c6ef 0%, #6f86d6 100%);
}

.gradient-dark-blue-midnight-blue {
  background-image: linear-gradient(213deg, #09203f 0%, #537895 100%);
}

.gradient-yellow-red {
  background: linear-gradient(147deg, #fec867 0%, #f72c61 100%);
}

.gradient-yellow {
  background: linear-gradient(147deg, #fceb02 0%, #fec401 100%);
}

.gradient-midnight-blue {
  background-image: linear-gradient(-20deg, #2b5876 0%, #4e4376 100%);
}

.gradient-orange-red {
  background-image: linear-gradient(147deg, #ffe53b 0%, #ff2525 74%);
}

.gradient-moonstone-blue {
  background-image: linear-gradient(
    147deg,
    hsl(200, 34%, 8%) 0%,
    hsl(204, 35%, 38%) 50%,
    hsl(200, 34%, 18%) 100%
  );
}

.gradient-pink-purple-blue {
  background-image: linear-gradient(to right, #ff3cac 0%, #784ba0 50%, #2b86c5 100%) !important;
}

.gradient-green {
  background-image: linear-gradient(90deg, #c6f6d5, #68d391, #38b2ac) !important;
}

.user-info {
  h1 {
    font-size: 42px;
    position: relative;
    color: var(--color-text);
    .avatar {
      height: 44px;
      margin-right: 12px;
      vertical-align: -7px;
      border-radius: 50%;
      border: rgba(0, 0, 0, 0.2);
    }
  }
}

.search-box-likepage {
  display: flex;
  position: absolute;
  right: 12vw;
  top: 95px;
  justify-content: flex-end;
  -webkit-app-region: no-drag;

  .input {
    transition: all 0.5s;
  }

  .container {
    display: flex;
    align-items: center;
    height: 32px;
    background: var(--color-secondary-bg-for-transparent);
    border-radius: 8px;
  }

  .svg-icon {
    height: 15px;
    width: 15px;
    color: var(--color-text);
    opacity: 0.28;
    margin: {
      left: 8px;
      right: 8px;
    }
  }

  input {
    font-size: 16px;
    border: none;
    background: transparent;
    width: 96%;
    font-weight: 600;
    margin-top: -1px;
    color: var(--color-text);
  }

  .active {
    background: color-mix(in oklab, var(--color-primary) var(--bg-alpha), white);
    input,
    .svg-icon {
      opacity: 1;
      color: var(--color-primary);
    }
  }
}

[data-theme='dark'] {
  .search-box-likepage {
    .active {
      input,
      .svg-icon {
        color: var(--color-text);
      }
    }
  }
}

@media (max-width: 1336px) {
  .search-box-likepage {
    right: 8vw;
  }
}

.load-more {
  display: flex;
  justify-content: center;
  margin-top: 32px;
}

.comment {
  background-color: rgba(0, 0, 0, 0.38);
  position: fixed;
  top: 0;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 1000;
  display: grid;
  grid-template-columns: repeat(2, 1fr);
}

.comment-container {
  background-color: var(--color-body-bg);
}
</style>
