<script lang="ts">
  import { onMount, onDestroy } from 'svelte'
  import birthdayHero from './assets/birthday-hero.jpg'

  // Dev preview: make the countdown end 10 seconds after each page load.
  // Restore to new Date('2026-09-15T00:00:00+02:00') for production.
  const targetDate = new Date(Date.now() + 10_000)

  type TimeLeft = {
    total: number
    days: number
    hours: number
    minutes: number
    seconds: number
  }

  let now = new Date()
  let path = window.location.pathname
  let isPlaying = false
  let revealScheduled = false
  let showFirework = false
  let birthdayAutoplayTried = false
  let audioContext: AudioContext | null = null
  let revealObserver: IntersectionObserver | null = null
  let stopSong: (() => void) | null = null

  $: isBirthdayRoute = path.startsWith('/geburtstag')
  $: timeLeft = getTimeLeft(now, targetDate)
  $: isCountdownDone = timeLeft.total === 0
  $: if (!isBirthdayRoute && isCountdownDone && !revealScheduled) {
    revealScheduled = true
    showFirework = true
    window.setTimeout(() => {
      window.location.assign('/geburtstag')
    }, 1800)
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

  function navigate(to: string) {
    history.pushState({}, '', to)
    path = window.location.pathname
    window.setTimeout(revealVisibleBirthdayContent, 0)
  }

  function createOscillator(ctx: AudioContext, frequency: number, start: number, duration: number) {
    const oscillator = ctx.createOscillator()
    const gain = ctx.createGain()

    oscillator.type = 'triangle'
    oscillator.frequency.setValueAtTime(frequency, start)
    gain.gain.setValueAtTime(0.0001, start)
    gain.gain.exponentialRampToValueAtTime(0.12, start + 0.03)
    gain.gain.exponentialRampToValueAtTime(0.0001, start + duration)

    oscillator.connect(gain)
    gain.connect(ctx.destination)
    oscillator.start(start)
    oscillator.stop(start + duration + 0.03)
  }

  async function startHappyBirthday() {
    if (isPlaying) return

    stopHappyBirthday()

    const AudioCtor = window.AudioContext || window.webkitAudioContext
    audioContext = new AudioCtor()
    await audioContext.resume()

    const notes: Array<[string, number]> = [
      ['G4', 0.35], ['G4', 0.18], ['A4', 0.55], ['G4', 0.55], ['C5', 0.55], ['B4', 0.9],
      ['G4', 0.35], ['G4', 0.18], ['A4', 0.55], ['G4', 0.55], ['D5', 0.55], ['C5', 0.9],
      ['G4', 0.35], ['G4', 0.18], ['G5', 0.55], ['E5', 0.55], ['C5', 0.55], ['B4', 0.55], ['A4', 0.9],
      ['F5', 0.35], ['F5', 0.18], ['E5', 0.55], ['C5', 0.55], ['D5', 0.55], ['C5', 1.1],
    ]

    const frequencies: Record<string, number> = {
      G4: 392.0,
      A4: 440.0,
      B4: 493.88,
      C5: 523.25,
      D5: 587.33,
      E5: 659.25,
      F5: 698.46,
      G5: 783.99,
    }

    let start = audioContext.currentTime + 0.08
    notes.forEach(([note, duration]) => {
      createOscillator(audioContext!, frequencies[note], start, duration)
      start += duration + 0.08
    })

    isPlaying = true
    const timeout = window.setTimeout(() => {
      isPlaying = false
      stopSong = null
    }, (start - audioContext.currentTime + 0.5) * 1000)

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
    }, 1000)

    observeRevealNodes()

    const popstate = () => {
      path = window.location.pathname
      window.setTimeout(observeRevealNodes, 0)
    }

    const startAfterInteraction = () => {
      if (isBirthdayRoute && !isPlaying) {
        void startHappyBirthday().catch(() => undefined)
      }
    }

    window.addEventListener('popstate', popstate)

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
        {#each Array(3) as _, burst}
          <div class="firework-burst" style={`--x:${burst === 0 ? 50 : burst === 1 ? 34 : 66}%; --y:${burst === 0 ? 38 : burst === 1 ? 47 : 46}%; --delay:${burst * 180}ms`}>
            {#each Array(14) as _, spark}
              <span style={`--angle:${spark * 25.7}deg; --distance:${72 + (spark % 3) * 12}px; --color:${spark % 4 === 0 ? '#fff3bd' : spark % 4 === 1 ? '#ff8fb0' : spark % 4 === 2 ? '#9fe8ff' : '#c6a8ff'}`}></span>
            {/each}
          </div>
        {/each}
      </div>
    {/if}

    <section class="countdown-content">
      <p class="kicker countdown-enter">{isCountdownDone ? 'DAS WARTEN IST VORBEI' : 'ETWAS BESONDERES NAHT'}</p>
      <h1 class="countdown-enter delay-1">
        {#if isCountdownDone}
          Gleich wird das Geheimnis gelüftet.
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
        {isCountdownDone ? 'Öffne die Überraschung…' : 'Sei geduldig. Manche Dinge sind es wert, darauf zu warten.'}
      </p>
    </section>
  </main>
{/if}
