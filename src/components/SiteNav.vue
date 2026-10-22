<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'
import gsap from 'gsap'
import { copy, locale, setLocale, type Locale } from '../i18n'
import connectLogo from '../../assets/section04/connect-logo-reference.png'
import { MOBILE_SCROLL_QUERY, usesNativeScroll } from '../mobileScroll'
import '../site-nav.css'

const links = [
  { key: 'home', id: 'home' },
  { key: 'performance', id: 'section-02' },
  { key: 'what', id: 'section-03' },
  { key: 'connect', id: 'section-04' },
  { key: 'partnerships', id: 'section-05' },
  { key: 'faq', id: 'section-06' },
  { key: 'contact', id: 'section-07' },
] as const
const open = ref(false)
const current = ref<(typeof links)[number]['id']>('home')
const currentIndex = computed(() => Math.max(0, links.findIndex(link => link.id === current.value)))
const trigger = ref<HTMLButtonElement>()
const stepTrigger = ref<HTMLButtonElement>()
let frame = 0
let scrollTween: gsap.core.Tween | undefined
let scrolling = false
let wheelTotal = 0
let lastWheel = 0
let wheelConsumed = false
let mobileNavigating = false
let scrollQuery: MediaQueryList | undefined

function activePerformance() { return !usesNativeScroll() && document.getElementById('section-02')?.dataset.sceneActive === 'true' }
function updateCurrent() {
  frame = 0
  if (activePerformance()) { current.value = 'section-02'; return }
  const threshold = Math.min(innerHeight * .34, 280)
  let selected: (typeof links)[number]['id'] = 'home'
  for (const link of links) {
    const section = document.getElementById(link.id)
    if (section && section.getBoundingClientRect().top <= threshold) selected = link.id
  }
  if (current.value !== selected) open.value = false
  current.value = selected
}
function queueUpdate() { if (!frame) frame = requestAnimationFrame(updateCurrent) }
function moveTo(targetY: number, duration = .62, complete?: () => void) {
  scrollTween?.kill()
  const destination = Math.max(0, targetY)
  if (matchMedia('(prefers-reduced-motion: reduce)').matches) { scrollTo(0, destination); scrolling = false; queueUpdate(); complete?.(); return }
  const position = { y: scrollY }
  scrolling = true
  scrollTween = gsap.to(position, {
    y: destination, duration, ease: 'power2.inOut', overwrite: true,
    onUpdate: () => scrollTo(0, position.y),
    onComplete: () => { scrolling = false; scrollTween = undefined; queueUpdate(); complete?.() },
  })
}
function navigate(id: string, record = true, reverseIntoPerformance = false) {
  open.value = false
  const target = document.getElementById(id)
  if (!target) return
  if (record) history.pushState(null, '', `#${id}`)
  scrollTween?.kill(); scrolling = false
  if (usesNativeScroll()) {
    mobileNavigating = !matchMedia('(prefers-reduced-motion: reduce)').matches
    window.scrollTo({ top: target.getBoundingClientRect().top + scrollY, behavior: mobileNavigating ? 'smooth' : 'instant' })
  }
  else if (id === 'section-02' && reverseIntoPerformance) window.dispatchEvent(new Event('ftdlab:performance-end'))
  else if (id === 'section-02') moveTo(target.getBoundingClientRect().top + scrollY + 2, 1.08, () => window.dispatchEvent(new Event('ftdlab:performance-start')))
  else moveTo(target.getBoundingClientRect().top + scrollY)
  current.value = id as typeof current.value
}
function select(event: MouseEvent, id: string) {
  if (event.button || event.ctrlKey || event.metaKey || event.shiftKey || event.altKey) return
  event.preventDefault()
  navigate(id)
}
function onNavigate(event: Event) { navigate((event as CustomEvent<string>).detail) }
function onHistory() {
  const id = decodeURIComponent(location.hash.slice(1)) || 'home'
  navigate(id, false)
}
function onKey(event: KeyboardEvent) {
  if (event.key === 'Escape' && open.value) { open.value = false; (current.value === 'section-04' && innerWidth > 900 ? stepTrigger.value : trigger.value)?.focus(); return }
  if (usesNativeScroll() || matchMedia('(prefers-reduced-motion: reduce)').matches || activePerformance() || event.repeat || event.altKey || event.ctrlKey || event.metaKey) return
  const down = event.key === 'PageDown' || event.key === 'ArrowDown' || (event.key === ' ' && !event.shiftKey)
  const up = event.key === 'PageUp' || event.key === 'ArrowUp' || (event.key === ' ' && event.shiftKey)
  if (!down && !up) return
  if ((event.target as HTMLElement).closest('input, textarea, select, button, a, [contenteditable="true"]')) return
  if (trySnap(down ? 1 : -1)) event.preventDefault()
}
function trySnap(direction: number) {
  if (usesNativeScroll() || scrolling || activePerformance()) return false
  const index = currentIndex.value
  const link = links[index]
  if (!link || link.id === 'section-02') return false
  const section = document.getElementById(link.id)
  if (!section) return false
  const top = section.getBoundingClientRect().top + scrollY
  const bottom = top + section.getBoundingClientRect().height
  const edge = Math.min(24, innerHeight * .03)
  if (direction > 0 && scrollY + innerHeight < bottom - edge) return false
  if (direction < 0 && scrollY > top + edge) return false
  if (link.id === 'home' && direction > 0 && scrollY < 84) {
    moveTo(84, .38)
    return true
  }
  const next = links[index + direction]
  if (!next) return false // Footer remains naturally scrollable.
  navigate(next.id, false, direction < 0 && next.id === 'section-02')
  return true
}
function onWheel(event: WheelEvent) {
  if (usesNativeScroll() || matchMedia('(prefers-reduced-motion: reduce)').matches || activePerformance() || event.ctrlKey) return
  const delta = event.deltaY * (event.deltaMode === 1 ? 16 : event.deltaMode === 2 ? innerHeight : 1)
  const now = performance.now()
  if (now - lastWheel > 280) { wheelTotal = 0; wheelConsumed = false }
  lastWheel = now
  if (scrolling || wheelConsumed) { event.preventDefault(); return }
  wheelTotal += delta
  if (Math.abs(wheelTotal) < 48) return
  if (trySnap(Math.sign(wheelTotal))) { event.preventDefault(); wheelConsumed = true }
  else wheelTotal = 0
}
function onScrollEnd() {
  mobileNavigating = false
}
function interruptNavigation() {
  if (!usesNativeScroll() || !mobileNavigating) return
  mobileNavigating = false
  window.scrollTo({ top: scrollY, behavior: 'instant' })
}
function scrollModeChange() {
  scrollTween?.kill(); scrollTween = undefined; scrolling = false
  mobileNavigating = false; wheelTotal = 0; wheelConsumed = false
  removeEventListener('wheel', onWheel)
  if (!usesNativeScroll()) addEventListener('wheel', onWheel, { passive: false })
}
function changeLanguage(next: Locale) { setLocale(next); open.value = false; queueUpdate() }
onMounted(() => {
  addEventListener('scroll', queueUpdate, { passive: true })
  addEventListener('scrollend', onScrollEnd)
  addEventListener('resize', queueUpdate)
  scrollQuery = matchMedia(MOBILE_SCROLL_QUERY)
  scrollQuery.addEventListener('change', scrollModeChange)
  scrollModeChange()
  addEventListener('pointerdown', interruptNavigation, { passive: true })
  addEventListener('ftdlab:navigate', onNavigate)
  addEventListener('popstate', onHistory)
  addEventListener('keydown', onKey)
  queueUpdate()
  if (location.hash) requestAnimationFrame(onHistory)
})
onBeforeUnmount(() => {
  removeEventListener('scroll', queueUpdate)
  removeEventListener('scrollend', onScrollEnd)
  removeEventListener('resize', queueUpdate)
  removeEventListener('wheel', onWheel)
  scrollQuery?.removeEventListener('change', scrollModeChange)
  removeEventListener('pointerdown', interruptNavigation)
  removeEventListener('ftdlab:navigate', onNavigate)
  removeEventListener('popstate', onHistory)
  removeEventListener('keydown', onKey)
  scrollTween?.kill()
  if (frame) cancelAnimationFrame(frame)
})
</script>

<template>
  <header class="site-nav" :data-current-section="current">
    <a class="site-nav__logo" href="#home" :aria-label="`FTD.LAB — ${copy.nav.home}`" @click="select($event, 'home')"><span class="site-nav__logo-standard">FTD.<span>LAB</span></span><img class="site-nav__connect-logo" :src="connectLogo" alt="" aria-hidden="true" /></a>
    <button ref="stepTrigger" type="button" class="site-nav__mobile-step" :aria-expanded="open" aria-controls="site-nav-links" :aria-label="`${copy.nav.toggle}; ${copy.nav.current}: ${copy.nav[links[currentIndex]!.key]}`" @click="open = !open">{{ String(currentIndex + 1).padStart(2, '0') }} / 07</button>
    <nav id="site-nav-links" class="site-nav__links" :class="{ 'site-nav__links--open': open }" :aria-label="copy.nav.main">
      <a v-for="link in links" :key="link.id" :href="`#${link.id}`" :aria-current="current === link.id ? 'location' : undefined" @click="select($event, link.id)">{{ copy.nav[link.key] }}</a>
      <div class="site-nav__language site-nav__language--mobile" :aria-label="copy.nav.language" role="group">
        <button v-for="language in (['en', 'ru'] as const)" :key="language" type="button" :lang="language" :aria-pressed="locale === language" @click="changeLanguage(language)">{{ language.toUpperCase() }}</button>
      </div>
    </nav>
    <div class="site-nav__language site-nav__language--desktop" :aria-label="copy.nav.language" role="group">
      <button v-for="language in (['en', 'ru'] as const)" :key="language" type="button" :lang="language" :aria-pressed="locale === language" @click="changeLanguage(language)">{{ language.toUpperCase() }}</button>
    </div>
    <button ref="trigger" type="button" class="site-nav__toggle" :aria-expanded="open" aria-controls="site-nav-links" :aria-label="copy.nav.toggle" @click="open = !open"><span></span><span></span><span></span></button>
  </header>
</template>
