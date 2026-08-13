<script lang="ts">
  import { onMount, onDestroy } from 'svelte'

  const targetDate = new Date('2026-09-15T00:00:00+02:00')
  const targetLabel = '15.09.2026'

  type TimeLeft = {
    total: number
    days: number
    hours: number
    minutes: number
    seconds: number
  }

  let now = new Date()
  let isBirthday = false
  let isPlaying = false
  let showConfetti = false
  let audioContext: AudioContext | null = null
  let stopSong: (() => void) | null = null

  $: path = window.location.pathname
  $: isBirthdayRoute = path.startsWith('/geburtstag')
  $: timeLeft = getTimeLeft(now, targetDate)
  $: isUnlocked = isBirthdayRoute || timeLeft.total <= 0 || isBirthday

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

  function goBirthday() {
    history.pushState({}, '', '/geburtstag')
    path = window.location.pathname
    isBirthday = true
    showConfetti = true
    void playHappyBirthday()
  }

  function goHome() {
    history.pushState({}, '', '/')
    path = window.location.pathname
    isBirthday = false
    showConfetti = false
    stopHappyBirthday()
  }

  function createOscillator(ctx: AudioContext, frequency: number, start: number, duration: number) {
    const oscillator = ctx.createOscillator()
    const gain = ctx.createGain()

    oscillator.type = 'triangle'
    oscillator.frequency.setValueAtTime(frequency, start)
    gain.gain.setValueAtTime(0.0001, start)
    gain.gain.exponentialRampToValueAtTime(0.18, start + 0.03)
    gain.gain.exponentialRampToValueAtTime(0.0001, start + duration)

    oscillator.connect(gain)
    gain.connect(ctx.destination)
    oscillator.start(start)
    oscillator.stop(start + duration + 0.03)
  }

  async function playHappyBirthday() {
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

    if (isBirthdayRoute) {
      showConfetti = true
    }

    return () => {
      window.clearInterval(tick)
      window.removeEventListener('popstate', popstate)
    }
  })

  onDestroy(() => stopHappyBirthday())
</script>

<svelte:head>
  <title>{isUnlocked ? 'Alles Gute zum Geburtstag!' : 'Mystery Countdown'}</title>
  <meta
    name="description"
    content="A mysterious countdown to 15.09.2026 with a birthday surprise and Happy Birthday music."
  />
</svelte:head>

<main class:party={isUnlocked}>
  <div class="background-orb orb-one"></div>
  <div class="background-orb orb-two"></div>
  <div class="background-orb orb-three"></div>

  {#if showConfetti || isUnlocked}
    <div class="confetti" aria-hidden="true">
      {#each Array(34) as _, i}
        <i style={`--i: ${i}; --delay: ${-(i % 13) * 0.38}s; --left: ${(i * 29) % 100}%; --hue: ${(i * 37) % 360}deg`}></i>
      {/each}
    </div>
  {/if}

  <section class="card" aria-live="polite">
    <div class="eyebrow">{isUnlocked ? 'Überraschung freigeschaltet' : 'Mystery Countdown'}</div>

    {#if isUnlocked}
      <div class="cake" aria-hidden="true">
        <span class="flame"></span>
        <span class="candle"></span>
        <span class="icing"></span>
        <span class="base"></span>
      </div>
      <h1>Alles Gute zum Geburtstag!</h1>
      <p class="subtitle">
        Heute ist dein Tag. Ich wünsche dir ganz viel Liebe, Glück, Kuchen und magische Momente. 🎉
      </p>

      <div class="actions">
        <button class="primary" type="button" on:click={playHappyBirthday}>
          {isPlaying ? 'Happy Birthday läuft …' : 'Happy Birthday Song abspielen'}
        </button>
        <button class="secondary" type="button" on:click={goHome}>Countdown ansehen</button>
      </div>
    {:else}
      <div class="lock" aria-hidden="true">?</div>
      <h1>Etwas Besonderes wartet auf dich</h1>
      <p class="subtitle">Der Countdown läuft bis zum {targetLabel}. Komm zurück, wenn die Zeit abgelaufen ist.</p>

      <div class="timer" aria-label={`Countdown bis ${targetLabel}`}>
        <div class="time-box">
          <strong>{timeLeft.days}</strong>
          <span>Tage</span>
        </div>
        <div class="time-box">
          <strong>{pad(timeLeft.hours)}</strong>
          <span>Stunden</span>
        </div>
        <div class="time-box">
          <strong>{pad(timeLeft.minutes)}</strong>
          <span>Minuten</span>
        </div>
        <div class="time-box">
          <strong>{pad(timeLeft.seconds)}</strong>
          <span>Sekunden</span>
        </div>
      </div>

      <button class="ghost" type="button" on:click={goBirthday}>Geburtstagsseite öffnen</button>
    {/if}
  </section>
</main>
