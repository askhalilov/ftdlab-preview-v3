<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import { usesNativeScroll } from '../mobileScroll'
import { asset } from '../assets'
import { copy } from '../i18n'
import '../section05.css'

gsap.registerPlugin(ScrollTrigger)
const emit = defineEmits<{ (e: 'contact', partnership?: string): void }>()
const root = ref<HTMLElement>()
const active = ref(false)
const visible = ref(!document.hidden)
const reduced = ref(matchMedia('(prefers-reduced-motion: reduce)').matches)
type EyeState = 'open' | 'half' | 'closed'
type Mascot = 'female' | 'male'
const femaleEye = ref<EyeState>('open')
const maleEye = ref<EyeState>('open')
const mascotAsset = (sex: Mascot, name: string) => asset(`section05/approved/${sex}/${name}.png`)
const eyeAsset = (sex: Mascot, state: EyeState) => mascotAsset(sex, `eyes_${state}`)
const blinkTimers = new Set<number>()
let blinking = false
let eyesReady = false
function after(ms: number, action: () => void) {
  const timer = window.setTimeout(() => { blinkTimers.delete(timer); action() }, ms)
  blinkTimers.add(timer)
}
function canBlink() { return active.value && visible.value && !reduced.value && eyesReady }
function stopBlink() {
  blinking = false
  blinkTimers.forEach(clearTimeout)
  blinkTimers.clear()
  femaleEye.value = 'open'
  maleEye.value = 'open'
}
function blink(sex: Mascot) {
  if (!canBlink()) return
  const eye = sex === 'female' ? femaleEye : maleEye
  eye.value = 'half'
  after(85, () => { if (canBlink()) eye.value = 'closed' })
  after(160, () => { if (canBlink()) eye.value = 'half' })
  after(245, () => {
    if (!canBlink()) return
    eye.value = 'open'
    const pause = sex === 'female' ? 3500 + Math.random() * 1500 : 4500 + Math.random() * 2500
    after(pause, () => blink(sex))
  })
}
function syncBlink() {
  if (!canBlink()) { stopBlink(); return }
  if (blinking) return
  blinking = true
  after(1200, () => blink('female'))
  after(2400, () => blink('male'))
}
async function preloadEyes() {
  await Promise.all((['female', 'male'] as const).flatMap(sex =>
    (['open', 'half', 'closed'] as const).map(state => new Promise<void>(resolve => {
      const image = new Image()
      image.onload = () => { void image.decode().catch(() => {}).finally(resolve) }
      image.onerror = () => resolve()
      image.src = eyeAsset(sex, state)
    }))))
  eyesReady = true
  syncBlink()
}
let motionQuery: MediaQueryList | undefined
let trigger: ScrollTrigger | undefined
let timeline: gsap.core.Timeline | undefined
let played = false

const cardIcons = ['advertisers', 'networks', 'media'] as const
function prepare() {
  if (!root.value || reduced.value || usesNativeScroll()) return
  const el = root.value
  gsap.set(el.querySelector('.s05__meta'), { opacity: 0, y: 13 })
  gsap.set(el.querySelector('.s05__title'), { opacity: 0, y: 24 })
  gsap.set(el.querySelector('.s05__lead'), { opacity: 0, y: 16 })
  gsap.set(el.querySelector('.s05__scene'), { opacity: 0, y: 23 })
  gsap.set(el.querySelector('.s05__pills'), { opacity: 0, y: 12 })
  gsap.set(el.querySelectorAll('.s05__card'), { opacity: 0, y: 19 })
}
function enter() {
  active.value = true
  syncBlink()
  if (!root.value || reduced.value) return
  if (played) { if (timeline && timeline.progress() < 1 && visible.value) timeline.resume(); return }
  played = true
  const el = root.value
  if (usesNativeScroll()) {
    timeline = gsap.timeline().fromTo(el.querySelectorAll('.s05__meta,.s05__title,.s05__lead,.s05__scene,.s05__pills,.s05__card'),
      { opacity: .65, y: 10 }, { opacity: 1, y: 0, duration: .42, ease: 'power2.out' })
    return
  }
  timeline = gsap.timeline()
    .to(el.querySelector('.s05__meta'), { opacity: 1, y: 0, duration: .37 }, 0)
    .to(el.querySelector('.s05__title'), { opacity: 1, y: 0, duration: .55, ease: 'power2.out' }, .15)
    .to(el.querySelector('.s05__lead'), { opacity: 1, y: 0, duration: .45 }, .37)
    .to(el.querySelector('.s05__scene'), { opacity: 1, y: 0, duration: .7, ease: 'power2.out' }, .25)
    .to(el.querySelector('.s05__pills'), { opacity: 1, y: 0, duration: .4 }, .58)
    .to(el.querySelectorAll('.s05__card'), { opacity: 1, y: 0, duration: .45, stagger: .13 }, .68)
}
function leave() { active.value = false; stopBlink(); if (timeline && timeline.progress() < 1) timeline.pause() }
function visibility() {
  visible.value = !document.hidden
  syncBlink()
  if (!visible.value) timeline?.pause()
  else if (active.value && timeline && timeline.progress() < 1) timeline.resume()
}
function motionChange() {
  reduced.value = !!motionQuery?.matches
  syncBlink()
  if (reduced.value) {
    timeline?.kill(); timeline = undefined; played = true
    if (root.value) gsap.set(root.value.querySelectorAll('.s05__meta,.s05__title,.s05__lead,.s05__scene,.s05__pills,.s05__card'), { opacity: 1, y: 0 })
  }
}
onMounted(() => {
  void preloadEyes()
  motionQuery = matchMedia('(prefers-reduced-motion: reduce)')
  motionQuery.addEventListener('change', motionChange)
  document.addEventListener('visibilitychange', visibility)
  prepare()
  trigger = ScrollTrigger.create({ trigger: root.value!, start: 'top 78%', end: 'bottom top', invalidateOnRefresh: true, onEnter: enter, onEnterBack: enter, onLeave: leave, onLeaveBack: leave })
  ScrollTrigger.refresh()
})
onBeforeUnmount(() => {
  stopBlink()
  trigger?.kill(); timeline?.kill()
  motionQuery?.removeEventListener('change', motionChange)
  document.removeEventListener('visibilitychange', visibility)
})
</script>

<template>
  <section id="section-05" ref="root" class="s05" :data-active="active && visible && !reduced" aria-labelledby="s05-title">
    <div class="s05-boundary" aria-hidden="true"></div>
    <div class="s05__bridge" aria-hidden="true"></div>
    <div class="s05__hero">
      <div class="s05__copy">
        <div class="s05__meta"><span>05.</span><span class="s05__meta-line"></span><span class="s05__meta-name">{{ copy.s05.eyebrow }}</span></div>
        <h2 id="s05-title" class="s05__title">PARTNERSHIPS</h2>
        <p class="s05__lead">{{ copy.s05.description }}</p>
        <div class="s05__pills" :aria-label="copy.s05.principles">
          <div v-for="item in ['GEO', 'BRANDS', 'TRAFFIC', 'GROWTH']" :key="item" class="s05__pill">
            <span class="s05__pill-icon" aria-hidden="true">
              <svg v-if="item === 'GEO'" viewBox="0 0 40 40" fill="none" stroke="currentColor" stroke-width="2"><circle cx="20" cy="20" r="14"/><path d="M6 20h28M20 6c-5 5-7 10-7 14s2 9 7 14M20 6c5 5 7 10 7 14s-2 9-7 14M10 12h20M10 28h20"/></svg>
              <svg v-else-if="item === 'BRANDS'" viewBox="0 0 40 40" fill="none" stroke="currentColor" stroke-width="2"><path d="M5 12 11 28h18l6-16-10 8-5-11-5 11zM11 28v4h18v-4"/><circle cx="5" cy="10" r="1"/><circle cx="20" cy="7" r="1"/><circle cx="35" cy="10" r="1"/></svg>
              <svg v-else-if="item === 'TRAFFIC'" viewBox="0 0 40 40" fill="none" stroke="currentColor" stroke-width="2"><path d="M6 33V23h5v10M15 33V16h5v17M24 33V9h5v24M33 33V4h3v29M4 34h34"/></svg>
              <svg v-else viewBox="0 0 40 40" fill="none" stroke="currentColor" stroke-width="2"><path d="M8 31 31 8M19 8h12v12"/></svg>
            </span><span class="s05__pill-name">{{ item }}</span>
          </div>
        </div>
      </div>
      <div class="s05__scene" aria-hidden="true">
        <div class="s05__seat"></div>
        <div class="s05__mascot-pair">
          <div class="s05__mascot s05__mascot--female" :data-eye-state="femaleEye">
            <img class="s05__mascot-base" :src="mascotAsset('female', 'mascot_base_no_eyes')" alt="" draggable="false" loading="eager" decoding="sync" />
            <img class="s05__mascot-eyes" :src="eyeAsset('female', femaleEye)" alt="" draggable="false" loading="eager" decoding="async" />
          </div>
          <div class="s05__mascot s05__mascot--male" :data-eye-state="maleEye">
            <img class="s05__mascot-base" :src="mascotAsset('male', 'mascot_base_no_eyes')" alt="" draggable="false" loading="eager" decoding="sync" />
            <img class="s05__mascot-eyes" :src="eyeAsset('male', maleEye)" alt="" draggable="false" loading="eager" decoding="async" />
          </div>
        </div>
      </div>
    </div>
    <div class="s05__cards" :aria-label="copy.s05.directions">
      <article v-for="(card, index) in copy.s05.cards" :key="index" class="s05__card">
        <div class="s05__card-top">
          <svg class="s05__card-symbol" viewBox="0 0 40 40" aria-hidden="true" fill="none" stroke="currentColor" stroke-width="2">
            <template v-if="cardIcons[index] === 'advertisers'"><circle cx="15" cy="13" r="5"/><circle cx="28" cy="16" r="4"/><path d="M4 35v-4c0-6 5-9 11-9s11 3 11 9v4M25 25c6-1 12 3 12 8v2"/></template>
            <template v-else-if="cardIcons[index] === 'networks'"><circle cx="7" cy="20" r="4"/><circle cx="31" cy="9" r="4"/><circle cx="31" cy="31" r="4"/><path d="m11 18 16-8M11 22l16 8"/></template>
            <template v-else><path d="M7 34V23h6v11M17 34V16h6v18M27 34V7h6v27"/></template>
          </svg>
          <div class="s05__card-body"><div class="s05__card-label">{{ copy.s05.for }}</div><h3 class="s05__card-heading" :tabindex="index === 0 ? -1 : undefined">{{ card.title }}</h3><p class="s05__card-desc">{{ card.description }}</p></div>
          <button class="s05__card-arrow" type="button" :aria-label="`${copy.s05.discuss}: ${card.title}`" @click="emit('contact',card.title)">
            <svg width="17" height="17" viewBox="0 0 20 20" fill="none" stroke="currentColor" stroke-width="1.6" aria-hidden="true"><path d="M3 10h13m-5-5 5 5-5 5"/></svg>
          </button>
        </div>
        <ul class="s05__tags"><li v-for="tag in card.tags" :key="tag" class="s05__tag">{{ tag }}</li></ul>
      </article>
    </div>
  </section>
</template>
