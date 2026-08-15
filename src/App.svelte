<script lang="ts">
  import { onMount, onDestroy } from 'svelte'
  import birthdayHero from './assets/birthday-hero.jpg'

  const productionTargetDate = new Date('2026-09-15T00:00:00+02:00')

  type TimeLeft = {
    total: number
    days: number
    hours: number
    minutes: number
    seconds: number
  }

  let now = new Date()
  let selectedTargetDate = productionTargetDate
  let smoothTargetDate: Date | null = null
  let graceCountdownStarted = false
  // Dev controls are intentionally hidden for production. Set this to true to comment them back in.
  const showDevControls = false
  let devControlsOpen = false
  let targetInputValue = formatDateTimeLocal(productionTargetDate)
  let path = window.location.pathname
  let isPlaying = false
  let revealScheduled = false
  let showFirework = false
  let birthdayAutoplayTried = false
  let audioContext: AudioContext | null = null
  let revealObserver: IntersectionObserver | null = null
  let stopSong: (() => void) | null = null

  $: isBirthdayRoute = path.startsWith('/geburtstag')
  $: activeTargetDate = smoothTargetDate ?? selectedTargetDate
  $: isGraceCountdown = !!smoothTargetDate && !showFirework
  $: timeLeft = getTimeLeft(now, activeTargetDate)
  $: isCountdownDone = timeLeft.total === 0
  $: if (
    !isBirthdayRoute &&
    graceCountdownStarted &&
    smoothTargetDate &&
    now.getTime() >= smoothTargetDate.getTime() &&
    !revealScheduled
  ) {
    revealScheduled = true
    showFirework = true
    window.setTimeout(forwardToBirthday, 5000)
  }

  $: if (isBirthdayRoute && !birthdayAutoplayTried) {
    birthdayAutoplayTried = true
    window.setTimeout(() => {
      revealVisibleBirthdayContent()
      observeRevealNodes()
      void startHappyBirthday().catch(() => undefined)
    }, 0)
  }

  const wishes = [
    'Du machst die Welt ein Stück heller — einfach, weil du du bist. Alles Gute, Ece!',
    'Möge dein neues Lebensjahr so warm und hell erstrahlen wie dein Lachen.',
    'Jeder Tag mit dir ist ein Geschenk. Danke, dass es dich gibt.',
    'Ein Jahr älter, ein Jahr weiser, ein Jahr mehr geliebt.',
    'Heute feiern wir dich — denn du bist der Grund zum Feiern.',
    'Mögen all deine Wünsche ihren Weg zu dir finden.',
    'Du bist nicht nur ein Jahr älter, sondern ein Jahr wunderbarer.',
    'Träume groß, liebe tief, lass die Seele frei — dieses Jahr gehört dir.',
  ]

  const fireworkBursts = [
    { x: 50, y: 36, delay: 0 },
    { x: 34, y: 46, delay: 180 },
    { x: 66, y: 45, delay: 360 },
    { x: 44, y: 58, delay: 760 },
    { x: 58, y: 57, delay: 1080 },
    { x: 25, y: 34, delay: 1480 },
    { x: 75, y: 33, delay: 1880 },
    { x: 18, y: 52, delay: 2460 },
    { x: 82, y: 54, delay: 3080 },
    { x: 42, y: 31, delay: 3580 },
    { x: 61, y: 41, delay: 4020 },
  ]

  const fireworkColors = ['#fff3bd', '#ff4f8b', '#46e7ff', '#b47cff', '#7dff8a', '#ffb347']

  function getTimeLeft(current: Date, target: Date): TimeLeft {
    const total = Math.max(0, target.getTime() - current.getTime())
    const secondsTotal = Math.floor(total / 1000)

    return {
      total,
      days: Math.floor(secondsTotal / 86400),
      hours: Math.floor((secondsTotal % 86400) / 3600),
      minutes: Math.floor((secondsTotal % 3600) / 60),
      seconds: secondsTotal % 60,
    }
  }

  function pad(value: number) {
    return value.toString().padStart(2, '0')
  }

  function formatDateTimeLocal(date: Date) {
    const year = date.getFullYear()
    const month = pad(date.getMonth() + 1)
    const day = pad(date.getDate())
    const hours = pad(date.getHours())
    const minutes = pad(date.getMinutes())

    return `${year}-${month}-${day}T${hours}:${minutes}`
  }

  function startGraceCountdown() {
    now = new Date()
    smoothTargetDate = new Date(Date.now() + 3_000)
    graceCountdownStarted = true
    revealScheduled = false
    showFirework = false
  }

  function applyTargetDate() {
    const parsed = new Date(targetInputValue)
    if (Number.isNaN(parsed.getTime())) return

    selectedTargetDate = parsed
    resetCountdown()
  }

  function resetToProductionDate() {
    selectedTargetDate = productionTargetDate
    targetInputValue = formatDateTimeLocal(productionTargetDate)
    resetCountdown()
  }

  function setExpiredTestDate() {
    selectedTargetDate = new Date(Date.now() - 1_000)
    targetInputValue = formatDateTimeLocal(selectedTargetDate)
    resetCountdown()
  }

  function resetCountdown() {
    now = new Date()
    revealScheduled = false
    showFirework = false

    if (selectedTargetDate.getTime() <= now.getTime()) {
      smoothTargetDate = new Date(Date.now() + 3_000)
      graceCountdownStarted = true
    } else {
      smoothTargetDate = null
      graceCountdownStarted = false
    }
  }

  function forwardToBirthday() {
    showFirework = false
    navigate('/geburtstag')
  }

  function navigate(to: string) {
    history.pushState({}, '', to)
    path = window.location.pathname

    if (to === '/') {
      birthdayAutoplayTried = false
      resetCountdown()
      return
    }

    window.setTimeout(revealVisibleBirthdayContent, 0)
  }

  function createOscillator(
    ctx: AudioContext,
    frequency: number,
    start: number,
    duration: number,
    gainLevel = 0.1,
    type: OscillatorType = 'triangle',
  ) {
    const oscillator = ctx.createOscillator()
    const gain = ctx.createGain()

    oscillator.type = type
    oscillator.frequency.setValueAtTime(frequency, start)
    gain.gain.setValueAtTime(0.0001, start)
    gain.gain.exponentialRampToValueAtTime(gainLevel, start + 0.035)
    gain.gain.exponentialRampToValueAtTime(Math.max(gainLevel * 0.56, 0.0002), start + duration * 0.62)
    gain.gain.exponentialRampToValueAtTime(0.0001, start + duration)

    oscillator.connect(gain)
    gain.connect(ctx.destination)
    oscillator.start(start)
    oscillator.stop(start + duration + 0.05)
  }

  async function startHappyBirthday() {
    if (isPlaying) return

    stopHappyBirthday()

    const AudioCtor = window.AudioContext || window.webkitAudioContext
    audioContext = new AudioCtor()
    await audioContext.resume()

    const notes: Array<[string, number]> = [
      ['G4', 0.34], ['G4', 0.18], ['A4', 0.54], ['G4', 0.54], ['C5', 0.54], ['B4', 0.92],
      ['G4', 0.34], ['G4', 0.18], ['A4', 0.54], ['G4', 0.54], ['D5', 0.54], ['C5', 0.92],
      ['G4', 0.34], ['G4', 0.18], ['G5', 0.54], ['E5', 0.54], ['C5', 0.54], ['B4', 0.54], ['A4', 0.92],
      ['F5', 0.34], ['F5', 0.18], ['E5', 0.54], ['C5', 0.54], ['D5', 0.54], ['C5', 1.12],
    ]

    const frequencies: Record<string, number> = {
      C3: 130.81,
      G3: 196.0,
      C4: 261.63,
      D4: 293.66,
      E4: 329.63,
      F4: 349.23,
      G4: 392.0,
      A4: 440.0,
      B4: 493.88,
      C5: 523.25,
      D5: 587.33,
      E5: 659.25,
      F5: 698.46,
      G5: 783.99,
    }

    const harmonies: Record<string, string> = {
      A4: 'F4',
      B4: 'G4',
      C5: 'E4',
      D5: 'F4',
      E5: 'G4',
      F5: 'A4',
      G5: 'C5',
    }
    const bassRoots = ['C3', 'G3', 'C3', 'F4']

    let start = audioContext.currentTime + 0.08
    for (let repeat = 0; repeat < 3; repeat += 1) {
      notes.forEach(([note, duration], index) => {
        const phrase = Math.min(Math.floor(index / 6), bassRoots.length - 1)
        if (index % 6 === 0) {
          createOscillator(audioContext!, frequencies[bassRoots[phrase]], start, 2.85, 0.034, 'sine')
        }

        createOscillator(audioContext!, frequencies[note], start, duration, 0.095, 'triangle')

        const harmony = harmonies[note]
        if (harmony && index % 2 === 0) {
          createOscillator(audioContext!, frequencies[harmony], start + 0.015, duration * 0.92, 0.028, 'sine')
        }

        if (index % 5 === 2) {
          createOscillator(audioContext!, frequencies[note] * 2, start + 0.025, duration * 0.42, 0.012, 'sine')
        }

        start += duration + 0.08
      })
      start += 0.45
    }

    isPlaying = true
    const timeout = window.setTimeout(() => {
      void audioContext?.close()
      audioContext = null
      isPlaying = false
      stopSong = null
    }, (start - audioContext.currentTime + 0.25) * 1000)

    stopSong = () => {
      window.clearTimeout(timeout)
      void audioContext?.close()
      audioContext = null
      isPlaying = false
    }
  }

  function toggleHappyBirthday() {
    if (isPlaying) {
      stopHappyBirthday()
      return
    }

    void startHappyBirthday()
  }

  function stopHappyBirthday() {
    stopSong?.()
    void audioContext?.close()
    audioContext = null
    isPlaying = false
    stopSong = null
  }

  function revealVisibleBirthdayContent() {
    document.querySelectorAll<HTMLElement>('.reveal').forEach((node) => {
      const rect = node.getBoundingClientRect()
      if (rect.top < window.innerHeight * 0.92) {
        node.classList.add('is-visible')
      }
    })
  }

  function observeRevealNodes() {
    revealObserver ??= new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            entry.target.classList.add('is-visible')
            revealObserver?.unobserve(entry.target)
          }
        })
      },
      { threshold: 0.08, rootMargin: '0px 0px 12% 0px' },
    )

    document.querySelectorAll<HTMLElement>('.reveal:not(.is-visible)').forEach((node) => {
      revealObserver?.observe(node)
    })
  }

  onMount(() => {
    const tick = window.setInterval(() => {
      now = new Date()
      if (
        !isBirthdayRoute &&
        selectedTargetDate.getTime() <= now.getTime() &&
        !graceCountdownStarted &&
        !revealScheduled
      ) {
        startGraceCountdown()
      }
    }, 1000)

    if (!isBirthdayRoute) {
      resetCountdown()
    }

    observeRevealNodes()

    const popstate = () => {
      path = window.location.pathname
      if (path === '/') {
        resetCountdown()
      }
      window.setTimeout(observeRevealNodes, 0)
    }

    const pageshow = () => {
      path = window.location.pathname
      if (path === '/') {
        resetCountdown()
      }
    }

    const startAfterInteraction = () => {
      if (isBirthdayRoute && !isPlaying) {
        void startHappyBirthday().catch(() => undefined)
      }
    }

    window.addEventListener('popstate', popstate)
    window.addEventListener('pageshow', pageshow)

    if (isBirthdayRoute) {
      void startHappyBirthday().catch(() => {
        isPlaying = false
        window.addEventListener('pointerdown', startAfterInteraction, { once: true })
        window.addEventListener('keydown', startAfterInteraction, { once: true })
      })
    }

    return () => {
      window.clearInterval(tick)
      revealObserver?.disconnect()
      revealObserver = null
      window.removeEventListener('popstate', popstate)
      window.removeEventListener('pageshow', pageshow)
      window.removeEventListener('pointerdown', startAfterInteraction)
      window.removeEventListener('keydown', startAfterInteraction)
    }
  })

  onDestroy(() => stopHappyBirthday())
</script>

<svelte:head>
  <title>{isBirthdayRoute ? 'Alles Gute, Ece 🎂' : 'Etwas Besonderes erwartet dich…'}</title>
  <meta
    name="description"
    content="Ein geheimnisvoller Countdown bis zum 15.09.2026 und eine liebevolle Geburtstagsseite."
  />
</svelte:head>

{#if isBirthdayRoute}
  <main class="birthday-scene">
    <div class="soft-dots" aria-hidden="true">
      {#each Array(22) as _, i}
        <span style={`--i:${i}; --left:${(i * 37 + 3) % 100}%; --top:${(i * 29 + 2) % 42}%; --size:${8 + (i % 4) * 5}px; --delay:${-(i % 8) * 0.7}s`}></span>
      {/each}
    </div>

    <button
      class:muted={!isPlaying}
      class="music-button"
      type="button"
      aria-label={isPlaying ? 'Musik stumm schalten' : 'Musik abspielen'}
      on:click={toggleHappyBirthday}
    >
      <span aria-hidden="true">♪</span>
    </button>

    <section class="birthday-content">
      <p class="birthday-kicker reveal">HEUTE IST EIN BESONDERER TAG</p>
      <h1 class="birthday-title reveal">Herzlichen Glückwunsch,<br /><span>Ece</span></h1>
      <p class="birthday-subtitle reveal">
        Heute gehört dieser Tag ganz dir. Lass dich feiern, tragen und umhüllen<br class="desktop" />
        — mit Worten, die von Herzen kommen.
      </p>

      <img
        class="hero-image reveal float-card"
        src={birthdayHero}
        alt="Warm beleuchtete Geburtstagsszene mit Blumen und Kerzenschein"
      />

      <blockquote class="reveal">
        <span>“</span>
        <p>Du machst die Welt ein Stück heller — einfach, weil du du bist. Alles Gute, Ece!</p>
        <span>”</span>
      </blockquote>

      <h2 class="reveal">GEDANKEN FÜR DICH</h2>
      <div class="wish-grid">
        {#each wishes as wish, i}
          <p class="reveal wish-card" style={`--delay:${(i % 4) * 90}ms`}>{wish}</p>
        {/each}
      </div>

      <div class="birthday-footer reveal">
        <div></div>
        <p>Alles Liebe zum Geburtstag, Ece.</p>
        <a href="/" on:click|preventDefault={() => navigate('/')}>← <span>Zurück zum Anfang</span></a>
      </div>
    </section>
  </main>
{:else}
  <main class="countdown-scene">
    <div class="stars" aria-hidden="true">
      {#each Array(52) as _, i}
        <span style={`--left:${(i * 47 + 11) % 100}%; --top:${(i * 31 + 5) % 100}%; --size:${1.1 + (i % 4) * 0.55}px; --delay:${-(i % 9) * 0.45}s; --opacity:${0.35 + (i % 5) * 0.13}`}></span>
      {/each}
    </div>

    {#if showFirework}
      <div class="firework-layer" aria-hidden="true">
        {#each fireworkBursts as burst}
          <div class="firework-burst" style={`--x:${burst.x}%; --y:${burst.y}%; --delay:${burst.delay}ms`}>
            {#each Array(26) as _, spark}
              <span style={`--angle:${spark * 16.36 + (burst.x % 13) - (spark % 4) * 5}deg; --distance:${66 + ((spark * 17 + burst.x) % 8) * 12}px; --fall:${-36 + ((spark * 19 + burst.y) % 11) * 10}px; --spark-size:${4 + ((spark + burst.delay) % 5)}px; --color:${fireworkColors[(spark + burst.delay) % fireworkColors.length]}`}></span>
            {/each}
          </div>
        {/each}
        <div class="firework-glitter">
          {#each Array(58) as _, i}
            <span style={`--left:${(i * 23 + 7) % 100}%; --top:${(i * 37 + 11) % 70}%; --delay:${(i % 18) * 210}ms; --color:${fireworkColors[i % fireworkColors.length]}`}></span>
          {/each}
        </div>
      </div>
    {/if}

    <section class="countdown-content">
      <p class="kicker countdown-enter">
        {#if isCountdownDone}
          DAS WARTEN IST VORBEI
        {:else if isGraceCountdown}
          DIE ÜBERRASCHUNG STARTET
        {:else}
          ETWAS BESONDERES NAHT
        {/if}
      </p>
      <h1 class="countdown-enter delay-1">
        {#if isCountdownDone}
          Gleich wird das Geheimnis gelüftet.
        {:else if isGraceCountdown}
          Einen kleinen Moment noch.<br /><span>Die Magie beginnt gleich.</span>
        {:else}
          Die Zeit verrät es noch nicht.<br /><span>Aber sie läuft dir davon.</span>
        {/if}
      </h1>

      <div class:complete={isCountdownDone} class="timer countdown-enter delay-2" aria-label="Countdown bis 15.09.2026">
        <div class="unit">
          <div class="number-wrap"><i></i><span>{timeLeft.days}</span></div>
          <small>TAGE</small>
        </div>
        <b>:</b>
        <div class="unit">
          <div class="number-wrap"><i></i><span>{pad(timeLeft.hours)}</span></div>
          <small>STUNDEN</small>
        </div>
        <b>:</b>
        <div class="unit">
          <div class="number-wrap"><i></i><span>{pad(timeLeft.minutes)}</span></div>
          <small>MINUTEN</small>
        </div>
        <b>:</b>
        <div class="unit">
          <div class="number-wrap"><i></i><span>{pad(timeLeft.seconds)}</span></div>
          <small>SEKUNDEN</small>
        </div>
      </div>

      <p class="wait-text countdown-enter delay-3">
        {#if isCountdownDone}
          Öffne die Überraschung…
        {:else if isGraceCountdown}
          Der Countdown war bereit — jetzt gibt es noch einen sanften 3-Sekunden-Start.
        {:else}
          Sei geduldig. Manche Dinge sind es wert, darauf zu warten.
        {/if}
      </p>

      {#if showDevControls}
        <div class="dev-controls countdown-enter delay-3" class:open={devControlsOpen}>
          <button type="button" class="dev-toggle" on:click={() => (devControlsOpen = !devControlsOpen)}>
            Test-Zeit wählen
          </button>

          {#if devControlsOpen}
            <div class="dev-panel">
              <label for="target-date">Countdown-Ziel</label>
              <input
                id="target-date"
                type="datetime-local"
                bind:value={targetInputValue}
                on:change={applyTargetDate}
              />
              <div class="dev-actions">
                <button type="button" on:click={applyTargetDate}>Übernehmen</button>
                <button type="button" on:click={setExpiredTestDate}>Jetzt 3s testen</button>
                <button type="button" on:click={resetToProductionDate}>15.09.2026</button>
              </div>
            </div>
          {/if}
        </div>
      {/if}
    </section>
  </main>
{/if}
