<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import { MOBILE_SCROLL_QUERY, usesNativeScroll } from '../mobileScroll'
import female from '../../assets/section04/female_flight_alpha.png'
import male from '../../assets/section04/male_flight_alpha.png'
import '../section04.css'

const root = ref<HTMLElement>()
gsap.registerPlugin(ScrollTrigger)
let trigger: ScrollTrigger | undefined
let flightFrame = 0
let flightHeight = innerHeight
let flightWidth = innerWidth
let lastProgress = -1
let flight: gsap.core.Timeline | undefined
let idle: gsap.core.Timeline | undefined
let motionQuery: MediaQueryList | undefined
let scrollQuery: MediaQueryList | undefined
let played = false
let active = false
const finalReview = new URLSearchParams(location.search).get('s04static') === '1'
const isReduced = () => matchMedia('(prefers-reduced-motion: reduce)').matches || finalReview

function parts() {
  const section = root.value!
  return {
    female: section.querySelector<SVGImageElement>('.s04-female')!,
    male: section.querySelector<SVGImageElement>('.s04-male')!,
    title: section.querySelector<HTMLElement>('.s04-title')!,
  }
}
function stopIdle() { idle?.kill(); idle = undefined }
function updateScrollFlight() {
  flightFrame = 0
  if (!root.value || !usesNativeScroll() || isReduced() || !flight) return
  const top = root.value.getBoundingClientRect().top
  const progress = Math.min(1, Math.max(0, (flightHeight * .78 - top) / (flightHeight * .66)))
  if (progress === lastProgress) return
  lastProgress = progress
  flight.progress(progress).pause()
  root.value.dataset.flightProgress = progress.toFixed(3)
  if (progress < 1) {
    stopIdle()
    const { female, male } = parts()
    gsap.set([female, male], { y: 0 })
  } else if (active && !idle) startIdle()
}
function queueScrollFlight() {
  if (flightWidth !== innerWidth) {
    flightWidth = innerWidth
    flightHeight = parseFloat(getComputedStyle(document.documentElement).getPropertyValue('--mobile-viewport-height')) || innerHeight
    lastProgress = -1
  }
  if (usesNativeScroll() && !flightFrame) flightFrame = requestAnimationFrame(updateScrollFlight)
}
function startIdle() {
  stopIdle()
  if (!active || document.hidden || isReduced() || !root.value) return
  const { female, male } = parts()
  idle = gsap.timeline({ repeat: -1 })
    .to([female, male], { y: -2.5, duration: 2.8, ease: 'sine.inOut', yoyo: true, repeat: 1 }, 0)
}
function setFinal() {
  if (!root.value) return
  const { female, male, title } = parts()
  gsap.set(female, { attr: { x: 65 }, y: 0 })
  gsap.set(male, { attr: { x: 872 }, y: 0 })
  gsap.set(title, { opacity: 1, y: 0 })
}
function enter() {
  active = true
  if (isReduced()) { setFinal(); return }
  if (usesNativeScroll()) {
    // Native document scrolling owns the flight's progress, including reverse.
    if (flight && flight.progress() >= 1) startIdle()
    return
  }
  if (played) { if (flight && !flight.isActive() && flight.progress() < 1) flight.resume(); else startIdle(); return }
  played = true
  const { female, male, title } = parts()
  flight = gsap.timeline({ onComplete: startIdle })
    .to(female, { attr: { x: 65 }, duration: 1.3, ease: 'power2.out' }, 0)
    .to(male, { attr: { x: 872 }, duration: 1.3, ease: 'power2.out' }, 0)
    .to(title, { opacity: 1, y: 0, duration: .68, ease: 'power2.out' }, 1.43)
}
function leave() { active = false; flight?.pause(); stopIdle() }
function visibility() {
  if (document.hidden) { flight?.pause(); idle?.pause() }
  else if (active && !isReduced()) {
    if (usesNativeScroll()) { if (flight && flight.progress() >= 1) startIdle() }
    else if (flight && flight.progress() < 1) flight.resume()
    else if (played) startIdle()
  }
}
function setup() {
  trigger?.kill(); trigger = undefined
  cancelAnimationFrame(flightFrame); flightFrame = 0; lastProgress = -1
  flight?.kill(); flight = undefined
  stopIdle()
  if (!root.value) return
  played = false; active = false
  root.value.classList.toggle('s04-reduced', isReduced())
  if (isReduced()) { setFinal(); return }
  const { female, male, title } = parts()
  gsap.set([female, male], { clearProps: 'opacity' })
  if (usesNativeScroll()) {
    setFinal()
    gsap.set(female, { attr: { x: -860 } })
    gsap.set(male, { attr: { x: 1760 } })
    flight = gsap.timeline({ paused: true })
      .to(female, { attr: { x: 65 }, duration: 1, ease: 'power2.out' }, 0)
      .to(male, { attr: { x: 872 }, duration: 1, ease: 'power2.out' }, 0)
    // One passive scroll update per frame, without ScrollTrigger's touch
    // polling delay. The native document never pins or writes scroll position.
    flightHeight = parseFloat(getComputedStyle(document.documentElement).getPropertyValue('--mobile-viewport-height')) || innerHeight
    queueScrollFlight()
  } else {
    gsap.set(female, { attr: { x: -860 }, y: 0 })
    gsap.set(male, { attr: { x: 1760 }, y: 0 })
    gsap.set(title, { opacity: 0, y: 20 })
  }
  trigger = ScrollTrigger.create({ trigger: root.value, start: 'top 74%', end: 'bottom top', invalidateOnRefresh: true, onEnter: enter, onEnterBack: enter, onLeave: leave, onLeaveBack: leave })
}
onMounted(() => {
  motionQuery = matchMedia('(prefers-reduced-motion: reduce)')
  motionQuery.addEventListener('change', setup)
  scrollQuery = matchMedia(MOBILE_SCROLL_QUERY)
  scrollQuery.addEventListener('change', setup)
  document.addEventListener('visibilitychange', visibility)
  window.addEventListener('scroll', queueScrollFlight, { passive: true })
  window.addEventListener('resize', queueScrollFlight, { passive: true })
  setup()
  ScrollTrigger.refresh()
})
onBeforeUnmount(() => {
  motionQuery?.removeEventListener('change', setup)
  scrollQuery?.removeEventListener('change', setup)
  document.removeEventListener('visibilitychange', visibility)
  window.removeEventListener('scroll', queueScrollFlight)
  window.removeEventListener('resize', queueScrollFlight)
  cancelAnimationFrame(flightFrame)
  trigger?.kill(); flight?.kill(); stopIdle()
})
</script>

<template>
  <section id="section-04" ref="root" class="s04" aria-labelledby="s04-title">
    <div class="s04-boundary" aria-hidden="true"></div>
    <div class="s04-viewport">
      <h2 id="s04-title" class="s04-title"><span>LET'S</span><span>CONNECT<span class="s04-period">.</span></span></h2>
      <div class="s04-scene" aria-hidden="true">
        <svg class="s04-stage" viewBox="0 0 1672 941" xmlns="http://www.w3.org/2000/svg" preserveAspectRatio="xMidYMid meet">
          <image class="s04-female" :href="female" x="65" y="360" width="819" height="552" />
          <image class="s04-male" :href="male" x="872" y="315" width="751" height="513" />
        </svg>
      </div>
      <p class="s04-signoff">PEOPLE <span>×</span> TECH <span>×</span> A BRIGHTER TOMORROW</p>
    </div>
  </section>
</template>
