<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

const albumModules = import.meta.glob('./assets/albums/*.{jpg,jpeg,png,webp}', {
  eager: true,
  import: 'default',
  query: '?url',
})

const albumPhotos = Object.entries(albumModules)
  .sort(([a], [b]) => a.localeCompare(b, 'zh-Hant', { numeric: true }))
  .map(([path, src]) => ({
    src,
    alt: `Paul 和 Peggy 的婚紗照 ${path.split('/').pop()?.replace(/\.[^.]+$/, '')}`,
  }))

const wedding = {
  groom: 'Paul',
  bride: 'Peggy',
  groomZh: '智帆',
  brideZh: '瑩庭',
  title: '誠摯邀請你一同見證我們的重要時刻',
  date: '2026-10-25T12:00:00+08:00',
  venue: '彭園新板館',
  address: '點擊下方地圖查看場地位置',
  ceremony: '12:00 午宴入席',
  banquet: '12:30 午宴開始',
  note: '期待在這一天，與你一起見證我們人生最重要的時刻。',
  mapUrl: 'https://maps.app.goo.gl/iYmfKt6GPxBbnfE17',
  rsvpUrl: 'https://forms.gle/D83uadHrTu8xJCXMA',
  schedule: [
    { time: '11:30', title: '賓客報到', detail: '歡迎提前抵達，輕鬆入場與拍照留念' },
    { time: '12:00', title: '午宴入席', detail: '邀請您一同入席，準備迎接婚禮開始' },
    { time: '12:30', title: '婚宴開始', detail: '與我們一起分享幸福與喜悅' },
    { time: '15:00', title: '圓滿禮成', detail: '感謝您的蒞臨與祝福' },
  ],
}

const targetTime = new Date(wedding.date).getTime()
const now = ref(Date.now())
const activePhotoIndex = ref(0)
const touchStartX = ref(0)
const touchStartY = ref(0)
let timerId

const activePhoto = computed(() => activePhotoIndex.value + 1)

function showPhoto(index) {
  if (!albumPhotos.length) return
  activePhotoIndex.value = (index + albumPhotos.length) % albumPhotos.length
}

function showPreviousPhoto() {
  showPhoto(activePhotoIndex.value - 1)
}

function showNextPhoto() {
  showPhoto(activePhotoIndex.value + 1)
}

function handleTouchStart(event) {
  const touch = event.touches[0]
  touchStartX.value = touch.clientX
  touchStartY.value = touch.clientY
}

function handleTouchEnd(event) {
  const touch = event.changedTouches[0]
  const deltaX = touch.clientX - touchStartX.value
  const deltaY = touch.clientY - touchStartY.value

  if (Math.abs(deltaX) < 45 || Math.abs(deltaY) > Math.abs(deltaX)) return

  if (deltaX < 0) {
    showNextPhoto()
  } else {
    showPreviousPhoto()
  }
}

const remaining = computed(() => {
  const diff = Math.max(targetTime - now.value, 0)
  const days = Math.floor(diff / (1000 * 60 * 60 * 24))
  const hours = Math.floor((diff / (1000 * 60 * 60)) % 24)
  const minutes = Math.floor((diff / (1000 * 60)) % 60)
  const seconds = Math.floor((diff / 1000) % 60)

  return [
    { label: '天', value: String(days).padStart(2, '0') },
    { label: '時', value: String(hours).padStart(2, '0') },
    { label: '分', value: String(minutes).padStart(2, '0') },
    { label: '秒', value: String(seconds).padStart(2, '0') },
  ]
})

const weddingDate = computed(() => {
  return new Intl.DateTimeFormat('zh-TW', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    weekday: 'long',
    hour: '2-digit',
    minute: '2-digit',
  }).format(new Date(wedding.date))
})

onMounted(() => {
  timerId = window.setInterval(() => {
    now.value = Date.now()
  }, 1000)
})

onBeforeUnmount(() => {
  window.clearInterval(timerId)
})
</script>

<template>
  <main class="page-shell">
    <section class="invite-card">
      <div class="hero">
        <p class="eyebrow">Wedding Invitation</p>
        <p class="chinese-title">{{ wedding.groomZh }} &amp; {{ wedding.brideZh }} 的婚禮</p>
        <h1>{{ wedding.groom }} &amp; {{ wedding.bride }}</h1>
        <p class="subtitle">{{ wedding.title }}</p>
      </div>

      <div class="album-frame" aria-label="婚紗照相簿">
        <div
          class="album-viewport"
          @touchstart.passive="handleTouchStart"
          @touchend.passive="handleTouchEnd"
        >
          <div
            class="album-track"
            :style="{ transform: `translateX(-${activePhotoIndex * 100}%)` }"
          >
            <img
              v-for="photo in albumPhotos"
              :key="photo.src"
              :src="photo.src"
              :alt="photo.alt"
            />
          </div>
        </div>

        <button class="album-control album-control-prev" type="button" @click="showPreviousPhoto">
          ‹
          <span class="sr-only">上一張照片</span>
        </button>
        <button class="album-control album-control-next" type="button" @click="showNextPhoto">
          ›
          <span class="sr-only">下一張照片</span>
        </button>

        <div class="album-footer">
          <div class="album-dots" aria-label="選擇照片">
            <button
              v-for="(photo, index) in albumPhotos"
              :key="`${photo.src}-dot`"
              type="button"
              :class="{ active: index === activePhotoIndex }"
              :aria-label="`切換到第 ${index + 1} 張照片`"
              :aria-current="index === activePhotoIndex"
              @click="showPhoto(index)"
            ></button>
          </div>
          <span>{{ activePhoto }} / {{ albumPhotos.length }}</span>
        </div>
      </div>

      <section class="panel countdown-panel">
        <p class="panel-label">婚禮倒數</p>
        <div class="countdown-grid">
          <article v-for="item in remaining" :key="item.label" class="count-box">
            <strong>{{ item.value }}</strong>
            <span>{{ item.label }}</span>
          </article>
        </div>
      </section>

      <section class="panel rsvp-panel">
        <p class="panel-label">出席回覆</p>
        <p>期待與你一起分享這一天的喜悅，請協助填寫婚禮出席表單。</p>
        <a class="primary-button" :href="wedding.rsvpUrl" target="_blank" rel="noreferrer">
          填寫出席表單
        </a>
      </section>

      <section class="panel info-panel">
        <p class="panel-label">婚禮資訊</p>
        <div class="info-row">
          <span>日期時間</span>
          <strong>{{ weddingDate }}</strong>
        </div>
        <div class="info-row">
          <span>宴會地點</span>
          <strong>{{ wedding.venue }}</strong>
        </div>
        <div class="info-row muted">
          <span>地址</span>
          <strong>{{ wedding.address }}</strong>
        </div>
        <a class="primary-button" :href="wedding.mapUrl" target="_blank" rel="noreferrer">
          查看地圖
        </a>
      </section>

      <section class="panel detail-panel">
        <p class="panel-label">當日安排</p>
        <div class="pill-group">
          <span class="pill">{{ wedding.ceremony }}</span>
          <span class="pill">{{ wedding.banquet }}</span>
        </div>
        <div class="timeline">
          <article
            v-for="item in wedding.schedule"
            :key="`${item.time}-${item.title}`"
            class="timeline-item"
          >
            <span class="timeline-time">{{ item.time }}</span>
            <span class="timeline-marker" aria-hidden="true"></span>
            <div class="timeline-content">
              <h2>{{ item.title }}</h2>
              <p>{{ item.detail }}</p>
            </div>
          </article>
        </div>
      </section>

      <section class="closing">
        <p>{{ wedding.note }}</p>
      </section>
    </section>
  </main>
</template>
