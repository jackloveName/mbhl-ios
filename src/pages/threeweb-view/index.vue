<template>
  <view class="web-page">
    <web-view
      v-if="url"
      :src="url"
      :progress="false"
    />
  </view>
</template>

<script setup>
import { getCurrentInstance, ref } from 'vue'
import { onBackPress, onLoad, onReady, onUnload } from '@dcloudio/uni-app'
import { toast } from '@/utils/common'

const CloseSentinel = 'https://mbhl.local/close-webview'
const instance = getCurrentInstance()
const url = ref('')
let returnTimer = null
let pinTimer = null
let childWebview = null
let childEventsBound = false

function isSafeWebUrl (value) {
  return /^https?:\/\//i.test(String(value || '').trim())
}

function isExternalUrl (value) {
  return /apps\.apple\.com|open_external=true|\.(?:apk|plist)(?:$|[?#])/i.test(value)
}

function returnToAgreement (delay = 0) {
  clearTimeout(returnTimer)
  returnTimer = setTimeout(() => {
    uni.reLaunch({ url: '/pages/agreement/index' })
  }, delay)
}

function openExternalUrl (target) {
  // #ifdef APP-PLUS
  plus.runtime.openURL(target, () => toast('无法打开外部链接'))
  // #endif

  // #ifdef H5
  window.open(target, '_blank', 'noopener,noreferrer')
  // #endif

  returnToAgreement(500)
}

function getPageWebview () {
  // #ifdef APP-PLUS
  try {
    const scope = instance && instance.proxy && instance.proxy.$scope
    if (scope && typeof scope.$getAppWebview === 'function') {
      return scope.$getAppWebview()
    }
  } catch (error) {}

  try {
    const pages = getCurrentPages()
    const page = pages[pages.length - 1]
    if (page && typeof page.$getAppWebview === 'function') {
      return page.$getAppWebview()
    }
  } catch (error) {}

  try {
    return plus.webview.currentWebview()
  } catch (error) {}
  // #endif
  return null
}

function getChildWebview () {
  if (childWebview) return childWebview
  const current = getPageWebview()
  const wv = current && typeof current.children === 'function' ? current.children()[0] : null
  if (wv) childWebview = wv
  return wv
}

function handleWebViewBack () {
  // #ifdef APP-PLUS
  const wv = getChildWebview()
  if (wv && typeof wv.canBack === 'function') {
    wv.canBack((result) => {
      if (result && result.canBack) wv.back()
      else returnToAgreement()
    })
    return
  }
  // #endif
  returnToAgreement()
}

function injectBackBridge (wv) {
  if (!wv) return
  wv.evalJS(`(function () {
    if (window.__mbhlBackBridge) return;
    window.__mbhlBackBridge = true;
    var closeUrl = ${JSON.stringify(CloseSentinel)};
    var nativeGo = history.go.bind(history);
    function closeToApp () {
      location.replace(closeUrl);
    }
    function goBack () {
      try {
        if (window.plus && plus.webview) {
          var current = plus.webview.currentWebview();
          if (current && typeof current.canBack === 'function') {
            current.canBack(function (evt) {
              if (evt && evt.canBack) current.back();
              else closeToApp();
            });
            return;
          }
        }
      } catch (error) {}
      if (history.length > 1) nativeGo(-1);
      else closeToApp();
    }
    history.back = goBack;
    history.go = function (delta) {
      if (Number(delta) < 0) goBack();
      else nativeGo(delta);
    };
    window.close = closeToApp;
  })()`)
}

function bindChildWebview (wv) {
  if (!wv || childEventsBound) return
  childEventsBound = true

  try {
    wv.overrideUrlLoading({
      mode: 'reject',
      match: '.*mbhl\\.local/close-webview.*'
    }, () => {
      returnToAgreement()
    })
  } catch (error) {}

  wv.addEventListener('loaded', () => {
    injectBackBridge(wv)
  })
  injectBackBridge(wv)
}

/**
 * 第三方网页让出系统状态栏：时间、电量还在，网页从状态栏下面开始画。
 * iOS / 安卓沉浸式时 web-view 默认会铺到屏幕顶把状态栏盖住，这里把子 webview 往下挪。
 */
function reserveStatusBar (attempt = 0) {
  // #ifdef APP-PLUS
  if (!url.value) return

  const wv = getChildWebview()
  if (!wv) {
    if (attempt < 20) pinTimer = setTimeout(() => reserveStatusBar(attempt + 1), 100)
    return
  }

  bindChildWebview(wv)

  plus.navigator.setFullscreen(false)
  plus.navigator.setStatusBarStyle('dark')
  plus.navigator.setStatusBarBackground('#fff3f6')

  const info = uni.getSystemInfoSync() || {}
  const statusBarHeight = Number(info.statusBarHeight || plus.navigator.getStatusbarHeight() || 0)
  const screenHeight = Number((plus.screen && plus.screen.resolutionHeight) || info.screenHeight || 0)
  wv.setStyle({
    top: statusBarHeight,
    height: Math.max(screenHeight - statusBarHeight, 0)
  })
  // #endif
}

onLoad((query = {}) => {
  let target = ''
  try {
    target = decodeURIComponent(query.url || '')
  } catch (error) {
    target = String(query.url || '')
  }

  if (!isSafeWebUrl(target)) {
    toast('链接地址无效')
    returnToAgreement(300)
    return
  }

  if (isExternalUrl(target)) {
    openExternalUrl(target)
    return
  }

  url.value = target
})

onReady(() => {
  pinTimer = setTimeout(() => reserveStatusBar(), 200)
})

onBackPress(({ from }) => {
  if (from !== 'backbutton') return false
  handleWebViewBack()
  return true
})

onUnload(() => {
  clearTimeout(returnTimer)
  clearTimeout(pinTimer)
  childWebview = null
  childEventsBound = false
})
</script>

<style lang="scss" scoped>
.web-page {
  min-height: 100vh;
  background: var(--color-bg);
}
</style>
