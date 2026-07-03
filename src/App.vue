<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from 'vue'

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
  date: '2026-10-25T12:00:00+08:00',
  venue: '彭園新板館(八樓花園廳)',
  address: '新北市板橋區中山路一段161號8樓\n(新北市政府大樓北二門)\n搭乘 21,22,23 號專屬電梯',
  ceremony: '11:30 午宴入席',
  banquet: '12:00 午宴開始',
  note: ['期待在這一天', '與你一起見證我們人生最重要的時刻'],
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
const albumTrackIndex = ref(albumPhotos.length > 1 ? 1 : 0)
const isAlbumTransitionEnabled = ref(true)
const isAlbumAnimating = ref(false)
const albumTrackElement = ref(null)
const isLightboxOpen = ref(false)
const touchStartX = ref(0)
const touchStartY = ref(0)
let timerId
let previousBodyOverflow = ''

const activePhoto = computed(() => activePhotoIndex.value + 1)
const activePhotoItem = computed(() => albumPhotos[activePhotoIndex.value])
const renderedAlbumPhotos = computed(() => {
  if (albumPhotos.length <= 1) return albumPhotos.map((photo, index) => ({ ...photo, index }))

  const firstPhoto = albumPhotos[0]
  const lastPhoto = albumPhotos[albumPhotos.length - 1]

  return [
    { ...lastPhoto, index: albumPhotos.length - 1, clone: 'last' },
    ...albumPhotos.map((photo, index) => ({ ...photo, index })),
    { ...firstPhoto, index: 0, clone: 'first' },
  ]
})

async function resetAlbumTrack(index) {
  isAlbumTransitionEnabled.value = false
  albumTrackIndex.value = index
  isAlbumAnimating.value = false
  await nextTick()
  albumTrackElement.value?.getBoundingClientRect()
  requestAnimationFrame(() => {
    requestAnimationFrame(() => {
      isAlbumTransitionEnabled.value = true
    })
  })
}

function showPhoto(index) {
  if (!albumPhotos.length) return
  if (isAlbumAnimating.value && !isLightboxOpen.value) return

  isAlbumTransitionEnabled.value = true
  activePhotoIndex.value = (index + albumPhotos.length) % albumPhotos.length
  albumTrackIndex.value = albumPhotos.length > 1 ? activePhotoIndex.value + 1 : activePhotoIndex.value
  isAlbumAnimating.value = false
}

function showPreviousPhoto() {
  if (albumPhotos.length <= 1) return

  if (isLightboxOpen.value) {
    showPhoto(activePhotoIndex.value - 1)
    return
  }

  if (isAlbumAnimating.value) return

  isAlbumTransitionEnabled.value = true
  isAlbumAnimating.value = true
  activePhotoIndex.value = (activePhotoIndex.value - 1 + albumPhotos.length) % albumPhotos.length
  albumTrackIndex.value -= 1
}

function showNextPhoto() {
  if (albumPhotos.length <= 1) return

  if (isLightboxOpen.value) {
    showPhoto(activePhotoIndex.value + 1)
    return
  }

  if (isAlbumAnimating.value) return

  isAlbumTransitionEnabled.value = true
  isAlbumAnimating.value = true
  activePhotoIndex.value = (activePhotoIndex.value + 1) % albumPhotos.length
  albumTrackIndex.value += 1
}

function handleAlbumTransitionEnd(event) {
  if (event.propertyName !== 'transform' || albumPhotos.length <= 1) return

  if (albumTrackIndex.value === albumPhotos.length + 1) {
    resetAlbumTrack(1)
    return
  }

  if (albumTrackIndex.value === 0) {
    resetAlbumTrack(albumPhotos.length)
    return
  }

  isAlbumAnimating.value = false
}

function openLightbox(index) {
  showPhoto(index)
  previousBodyOverflow = document.body.style.overflow
  document.body.style.overflow = 'hidden'
  isLightboxOpen.value = true
}

function closeLightbox() {
  document.body.style.overflow = previousBodyOverflow
  isLightboxOpen.value = false
}

function handleKeydown(event) {
  if (!isLightboxOpen.value) return

  if (event.key === 'Escape') closeLightbox()
  if (event.key === 'ArrowLeft') showPreviousPhoto()
  if (event.key === 'ArrowRight') showNextPhoto()
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
  const date = new Intl.DateTimeFormat('zh-TW', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    weekday: 'long',
  }).format(new Date(wedding.date))

  return `${date} 中午12:00`
})

onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
  timerId = window.setInterval(() => {
    now.value = Date.now()
  }, 1000)
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleKeydown)
  document.body.style.overflow = previousBodyOverflow
  window.clearInterval(timerId)
})
</script>

<template>
  <main class="page-shell">
    <section class="invite-card">
      <div class="hero">
        <p class="eyebrow">Wedding Invitation</p>
        <h1>{{ wedding.groom }} &amp; {{ wedding.bride }}</h1>
        <img
          class="chinese-title-art"
          src="./assets/chinese-title.png"
          alt="智帆與瑩庭手寫簽名"
        />
      </div>

      <div class="album-frame" aria-label="婚紗照相簿">
        <div
          class="album-viewport"
          @touchstart.passive="handleTouchStart"
          @touchend.passive="handleTouchEnd"
        >
          <div
            ref="albumTrackElement"
            class="album-track"
            :class="{ 'album-track-instant': !isAlbumTransitionEnabled }"
            :style="{ transform: `translateX(-${albumTrackIndex * 100}%)` }"
            @transitionend="handleAlbumTransitionEnd"
          >
            <img
              v-for="(photo, index) in renderedAlbumPhotos"
              :key="`${photo.src}-${photo.clone ?? 'photo'}-${index}`"
              :src="photo.src"
              :alt="photo.alt"
              role="button"
              :tabindex="photo.index === activePhotoIndex && !photo.clone ? 0 : -1"
              aria-label="開啟照片大圖"
              @click="openLightbox(photo.index)"
              @keydown.enter="openLightbox(photo.index)"
              @keydown.space.prevent="openLightbox(photo.index)"
            />
          </div>
        </div>

        <button class="album-control album-control-prev" type="button" @click="showPreviousPhoto">
          <span aria-hidden="true">‹</span>
          <span class="sr-only">上一張照片</span>
        </button>
        <button class="album-control album-control-next" type="button" @click="showNextPhoto">
          <span aria-hidden="true">›</span>
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
        <p>
          期待與你一起分享這一天的喜悅<br />
          請協助填寫婚禮出席表單
        </p>
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
        <p>
          <template v-for="(line, index) in wedding.note" :key="line">
            {{ line }}<br v-if="index < wedding.note.length - 1" />
          </template>
        </p>
      </section>
    </section>
  </main>

  <Teleport to="body">
    <div
      v-if="isLightboxOpen"
      class="lightbox"
      role="dialog"
      aria-modal="true"
      aria-label="婚紗照大圖"
      @click.self="closeLightbox"
    >
      <button class="lightbox-close" type="button" aria-label="關閉大圖" @click="closeLightbox">
        ×
      </button>
      <button
        class="lightbox-control lightbox-control-prev"
        type="button"
        aria-label="上一張照片"
        @click="showPreviousPhoto"
      >
        <span aria-hidden="true">‹</span>
      </button>
      <img
        v-if="activePhotoItem"
        class="lightbox-image"
        :src="activePhotoItem.src"
        :alt="activePhotoItem.alt"
      />
      <button
        class="lightbox-control lightbox-control-next"
        type="button"
        aria-label="下一張照片"
        @click="showNextPhoto"
      >
        <span aria-hidden="true">›</span>
      </button>
      <p class="lightbox-count">{{ activePhoto }} / {{ albumPhotos.length }}</p>
    </div>
  </Teleport>
</template>
