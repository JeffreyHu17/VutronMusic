<template>
  <div class="login">
    <div class="login-container">
      <div class="section-1">
        <img src="../assets/images/vutronmusic-icon.png" />
        <svg-icon icon-class="x"></svg-icon>
        <img src="../assets/images/netease-music.png" />
      </div>
      <div class="title">{{ $t('login.loginText') }}</div>
      <div class="section-2">
        <div v-show="selectedMode.mode === 'phone'" class="input-box">
          <div class="container" :class="{ active: inputFocus === 'phone' }">
            <svg-icon icon-class="mobile" />
            <div class="inputs">
              <input
                id="countryCode"
                v-model="countryCode"
                :placeholder="inputFocus === 'phone' ? '' : $t('login.countryCode')"
                @focus="inputFocus = 'phone'"
                @blur="inputFocus = ''"
                @keyup.enter="login"
              />
              <input
                id="phoneNumber"
                v-model="phoneNumber"
                :placeholder="inputFocus === 'phone' ? '' : $t('login.phone')"
                @focus="inputFocus = 'phone'"
                @blur="inputFocus = ''"
                @keyup.enter="login"
              />
            </div>
          </div>
        </div>

        <div v-show="selectedMode.mode === 'email'" class="input-box">
          <div class="container" :class="{ active: inputFocus === 'email' }">
            <svg-icon icon-class="mail" />
            <div class="inputs">
              <input
                id="email"
                v-model="email"
                type="email"
                :placeholder="inputFocus === 'email' ? '' : $t('login.email')"
                @focus="inputFocus = 'email'"
                @blur="inputFocus = ''"
                @keyup.enter="login"
              />
            </div>
          </div>
        </div>

        <div v-show="selectedMode.mode === 'cookie'" class="input-box">
          <div class="container text-container">
            <textarea
              v-model="inputCookie"
              class="cookie-input"
              placeholder="请输入cookie"
            ></textarea>
          </div>
        </div>

        <div v-show="['phone', 'email'].includes(selectedMode.mode)" class="input-box">
          <div class="container" :class="{ active: inputFocus === 'password' }">
            <svg-icon icon-class="lock" />
            <div class="inputs">
              <input
                id="password"
                v-model="password"
                type="password"
                :placeholder="inputFocus === 'password' ? '' : $t('login.password')"
                @focus="inputFocus = 'password'"
                @blur="inputFocus = ''"
                @keyup.enter="login"
              />
            </div>
          </div>
        </div>

        <div v-show="selectedMode.mode == 'qrCode'">
          <div v-show="qrCodeSvg" class="qr-code-container">
            <img :src="qrCodeSvg" loading="lazy" />
          </div>
          <div class="qr-code-info">
            {{ qrCodeInformation }}
          </div>
        </div>
      </div>
      <div v-show="selectedMode.mode !== 'qrCode'" class="confirm">
        <button v-show="!processing" @click="login">
          {{ $t('login.login') }}
        </button>
      </div>
      <div class="other-login">
        <a
          v-for="mode in [...loginModes.filter((m) => !m.selected)]"
          :key="mode.mode"
          @click="changeMode(mode.mode)"
          >{{ mode.text }}</a
        >
      </div>
      <div v-show="['phone', 'email'].includes(selectedMode.mode)" class="notice">{{
        $t('login.noticeElectron')
      }}</div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, computed } from 'vue'
import SvgIcon from '../components/SvgIcon.vue'
import { loginQrCodeKey, loginQrCodeCheck, getLoginStatus } from '../api/auth'
import qrCode from 'qrcode'
import { useDataStore } from '../store/data'
import { useSettingsStore } from '../store/settings'
import { storeToRefs } from 'pinia'
import { useRoute, useRouter } from 'vue-router'
import { setCookies, setCookiesWithCookie } from '../utils/auth'
import { useI18n } from 'vue-i18n'
import _ from 'lodash'

const router = useRouter()
const route = useRoute()
const { theme } = storeToRefs(useSettingsStore())
const { t } = useI18n()

const modeList = ['phone', 'email', 'qrCode', 'cookie'] as const

const loginModes = ref([
  { mode: 'phone' as (typeof modeList)[number], selected: false, text: t('login.loginWithPhone') },
  { mode: 'email' as (typeof modeList)[number], selected: false, text: t('login.loginWithEmail') },
  { mode: 'qrCode' as (typeof modeList)[number], selected: true, text: t('login.loginWithQrCode') },
  { mode: 'cookie' as (typeof modeList)[number], selected: false, text: t('login.loginWithCookie') }
])

const selectedMode = computed(() => loginModes.value.find((M) => M.selected)!)

const processing = ref(false)
const inputFocus = ref('')
const countryCode = ref('+86')
const phoneNumber = ref('')
const email = ref('')
const password = ref('')
const qrCodeSvg = ref('')
const qrCodeKey = ref('')
const qrCodeCheckInterval = ref<any>(null)
const qrCodeInformation = ref('打开网易云音乐APP扫码登录')
const inputCookie = ref('')

const dataStore = useDataStore()
const { user } = storeToRefs(dataStore)

const selectedColor = computed(() => {
  const color = theme.value.colors.find((c) => c.selected)?.color || 'rgba(51, 94, 234, 1)'
  const parts = color.startsWith('rgba')
    ? color.slice(5, -1).split(',')
    : color.slice(4, -1).split(',')
  const r = parseInt(parts[0].trim(), 10)
  const g = parseInt(parts[1].trim(), 10)
  const b = parseInt(parts[2].trim(), 10)

  const red = Math.min(255, Math.max(0, r)).toString(16).padStart(2, '0')
  const green = Math.min(255, Math.max(0, g)).toString(16).padStart(2, '0')
  const blue = Math.min(255, Math.max(0, b)).toString(16).padStart(2, '0')

  return `#${red}${green}${blue}`
})

const login = () => {
  if (selectedMode.value.mode === 'cookie') {
    doCookieLogin()
  }
}

const checkQrCodeLogin = () => {
  qrCodeCheckInterval.value = setInterval(() => {
    if (qrCodeKey.value === '') return
    loginQrCodeCheck(qrCodeKey.value).then((result) => {
      if (result.code === 800) {
        getQrCodeKey()
        qrCodeInformation.value = '二维码已失效，请重新获取'
      } else if (result.code === 802) {
        qrCodeInformation.value = '扫描成功，请在手机上确认登录'
      } else if (result.code === 801) {
        qrCodeInformation.value = '打开网易云音乐APP扫码登录'
      } else if (result.code === 803) {
        clearInterval(qrCodeCheckInterval.value)
        qrCodeInformation.value = '登录成功，请稍等...'
        result.code = 200
        result.cookie = result.cookie.replaceAll(' HTTPOnly', '')
        handleLoginResponse(result)
      }
    })
  }, 3000)
}

const doCookieLogin = () => {
  const result = { code: 200, cookie: inputCookie.value }
  handleLoginResponse(result, true)
}

const changeMode = (md: (typeof modeList)[number]) => {
  loginModes.value.forEach((loginMode) => {
    loginMode.selected = loginMode.mode === md
  })
  if (md === 'qrCode') {
    checkQrCodeLogin()
  } else {
    clearInterval(qrCodeCheckInterval.value)
  }
}

const handleLoginResponse = (data: any, isCookie = false) => {
  if (!data) {
    processing.value = false
    return
  }
  if (data.code === 200) {
    if (isCookie) {
      setCookiesWithCookie(data.cookie)
    } else {
      setCookies(data.cookie)
    }
    const cookie = localStorage.getItem('cookie-MUSIC_U') || ''
    getLoginStatus(cookie).then((res) => {
      _.merge(user.value, res.data.profile)
      router.push('/')
    })
  }
}

const getQrCodeKey = () => {
  return loginQrCodeKey().then((res) => {
    if (res.code === 200) {
      qrCodeKey.value = res.data.unikey
      qrCode
        .toString(`https://music.163.com/login?codekey=${qrCodeKey.value}`, {
          width: 192,
          margin: 0,
          color: {
            dark: selectedColor.value,
            light: '#00000000'
          },
          type: 'svg'
        })
        .then((svg: string) => {
          qrCodeSvg.value = `data:image/svg+xml;utf8,${encodeURIComponent(svg)}`
        })
        .catch((err: any) => {
          console.log('err = ', err)
        })
    }
    checkQrCodeLogin()
  })
}

onMounted(() => {
  if (['phone', 'email', 'qrCode', 'cookie'].includes(route.query.mode as string)) {
    loginModes.value.forEach((M) => {
      M.selected = M.mode === (route.query.mode as 'qrCode' | 'phone' | 'email' | 'cookie')
    })
  }
  if (selectedMode.value.mode === 'qrCode') getQrCodeKey()
})
onBeforeUnmount(() => {
  clearInterval(qrCodeCheckInterval.value)
})
</script>

<style lang="scss" scoped>
.login {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  margin-top: 32px;
  color: var(--color-text);
}

.login-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.title {
  font-size: 24px;
  font-weight: 700;
  margin-bottom: 48px;
}

.section-1 {
  margin-bottom: 16px;
  display: flex;
  align-items: center;
  img {
    height: 64px;
    margin: 20px;
    user-select: none;
  }
}

.section-2 {
  display: flex;
  align-items: center;
  flex-direction: column;
}

.input-box {
  display: flex;
  justify-content: flex-end;
  margin-bottom: 16px;
  color: var(--color-text);

  .container {
    display: flex;
    align-items: center;
    height: 46px;
    background: var(--color-secondary-bg);
    border-radius: 8px;
    width: 300px;
  }

  .text-container {
    height: 108px !important;
    box-sizing: border-box;
  }

  .svg-icon {
    height: 18px;
    width: 18px;
    color: #aaaaaa;
    margin: 0 6px 0 12px;
  }

  .inputs {
    display: flex;
    width: 85%;
  }

  input {
    font-size: 20px;
    border: none;
    background: transparent;
    width: 100%;
    font-weight: 600;
    margin-top: -1px;
    color: var(--color-text);
  }

  .cookie-input {
    font-size: 16px;
    border: none;
    background: transparent;
    width: 100%;
    height: 100%;
    font-weight: 600;
    margin-top: -1px;
    color: var(--color-text);
    box-sizing: border-box;
    resize: none;
    border: none;
    outline: none;
    border-radius: 8px 0 0 8px;
    padding: 12px 20px;
    scrollbar-width: none;
  }

  .cookie-input::placeholder {
    color: var(--color-text);
    opacity: 0.38;
    font-size: 20px;
  }
  input::placeholder {
    color: var(--color-text);
    opacity: 0.38;
  }

  input#countryCode {
    flex: 3;
  }
  input#phoneNumber {
    flex: 12;
  }

  .active {
    background: color-mix(in oklab, var(--color-primary) var(--bg-alpha), white);
    input,
    .svg-icon {
      color: var(--color-primary);
    }
  }
}

.confirm button {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  font-weight: 600;
  background: color-mix(in oklab, var(--color-primary) var(--bg-alpha), white);
  color: var(--color-primary);
  border-radius: 8px;
  margin-top: 24px;
  transition: 0.2s;
  padding: 8px;
  width: 100%;
  width: 300px;
  &:hover {
    transform: scale(1.06);
  }
  &:active {
    transform: scale(0.94);
  }
}

.other-login {
  margin-top: 24px;
  font-size: 13px;
  opacity: 0.68;
  a {
    padding: 0 8px;
    border-right: 2px solid var(--color-text);

    &:last-child {
      border-right: 0;
    }
  }
}

.notice {
  width: 300px;
  border-top: 1px solid rgba(128, 128, 128);
  margin-top: 48px;
  padding-top: 12px;
  font-size: 12px;
  opacity: 0.48;
}

@keyframes loading {
  0% {
    opacity: 0.2;
  }
  20% {
    opacity: 1;
  }
  100% {
    opacity: 0.2;
  }
}

button.loading {
  height: 44px;
  cursor: unset;
  &:hover {
    transform: none;
  }
}
.loading span {
  width: 6px;
  height: 6px;
  background-color: var(--color-primary);
  border-radius: 50%;
  margin: 0 2px;
  animation: loading 1.4s infinite both;
}

.loading span:nth-child(2) {
  animation-delay: 0.2s;
}

.loading span:nth-child(3) {
  animation-delay: 0.4s;
}

.qr-code-container {
  background: color-mix(in oklab, var(--color-primary) var(--bg-alpha), white);
  padding: 24px 24px 21px 24px;
  border-radius: 1.25rem;
  margin-bottom: 12px;
}
.qr-code-info {
  text-align: center;
  margin-bottom: 28px;
}
</style>
