<script lang="ts">
  import { onMount, onDestroy } from 'svelte'
  import birthdayHero from './assets/birthday-hero.jpg'

  const targetDate = new Date('2026-09-15T00:00:00+02:00')

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
  let audioContext: AudioContext | null = null
  let stopSong: (() => void) | null = null

  $: isBirthdayRoute = path.startsWith('/geburtstag')
  $: timeLeft = getTimeLeft(now, targetDate)

  const wishes = [
    'Du machst die Welt ein Stück heller — einfach, weil du du bist. Alles Gute, liebe Ece!',
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

  async function toggleHappyBirthday() {
    if (isPlaying) {
      stopHappyBirthday()
      return
    }

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

  function stopHappyBirthday() {
    stopSong?.()
    stopSong = null
  }

  onMount(() => {
    const tick = window.setInterval(() => {
      now = new Date()
    }, 1000)

    const popstate = () => {
      path = window.location.pathname
    }

    window.addEventListener('popstate', popstate)

    return () => {
      window.clearInterval(tick)
      window.removeEventListener('popstate', popstate)
    }
  })

  onDestroy(() => stopHappyBirthday())
</script>

<svelte:head>
  <title>{isBirthdayRoute ? 'Alles Gute, liebe Ece 🎂' : 'Etwas Besonderes erwartet dich…'}</title>
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
      class="music-button"
      type="button"
      aria-label={isPlaying ? 'Musik stumm schalten' : 'Musik abspielen'}
      on:click={toggleHappyBirthday}
    >
      {isPlaying ? '🎵' : '♫'}
    </button>

    <section class="birthday-content">
      <p class="birthday-kicker">HEUTE IST EIN BESONDERER TAG</p>
      <h1 class="birthday-title">Herzlichen Glückwunsch,<br /><span>Ece</span></h1>
      <p class="birthday-subtitle">
        Heute gehört dieser Tag ganz dir. Lass dich feiern, tragen und umhüllen<br class="desktop" />
        — mit Worten, die von Herzen kommen.
      </p>

      <img
        class="hero-image"
        src={birthdayHero}
        alt="Warm beleuchtete Geburtstagsszene mit Blumen und Kerzenschein"
      />

      <blockquote>
        <span>“</span>
        <p>Du machst die Welt ein Stück heller — einfach, weil du du bist. Alles Gute, liebe Ece!</p>
        <span>”</span>
      </blockquote>

      <h2>GEDANKEN FÜR DICH</h2>
      <div class="wish-grid">
        {#each wishes as wish}
          <p>{wish}</p>
        {/each}
      </div>

      <div class="birthday-footer">
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

    <section class="countdown-content">
      <p class="kicker">ETWAS BESONDERES NAHT</p>
      <h1>Die Zeit verrät es noch nicht.<br /><span>Aber sie läuft dir davon.</span></h1>

      <div class="timer" aria-label="Countdown bis 15.09.2026">
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

      <p class="wait-text">Sei geduldig. Manche Dinge sind es wert, darauf zu warten.</p>
    </section>
  </main>
{/if}
