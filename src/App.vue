<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'
import heroPhoto from './assets/cover-photo.jpg'

const wedding = {
  groom: 'Paul',
  bride: 'Peggy',
  title: '誠摯邀請你一同見證我們的重要時刻',
  date: '2026-10-25T12:00:00+08:00',
  venue: '彭園新板館',
  address: '點擊下方地圖查看場地位置',
  ceremony: '12:00 午宴入席',
  banquet: '12:30 午宴開始',
  note: '期待在這一天，與你一起見證我們人生最重要的時刻。',
  mapUrl: 'https://maps.app.goo.gl/iYmfKt6GPxBbnfE17',
  schedule: [
    { time: '11:30', title: '賓客報到', detail: '歡迎提前抵達，輕鬆入場與拍照留念' },
    { time: '12:00', title: '午宴入席', detail: '邀請您一同入席，準備迎接婚禮開始' },
    { time: '12:30', title: '婚宴開始', detail: '與我們一起分享幸福與喜悅' },
    { time: '15:00', title: '圓滿禮成', detail: '感謝您的蒞臨與祝福' },
  ],
}

const targetTime = new Date(wedding.date).getTime()
const now = ref(Date.now())
let timerId

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
        <h1>{{ wedding.groom }} &amp; {{ wedding.bride }}</h1>
        <p class="subtitle">{{ wedding.title }}</p>
      </div>

      <div class="photo-frame">
        <img :src="heroPhoto" alt="Paul 和 Peggy 的婚紗照" />
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
        <a class="map-button" :href="wedding.mapUrl" target="_blank" rel="noreferrer">
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
