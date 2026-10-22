<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'
import gsap from 'gsap'
import { ScrollTrigger } from 'gsap/ScrollTrigger'
import { copy } from '../i18n'
import { MOBILE_SCROLL_QUERY, usesNativeScroll } from '../mobileScroll'
import '../section02.css'

gsap.registerPlugin(ScrollTrigger)
const root = ref<HTMLElement>()
const track = ref<HTMLElement>()
const count = ref(1)
const reel = ref<HTMLElement>()
const chartValues = [20, 28, 35, 44, 53, 65, 76, 91]
let context: gsap.Context | undefined
let geoCycle: gsap.core.Timeline | undefined
let trafficTimeline: gsap.core.Timeline | undefined
let chartTimeline: gsap.core.Timeline | undefined
let trigger: ScrollTrigger | undefined
let motionQuery: MediaQueryList | undefined
let motionHandler: (() => void) | undefined
let resizeTimer = 0
let currentScene = 0
let active = false
let transitioning = false
let lastWheel = 0
let entryWheelDirection = 0
let wheelTotal = 0
let wheelConsumed = false
let cooldownUntil = 0
let touchStart: { x: number; y: number } | undefined
let scrollQuery: MediaQueryList | undefined
let sceneObserver: IntersectionObserver | undefined
let desktopInput = false
let layoutWidth = innerWidth
const mobileVisible = new Set<Element>()
const mobileEntrances: gsap.core.Tween[] = []
function drawLengths(selector: string) {
  root.value?.querySelectorAll<SVGPathElement>(selector).forEach(path => {
    // The SVG stretches to the layout box; a longer dash covers its rendered path at every aspect ratio.
    const length = path.getTotalLength() * 2
    gsap.set(path, { strokeDasharray: length, strokeDashoffset: length })
  })
}

function resetTraffic() {
  gsap.set('.s02-traffic', { clearProps: 'boxShadow,borderColor' })
  gsap.set('.s02-connection', { opacity: 1, clearProps: 'strokeDasharray,strokeDashoffset' })
  gsap.set('.s02-connection-flow', { opacity: 0, strokeDasharray: '22 900', strokeDashoffset: 0 })
  gsap.set('.s02-card', { opacity: .75, clearProps: 'borderColor' })
}
function resetChart() {
  gsap.set('.s02-chart-bar', { scaleY: .08, transformOrigin: 'bottom center' })
  drawLengths('.s02-chart-line')
  gsap.set('.s02-chart-line', { opacity: 1 })
  gsap.set('.s02-chart-point', { opacity: .18, scale: .55, transformOrigin: 'center center', clearProps: 'filter' })
}
function trimConnections() {
  const svg = root.value?.querySelector<SVGSVGElement>('.s02-system-lines')
  if (!svg) return
  const bounds = svg.getBoundingClientRect()
  if (!bounds.width || !bounds.height) return
  const targets = ['.s02-traffic', '.s02-card-geo', '.s02-card-brand', '.s02-card-conditions', '.s02-card-execution']
  const holes = svg.querySelectorAll<SVGRectElement>('.s02-connector-cutout')
  const clearance = innerWidth <= 900 ? 6 : 12
  targets.forEach((selector, index) => {
    const card = root.value?.querySelector<HTMLElement>(selector)
    const hole = holes[index]
    if (!card || !hole) return
    const rect = card.getBoundingClientRect()
    hole.setAttribute('x', String((rect.left - bounds.left - clearance) / bounds.width * 640))
    hole.setAttribute('y', String((rect.top - bounds.top - clearance) / bounds.height * 470))
    hole.setAttribute('width', String((rect.width + clearance * 2) / bounds.width * 640))
    hole.setAttribute('height', String((rect.height + clearance * 2) / bounds.height * 470))
  })
}
function createTrafficLoop() {
  resetTraffic()
  return gsap.timeline({ repeat: -1, repeatDelay: .35 })
      .to('.s02-traffic', { boxShadow: '0 0 24px rgba(0,239,252,.2)', borderColor: '#00effc', duration: .25 }, 0)
      .to('.s02-card', { opacity: 1, borderColor: '#00dce9', duration: .28, stagger: .06 }, .34)
      .to('.s02-connection-flow', { opacity: .95, duration: .15, stagger: .16 }, .5)
      .to('.s02-connection-flow', { strokeDashoffset: (_index, path: SVGPathElement) => -path.getTotalLength() - 22, duration: 2, stagger: .16, ease: 'none' }, .5)
      .to('.s02-traffic', { boxShadow: '0 0 12px rgba(0,239,252,.09)', duration: 1.1, ease: 'sine.inOut' }, 1.1)
      .set('.s02-connection-flow', { opacity: 0, strokeDashoffset: 0 }, '>-0.05')
}
function createChartLoop() {
  resetChart()
  return gsap.timeline({ repeat: -1, repeatDelay: .55 })
      .to('.s02-chart-bar', { scaleY: 1, duration: .9, ease: 'power2.out' })
      .to('.s02-chart-line', { strokeDashoffset: 0, opacity: 1, duration: .85, ease: 'power2.out' }, .2)
      .to('.s02-chart-point', { opacity: 1, scale: 1, duration: .23, stagger: .1 }, .4)
      .to('.s02-chart-point:last-child', { filter: 'drop-shadow(0 0 9px #00effc)', duration: .25 }, 1.15)
      .to({}, { duration: 1.15 })
      .to('.s02-chart-bar', { scaleY: .82, duration: .9, ease: 'sine.inOut' })
      .to('.s02-chart-line', { opacity: .58, duration: .65 }, '<')
      .to('.s02-chart-point', { opacity: .62, duration: .55 }, '<')
}
function playScene(index: number) {
  geoCycle?.pause()
  trafficTimeline?.kill()
  chartTimeline?.kill()
  if (index === 0) {
    reel.value?.style.setProperty('--geo-index', '0')
    geoCycle?.restart()
  } else if (index === 1) trafficTimeline = createTrafficLoop()
  else chartTimeline = createChartLoop()
}
function syncMobileScenes() {
  const loops = [geoCycle, trafficTimeline, chartTimeline]
  root.value?.querySelectorAll('.s02-scene').forEach((scene, index) => {
    const running = mobileVisible.has(scene) && !document.hidden && !motionQuery?.matches
    scene.setAttribute('data-loop-active', String(running))
    if (running) loops[index]?.play()
    else loops[index]?.pause()
  })
}
function observeMobileScenes() {
  const revealed = new Set<Element>()
  sceneObserver = new IntersectionObserver(entries => {
    for (const entry of entries) {
      if (entry.isIntersecting) {
        mobileVisible.add(entry.target)
        if (!revealed.has(entry.target)) {
          revealed.add(entry.target)
          mobileEntrances.push(gsap.fromTo(entry.target.querySelectorAll('.s02-copy,.s02-visual'),
            { opacity: .6, y: 12 }, { opacity: 1, y: 0, duration: .42, ease: 'power2.out', clearProps: 'opacity,transform' }))
        }
      } else mobileVisible.delete(entry.target)
    }
    syncMobileScenes()
  }, { threshold: .05 })
  root.value?.querySelectorAll('.s02-scene').forEach(scene => sceneObserver!.observe(scene))
}
function setDesktopInput(enabled: boolean) {
  if (desktopInput === enabled) return
  desktopInput = enabled
  if (enabled) {
    window.addEventListener('wheel', wheel, { passive: false })
    window.addEventListener('touchstart', touchBegin, { passive: true })
    window.addEventListener('touchmove', touchMove, { passive: false })
    window.addEventListener('touchend', touchEnd, { passive: true })
    window.addEventListener('keydown', keydown)
  } else {
    window.removeEventListener('wheel', wheel)
    window.removeEventListener('touchstart', touchBegin)
    window.removeEventListener('touchmove', touchMove)
    window.removeEventListener('touchend', touchEnd)
    window.removeEventListener('keydown', keydown)
  }
}
function enter(index: number) {
  active = true
  if (root.value) root.value.dataset.sceneActive = 'true'
  transitioning = false
  currentScene = index
  count.value = index + 1
  wheelTotal = 0
  wheelConsumed = false
  cooldownUntil = 0
  entryWheelDirection = index === 0 ? 1 : -1
  if (track.value) gsap.set(track.value, { x: -index * innerWidth })
  playScene(index)
}
function leave(direction: number) {
  active = false
  if (root.value) root.value.dataset.sceneActive = 'false'
  transitioning = false
  geoCycle?.pause()
  trafficTimeline?.pause()
  chartTimeline?.pause()
  entryWheelDirection = 0
  if (trigger) window.scrollTo({
    top: direction > 0 ? trigger.end + innerHeight + 2 : Math.max(0, trigger.start - 2),
    behavior: matchMedia('(prefers-reduced-motion: reduce)').matches ? 'instant' : 'smooth',
  })
}
function step(direction: number) {
  if (usesNativeScroll() || !active || transitioning || performance.now() < cooldownUntil) return
  cooldownUntil = performance.now() + 1150
  const next = currentScene + direction
  if (next < 0 || next > 2) { leave(direction); return }
  transitioning = true
  geoCycle?.pause()
  trafficTimeline?.kill()
  chartTimeline?.kill()
  currentScene = next
  count.value = next + 1
  if (!track.value) return
  gsap.to(track.value, {
    x: -next * innerWidth,
    duration: 1,
    ease: 'power2.inOut',
    overwrite: true,
    onComplete: () => {
      transitioning = false
      if (active && !document.hidden) playScene(next)
    },
  })
}
function wheel(event: WheelEvent) {
  if (usesNativeScroll()) return
  const now = performance.now()
  const delta = event.deltaY * (event.deltaMode === 1 ? 16 : event.deltaMode === 2 ? innerHeight : 1)
  if (!active) { lastWheel = now; return }
  event.preventDefault()
  // A single wheel burst can carry the page across the pin boundary. Consume
  // its remaining events, while allowing an immediate reversal back to Hero.
  if (entryWheelDirection && Math.sign(delta) === entryWheelDirection && now - lastWheel < 280) {
    lastWheel = now
    return
  }
  entryWheelDirection = 0
  if (now - lastWheel > 280) { wheelTotal = 0; wheelConsumed = false }
  lastWheel = now
  if (wheelConsumed || transitioning) return
  wheelTotal += delta
  if (Math.abs(wheelTotal) < 52) return
  wheelConsumed = true
  step(Math.sign(wheelTotal))
}
function touchBegin(event: TouchEvent) {
  if (usesNativeScroll() || !active || event.touches.length !== 1) return
  touchStart = { x: event.touches[0]!.clientX, y: event.touches[0]!.clientY }
}
function touchMove(event: TouchEvent) {
  if (usesNativeScroll() || !active || !touchStart || event.touches.length !== 1) return
  const dx = event.touches[0]!.clientX - touchStart.x
  const dy = event.touches[0]!.clientY - touchStart.y
  if (Math.abs(dy) > Math.abs(dx) && Math.abs(dy) > 8) event.preventDefault()
}
function touchEnd(event: TouchEvent) {
  if (!active || !touchStart || !event.changedTouches.length) { touchStart = undefined; return }
  const dx = event.changedTouches[0]!.clientX - touchStart.x
  const dy = event.changedTouches[0]!.clientY - touchStart.y
  touchStart = undefined
  if (Math.abs(dy) > 60 && Math.abs(dy) > Math.abs(dx) * 1.2) step(dy < 0 ? 1 : -1)
}
function keydown(event: KeyboardEvent) {
  if (usesNativeScroll() || !active || event.repeat || event.altKey || event.ctrlKey || event.metaKey) return
  const down = event.key === 'PageDown' || event.key === 'ArrowDown' || (event.key === ' ' && !event.shiftKey)
  const up = event.key === 'PageUp' || event.key === 'ArrowUp' || (event.key === ' ' && event.shiftKey)
  if (!down && !up) return
  const target = event.target as HTMLElement
  if (target.closest('input, textarea, select, button, a, [contenteditable="true"]')) return
  event.preventDefault()
  step(down ? 1 : -1)
}
function resize() {
  if (usesNativeScroll() && layoutWidth === innerWidth) return
  layoutWidth = innerWidth
  clearTimeout(resizeTimer)
  resizeTimer = window.setTimeout(() => {
    if (!usesNativeScroll()) {
      if (track.value && !matchMedia('(prefers-reduced-motion: reduce)').matches) gsap.set(track.value, { x: -currentScene * innerWidth })
      ScrollTrigger.refresh()
    }
    trimConnections()
  }, 120)
}

function setup() {
  setDesktopInput(false)
  sceneObserver?.disconnect(); sceneObserver = undefined
  mobileVisible.clear()
  mobileEntrances.splice(0).forEach(tween => tween.revert())
  active = false
  if (root.value) root.value.dataset.sceneActive = 'false'
  transitioning = false
  trafficTimeline?.kill()
  chartTimeline?.kill()
  if (track.value) gsap.killTweensOf(track.value)
  context?.revert()
  trigger = undefined
  geoCycle?.kill()
  context = undefined
  geoCycle = undefined
  if (!root.value || !track.value) return
  const section = root.value
  const strip = track.value
  const reduced = matchMedia('(prefers-reduced-motion: reduce)').matches
  const native = usesNativeScroll()
  layoutWidth = innerWidth
  touchStart = undefined
  section.classList.toggle('s02-native', native)
  section.dataset.scrollMode = native ? 'native' : 'desktop'
  section.querySelectorAll('.s02-scene').forEach(scene => scene.removeAttribute('data-loop-active'))
  section.classList.toggle('s02-reduced', reduced)
  if (reduced) {
    count.value = 1
    trimConnections()
    return
  }
  context = gsap.context(() => {
    gsap.set(strip, { x: 0 })
    resetTraffic()
    resetChart()

    geoCycle = gsap.timeline({ paused: true, repeat: -1 })
    copy.value.s02.countries.forEach((_, index) => {
      geoCycle!.call(() => {
        if (reel.value) reel.value.style.setProperty('--geo-index', String(index))
      }, [], index * 1.25)
    })
    geoCycle.to({}, { duration: copy.value.s02.countries.length * 1.25 }, 0)

    if (native) {
      // Each visual owns one loop. Visibility controls playback, never scrolling.
      trafficTimeline = createTrafficLoop().pause(0)
      chartTimeline = createChartLoop().pause(0)
      return
    }
    trigger = ScrollTrigger.create({
      trigger: section,
      start: 'top top',
      end: () => `+=${Math.max(innerHeight * 3, 1200)}`,
      pin: true,
      invalidateOnRefresh: true,
      anticipatePin: 1,
      onEnter: () => enter(0),
      onEnterBack: () => enter(2),
      onLeave: () => { active = false; section.dataset.sceneActive = 'false'; geoCycle?.pause(); trafficTimeline?.pause(); chartTimeline?.pause() },
      onLeaveBack: () => { active = false; section.dataset.sceneActive = 'false'; geoCycle?.pause(); trafficTimeline?.pause(); chartTimeline?.pause() },
    })
  }, section)
  if (native) observeMobileScenes()
  else { setDesktopInput(true); ScrollTrigger.refresh() }
  trimConnections()
}

onMounted(() => {
  motionQuery = matchMedia('(prefers-reduced-motion: reduce)')
  motionHandler = () => setup()
  motionQuery.addEventListener('change', motionHandler)
  scrollQuery = matchMedia(MOBILE_SCROLL_QUERY)
  scrollQuery.addEventListener('change', setup)
  window.addEventListener('resize', resize)
  setup()
  void document.fonts.ready.then(trimConnections)
  document.addEventListener('visibilitychange', visibility)
  window.addEventListener('ftdlab:performance-start', performanceStart)
  window.addEventListener('ftdlab:performance-end', performanceEnd)
})
function performanceStart() {
  if (usesNativeScroll() || matchMedia('(prefers-reduced-motion: reduce)').matches) { root.value?.scrollIntoView({ behavior: 'instant' }); return }
  if (!trigger) return
  window.scrollTo({ top: trigger.start + 2, behavior: 'instant' })
  requestAnimationFrame(() => enter(0))
}
function performanceEnd() {
  if (usesNativeScroll() || matchMedia('(prefers-reduced-motion: reduce)').matches) { root.value?.scrollIntoView({ behavior: 'instant' }); return }
  if (!trigger) return
  window.scrollTo({ top: trigger.end - 2, behavior: 'instant' })
  requestAnimationFrame(() => enter(2))
}
function visibility() {
  if (usesNativeScroll()) { syncMobileScenes(); return }
  if (document.hidden) { geoCycle?.pause(); trafficTimeline?.pause(); chartTimeline?.pause() }
  else if (active && !transitioning) playScene(currentScene)
}
onBeforeUnmount(() => {
  document.removeEventListener('visibilitychange', visibility)
  window.removeEventListener('ftdlab:performance-start', performanceStart)
  window.removeEventListener('ftdlab:performance-end', performanceEnd)
  motionQuery?.removeEventListener('change', motionHandler!)
  scrollQuery?.removeEventListener('change', setup)
  setDesktopInput(false)
  sceneObserver?.disconnect()
  mobileEntrances.splice(0).forEach(tween => tween.revert())
  window.removeEventListener('resize', resize)
  clearTimeout(resizeTimer)
  geoCycle?.kill()
  trafficTimeline?.kill()
  chartTimeline?.kill()
  if (track.value) gsap.killTweensOf(track.value)
  context?.revert()
})
</script>

<template>
  <section id="section-02" ref="root" class="s02" :aria-label="copy.s02.aria">
    <div class="s02-frame">
      <div ref="track" class="s02-track">
        <article class="s02-scene s02-scene-one" aria-labelledby="s02-title-one">
          <div class="s02-copy">
            <span class="s02-eyebrow"><b>01</b><i></i>{{ copy.s02.who }}</span>
            <h2 id="s02-title-one">BUILT<br>AROUND<br><em>PERFORMANCE.</em></h2>
            <p>{{ copy.s02.description1 }}</p>
          </div>
          <div class="s02-visual s02-geo s02-phone" :aria-label="copy.s02.geoAria">
            <div class="s02-phone-top"><span>FTD.LAB</span><i></i><small>01 / 05</small></div>
            <div class="s02-geo-header"><span>GEO</span><small>{{ copy.s02.geoNote }}</small></div>
            <div ref="reel" class="s02-reel" aria-hidden="true">
              <div v-for="(country, index) in copy.s02.countries" :key="index" class="s02-country" :style="{ '--country-index': index }">{{ country }}</div>
            </div>
            <div class="s02-geo-footer"><span>{{ copy.s02.reach }}</span><span>01 — 05</span></div>
          </div>
          <span class="s02-scene-index">01 / 03</span>
        </article>

        <article class="s02-scene s02-scene-two" aria-labelledby="s02-title-two">
          <div class="s02-copy">
            <span class="s02-eyebrow"><b>02</b><i></i>{{ copy.s02.think }}</span>
            <h2 id="s02-title-two">THE RIGHT<br><em>COMBINATION</em><br>CREATES<br>RESULTS.</h2>
            <p>{{ copy.s02.description2 }}</p>
          </div>
          <div class="s02-visual s02-system" :aria-label="copy.s02.systemAria">
            <svg class="s02-system-lines" viewBox="0 0 640 470" preserveAspectRatio="none" aria-hidden="true">
              <defs><mask id="s02-card-line-mask" x="0" y="0" width="640" height="470" maskUnits="userSpaceOnUse" maskContentUnits="userSpaceOnUse"><rect width="640" height="470" fill="white" /><rect v-for="index in 5" :key="index" class="s02-connector-cutout" fill="black" /></mask></defs>
              <g mask="url(#s02-card-line-mask)">
              <path class="s02-connection" d="M320 235 C230 235 210 105 140 105" />
              <path class="s02-connection" d="M320 235 C410 235 430 105 500 105" />
              <path class="s02-connection" d="M320 235 C230 235 210 365 140 365" />
              <path class="s02-connection" d="M320 235 C410 235 430 365 525 374" />
              <path class="s02-connection-flow" d="M320 235 C230 235 210 105 140 105" />
              <path class="s02-connection-flow" d="M320 235 C410 235 430 105 500 105" />
              <path class="s02-connection-flow" d="M320 235 C230 235 210 365 140 365" />
              <path class="s02-connection-flow" d="M320 235 C410 235 430 365 525 374" />
              </g>
            </svg>
            <div class="s02-traffic">{{ copy.s02.traffic }}<span>{{ copy.s02.central }}</span></div>
            <div class="s02-card s02-card-geo">GEO <small>01</small></div>
            <div class="s02-card s02-card-brand">{{ copy.s02.brand }} <small>02</small></div>
            <div class="s02-card s02-card-conditions">{{ copy.s02.conditions }} <small>03</small></div>
            <div class="s02-card s02-card-execution">{{ copy.s02.execution }} <small>04</small></div>
          </div>
          <span class="s02-scene-index">02 / 03</span>
        </article>

        <article class="s02-scene s02-scene-three" aria-labelledby="s02-title-three">
          <div class="s02-copy">
            <span class="s02-eyebrow"><b>03</b><i></i>{{ copy.s02.scale }}</span>
            <h2 id="s02-title-three">SCALING<br><em>IS A RESULT</em><br>OF <br class="s02-mobile-break">PERFORMANCE.</h2>
            <p>{{ copy.s02.description3 }}</p>
          </div>
          <div class="s02-visual s02-chart" role="img" :aria-label="copy.s02.chartAria">
            <div class="s02-chart-grid"></div>
            <div class="s02-bars"><div v-for="(value, index) in chartValues" :key="index" class="s02-chart-bar" :style="{ height: `${value}%` }"></div></div>
            <svg viewBox="0 0 640 400" preserveAspectRatio="none" aria-hidden="true">
              <path class="s02-chart-line" d="M38 320 L120 294 L202 274 L284 236 L366 199 L448 151 L530 111 L606 38" />
              <circle v-for="(point, index) in [[38,320],[120,294],[202,274],[284,236],[366,199],[448,151],[530,111],[606,38]]" :key="index" class="s02-chart-point" :cx="point[0]" :cy="point[1]" r="5" />
            </svg>
            <span class="s02-chart-caption">{{ copy.s02.scale }} ↗</span>
          </div>
          <span class="s02-scene-index">03 / 03</span>
        </article>
      </div>
      <div class="s02-progress" aria-hidden="true"><span>{{ copy.s02.scroll }}</span><div><i :class="{ active: count === 1 }"><small>01</small></i><i :class="{ active: count === 2 }"><small>02</small></i><i :class="{ active: count === 3 }"><small>03</small></i></div><span>0{{ count }} / 03</span></div>
    </div>
  </section>
</template>
