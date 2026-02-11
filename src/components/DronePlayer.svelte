<script lang="ts">
  // ── Data ───────────────────────────────────────────────────
  const ROOTS = [
    { label: "G3", hz: 196.00 },
    { label: "A3", hz: 220.00 },
    { label: "C4", hz: 261.63 },
    { label: "D4", hz: 293.66 },
    { label: "E4", hz: 329.63 },
    { label: "G4", hz: 392.00 },
    { label: "A4", hz: 440.00 },
  ] as const;

  const INTERVALS = [
    { label: "Major 3rd", semitones: 4, pureRatio: 5 / 4 },
    { label: "Perfect 5th", semitones: 7, pureRatio: 3 / 2 },
  ] as const;

  const NOTE_NAMES = ["C","C♯","D","D♯","E","F","F♯","G","G♯","A","A♯","B"];

  function intervalNoteName(rootLabel: string, semitones: number): string {
    const rootNote = rootLabel.replace(/\d/, "");
    const octave = parseInt(rootLabel.replace(/\D/g, ""));
    let idx = NOTE_NAMES.indexOf(rootNote);
    if (idx === -1) idx = 0;
    const newIdx = (idx + semitones) % 12;
    const newOctave = octave + Math.floor((idx + semitones) / 12);
    return `${NOTE_NAMES[newIdx]}${newOctave}`;
  }

  const MASTER_GAIN = 0.15;
  const FADE = 0.3;
  const STAGGER_DELAY = 1.2;

  // Drone harmonics — bowed string
  const HARMONICS: [number, number][] = [
    [1, 1.0], [2, 0.5], [3, 0.35], [4, 0.2], [5, 0.12], [6, 0.08],
  ];

  // Piano harmonics — slightly stretched (inharmonicity), faster decay on higher partials
  const PIANO_HARMONICS: { ratio: number; amp: number; decay: number }[] = [
    { ratio: 1.0,    amp: 1.0,  decay: 3.0  },
    { ratio: 2.001,  amp: 0.6,  decay: 2.2  },
    { ratio: 3.003,  amp: 0.35, decay: 1.5  },
    { ratio: 4.007,  amp: 0.2,  decay: 1.0  },
    { ratio: 5.012,  amp: 0.1,  decay: 0.7  },
    { ratio: 6.02,   amp: 0.05, decay: 0.5  },
  ];

  const PIANO_REPEAT_SEC = 3.0; // re-strike interval

  // ── State ──────────────────────────────────────────────────
  let isPlaying = $state(false);
  let tuningMode = $state<"equal" | "pure">("equal");
  let soundMode = $state<"drone" | "piano">("drone");
  let volume = $state(MASTER_GAIN);
  let phase = $state<"idle" | "root-only" | "both">("idle");
  let selectedRootIdx = $state(3);
  let selectedIntervalIdx = $state(0);

  // ── Audio graph refs — shared ──────────────────────────────
  let ctx: AudioContext | null = null;
  let masterGain: GainNode | null = null;

  // Drone refs
  let rootGain: GainNode | null = null;
  let intGain: GainNode | null = null;
  let rootOscs: OscillatorNode[] = [];
  let intOscs: OscillatorNode[] = [];
  let staggerTimer: ReturnType<typeof setTimeout> | null = null;

  // Piano refs
  let pianoRepeatId: ReturnType<typeof setInterval> | null = null;
  let pianoStaggerTimer: ReturnType<typeof setTimeout> | null = null;

  // ── Derived ────────────────────────────────────────────────
  let rootHz = $derived(ROOTS[selectedRootIdx].hz);
  let rootLabel = $derived(ROOTS[selectedRootIdx].label);
  let interval = $derived(INTERVALS[selectedIntervalIdx]);

  let intervalFreq = $derived(
    tuningMode === "equal"
      ? rootHz * Math.pow(2, interval.semitones / 12)
      : rootHz * interval.pureRatio
  );

  let intNoteName = $derived(intervalNoteName(rootLabel, interval.semitones));
  let freqDisplay = $derived(intervalFreq.toFixed(2));

  let statusText = $derived(
    phase === "both"
      ? tuningMode === "equal" ? "Wobble detected" : "Perfectly In Tune"
      : phase === "root-only" ? "Root sounding…" : "\u00A0"
  );

  let statusColor = $derived(
    phase !== "both" ? "text-slate-400"
      : tuningMode === "equal" ? "text-amber-400" : "text-emerald-400"
  );

  let showPing = $derived(phase === "both" && tuningMode === "equal");

  let dotColor = $derived(
    phase !== "both" ? "bg-slate-500"
      : tuningMode === "equal" ? "bg-amber-400" : "bg-emerald-400"
  );

  // ── Drone: slide freqs live ────────────────────────────────
  $effect(() => {
    const freq = intervalFreq;
    if (soundMode === "drone" && ctx && intOscs.length > 0) {
      intOscs.forEach((osc, i) => {
        osc.frequency.setTargetAtTime(freq * HARMONICS[i][0], ctx!.currentTime, 0.025);
      });
    }
  });

  $effect(() => {
    const freq = rootHz;
    if (soundMode === "drone" && ctx && rootOscs.length > 0) {
      rootOscs.forEach((osc, i) => {
        osc.frequency.setTargetAtTime(freq * HARMONICS[i][0], ctx!.currentTime, 0.025);
      });
    }
  });

  // ── Volume sync ────────────────────────────────────────────
  $effect(() => {
    const v = volume;
    if (masterGain && ctx) {
      masterGain.gain.cancelScheduledValues(ctx.currentTime);
      masterGain.gain.setValueAtTime(masterGain.gain.value, ctx.currentTime);
      masterGain.gain.linearRampToValueAtTime(v, ctx.currentTime + 0.05);
    }
  });

  // ── Drone: build additive voice ────────────────────────────
  function buildVoice(baseFreq: number, dest: GainNode): OscillatorNode[] {
    if (!ctx) return [];
    const oscs: OscillatorNode[] = [];
    for (const [h, amp] of HARMONICS) {
      const g = ctx.createGain();
      g.gain.value = amp;
      g.connect(dest);
      const osc = ctx.createOscillator();
      osc.type = "sine";
      osc.frequency.value = baseFreq * h;
      osc.connect(g);
      oscs.push(osc);
    }
    return oscs;
  }

  // ── Piano: strike a single note ────────────────────────────
  function pianoStrike(freq: number, startTime: number) {
    if (!ctx || !masterGain) return;

    // Hammer transient — short noise burst
    const noiseLen = 0.015;
    const noiseBuf = ctx.createBuffer(1, ctx.sampleRate * noiseLen, ctx.sampleRate);
    const noiseData = noiseBuf.getChannelData(0);
    for (let i = 0; i < noiseData.length; i++) {
      noiseData[i] = (Math.random() * 2 - 1) * 0.3;
    }
    const noiseSrc = ctx.createBufferSource();
    noiseSrc.buffer = noiseBuf;

    const noiseBand = ctx.createBiquadFilter();
    noiseBand.type = "bandpass";
    noiseBand.frequency.value = freq * 4;
    noiseBand.Q.value = 2;

    const noiseGain = ctx.createGain();
    noiseGain.gain.setValueAtTime(0.4, startTime);
    noiseGain.gain.exponentialRampToValueAtTime(0.001, startTime + 0.04);

    noiseSrc.connect(noiseBand).connect(noiseGain).connect(masterGain);
    noiseSrc.start(startTime);
    noiseSrc.stop(startTime + 0.05);

    // Harmonic partials with individual decay envelopes
    for (const h of PIANO_HARMONICS) {
      const osc = ctx.createOscillator();
      osc.type = "sine";
      osc.frequency.value = freq * h.ratio;

      const env = ctx.createGain();
      // Sharp attack
      env.gain.setValueAtTime(0, startTime);
      env.gain.linearRampToValueAtTime(h.amp * 0.5, startTime + 0.005);
      // Natural decay
      env.gain.exponentialRampToValueAtTime(0.001, startTime + h.decay);

      osc.connect(env).connect(masterGain);
      osc.start(startTime);
      osc.stop(startTime + h.decay + 0.05);
    }
  }

  // Strike both notes (root first, interval staggered on first hit)
  function pianoStrikeBoth() {
    if (!ctx) return;
    const now = ctx.currentTime;
    pianoStrike(rootHz, now);
    pianoStrike(intervalFreq, now);
  }

  function pianoStrikeRootOnly() {
    if (!ctx) return;
    pianoStrike(rootHz, ctx.currentTime);
  }

  // ── Start / Stop ───────────────────────────────────────────
  function startDrone() {
    if (!ctx || !masterGain) return;
    const now = ctx.currentTime;

    rootGain = ctx.createGain();
    rootGain.gain.value = 0.5;
    rootGain.connect(masterGain);
    rootOscs = buildVoice(rootHz, rootGain);
    rootOscs.forEach((o) => o.start());

    intGain = ctx.createGain();
    intGain.gain.setValueAtTime(0, now);
    intGain.connect(masterGain);
    intOscs = buildVoice(intervalFreq, intGain);
    intOscs.forEach((o) => o.start());

    phase = "root-only";
    staggerTimer = setTimeout(() => {
      if (ctx && intGain) {
        intGain.gain.setTargetAtTime(0.5, ctx.currentTime, 0.15);
        phase = "both";
      }
    }, STAGGER_DELAY * 1000);
  }

  function stopDrone() {
    if (staggerTimer) { clearTimeout(staggerTimer); staggerTimer = null; }
    const allOsc = [...rootOscs, ...intOscs];
    allOsc.forEach((o) => { try { o.stop(); } catch {} });
    rootGain = null; intGain = null;
    rootOscs = []; intOscs = [];
  }

  function startPiano() {
    // First strike: root only
    pianoStrikeRootOnly();
    phase = "root-only";

    // After stagger, strike both and start repeating
    pianoStaggerTimer = setTimeout(() => {
      pianoStrikeBoth();
      phase = "both";
      pianoRepeatId = setInterval(() => {
        pianoStrikeBoth();
      }, PIANO_REPEAT_SEC * 1000);
    }, STAGGER_DELAY * 1000);
  }

  function stopPiano() {
    if (pianoStaggerTimer) { clearTimeout(pianoStaggerTimer); pianoStaggerTimer = null; }
    if (pianoRepeatId) { clearInterval(pianoRepeatId); pianoRepeatId = null; }
  }

  function startAudio() {
    ctx = new AudioContext();
    const now = ctx.currentTime;

    masterGain = ctx.createGain();
    masterGain.gain.setValueAtTime(0, now);
    masterGain.gain.linearRampToValueAtTime(volume, now + FADE);
    masterGain.connect(ctx.destination);

    isPlaying = true;

    if (soundMode === "drone") {
      startDrone();
    } else {
      startPiano();
    }
  }

  function stopAudio() {
    stopDrone();
    stopPiano();

    if (ctx && masterGain) {
      const now = ctx.currentTime;
      masterGain.gain.setTargetAtTime(0, now, 0.06);
      setTimeout(() => {
        ctx?.close();
        ctx = null; masterGain = null;
      }, 250);
    }
    isPlaying = false;
    phase = "idle";
  }

  function toggle() { isPlaying ? stopAudio() : startAudio(); }

  // ── Switch sound mode while playing ────────────────────────
  function switchSoundMode(mode: "drone" | "piano") {
    if (mode === soundMode) return;
    const wasPlaying = isPlaying;
    if (wasPlaying) stopAudio();
    soundMode = mode;
    if (wasPlaying) {
      // Small delay to let cleanup finish
      setTimeout(() => startAudio(), 300);
    }
  }

  // ── Re-strike piano when tuning/root/interval changes ──────
  $effect(() => {
    // Subscribe to these
    const _r = rootHz;
    const _f = intervalFreq;
    if (soundMode === "piano" && isPlaying && phase === "both") {
      // Re-strike with new frequencies
      pianoStrikeBoth();
    }
  });
</script>

<!-- ── UI ──────────────────────────────────────────────────── -->
<div class="flex flex-col items-center gap-8 w-full max-w-md mx-auto">

  <!-- Selectors row -->
  <div class="flex gap-4 w-full">
    <div class="flex-1">
      <label for="root-select" class="block text-xs text-slate-500 uppercase tracking-widest mb-1.5">Root Note</label>
      <select
        id="root-select"
        bind:value={selectedRootIdx}
        class="w-full bg-slate-800 border border-slate-700 text-slate-200 rounded-lg px-3 py-2 text-sm
               focus:outline-none focus:border-slate-500 cursor-pointer"
      >
        {#each ROOTS as root, i}
          <option value={i}>{root.label} ({root.hz} Hz)</option>
        {/each}
      </select>
    </div>
    <div class="flex-1">
      <label for="interval-select" class="block text-xs text-slate-500 uppercase tracking-widest mb-1.5">Interval</label>
      <select
        id="interval-select"
        bind:value={selectedIntervalIdx}
        class="w-full bg-slate-800 border border-slate-700 text-slate-200 rounded-lg px-3 py-2 text-sm
               focus:outline-none focus:border-slate-500 cursor-pointer"
      >
        {#each INTERVALS as int, i}
          <option value={i}>{int.label}</option>
        {/each}
      </select>
    </div>
  </div>

  <!-- Sound mode toggle -->
  <div class="flex rounded-lg overflow-hidden border border-slate-700">
    <button
      onclick={() => switchSoundMode("drone")}
      class="px-5 py-2 text-sm font-medium transition-colors cursor-pointer
             {soundMode === 'drone' ? 'bg-slate-700 text-slate-100' : 'bg-slate-800/50 text-slate-500 hover:text-slate-300'}"
    >
      🎻 Drone
    </button>
    <button
      onclick={() => switchSoundMode("piano")}
      class="px-5 py-2 text-sm font-medium transition-colors cursor-pointer
             {soundMode === 'piano' ? 'bg-slate-700 text-slate-100' : 'bg-slate-800/50 text-slate-500 hover:text-slate-300'}"
    >
      🎹 Piano
    </button>
  </div>

  <!-- Frequency readout -->
  <div class="text-center space-y-1">
    <p class="text-sm text-slate-500 tracking-widest uppercase">
      {rootLabel} + {interval.label} ({intNoteName})
    </p>
    <p class="text-3xl font-mono tabular-nums text-slate-200">
      {rootHz} Hz &plus; {freqDisplay} Hz
    </p>
  </div>

  <!-- Play / Stop -->
  <button
    onclick={toggle}
    class="group relative w-36 h-36 rounded-full border-2 transition-all duration-300 cursor-pointer
           {isPlaying
             ? 'border-rose-500 bg-rose-500/10 shadow-[0_0_40px_rgba(244,63,94,0.25)]'
             : 'border-slate-600 bg-slate-800 hover:border-slate-400 hover:shadow-[0_0_30px_rgba(148,163,184,0.1)]'}"
    aria-label={isPlaying ? "Stop" : "Play"}
  >
    {#if isPlaying}
      <span class="block mx-auto w-10 h-10 rounded-sm bg-rose-500"></span>
    {:else}
      <svg class="mx-auto w-12 h-12 text-slate-300 group-hover:text-white transition-colors" viewBox="0 0 24 24" fill="currentColor">
        <path d="M8 5v14l11-7z"/>
      </svg>
    {/if}
  </button>

  <!-- Phase indicator -->
  {#if phase === "root-only"}
    <p class="text-xs text-slate-500 animate-pulse -mt-4">Root sounding… interval entering</p>
  {/if}

  <!-- Tuning toggle -->
  <div class="flex items-center gap-4">
    <span class="text-sm font-medium {tuningMode === 'equal' ? 'text-amber-400' : 'text-slate-500'} transition-colors">
      Equal (Piano)
    </span>
    <button
      onclick={() => tuningMode = tuningMode === "equal" ? "pure" : "equal"}
      class="relative w-16 h-8 rounded-full transition-colors duration-300 cursor-pointer
             {tuningMode === 'pure' ? 'bg-emerald-600' : 'bg-amber-600'}"
      role="switch"
      aria-checked={tuningMode === "pure"}
      aria-label="Toggle tuning mode"
    >
      <span
        class="absolute top-1 left-1 w-6 h-6 rounded-full bg-white shadow transition-transform duration-300
               {tuningMode === 'pure' ? 'translate-x-8' : 'translate-x-0'}"
      ></span>
    </button>
    <span class="text-sm font-medium {tuningMode === 'pure' ? 'text-emerald-400' : 'text-slate-500'} transition-colors">
      Pure (Violin)
    </span>
  </div>

  <!-- Status badge -->
  <div class="flex items-center gap-2 px-4 py-2 rounded-full bg-slate-800/80 border border-slate-700">
    <span class="relative flex h-2.5 w-2.5">
      {#if showPing}
        <span class="animate-ping absolute inline-flex h-full w-full rounded-full bg-amber-400 opacity-75"></span>
      {/if}
      <span class="relative inline-flex rounded-full h-2.5 w-2.5 {dotColor}"></span>
    </span>
    <span class="text-sm font-medium {statusColor}">{statusText}</span>
  </div>

  <!-- Volume slider -->
  <div class="flex items-center gap-3 w-56">
    <svg class="w-4 h-4 text-slate-500 shrink-0" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
      <path d="M11 5L6 9H2v6h4l5 4V5z"/>
    </svg>
    <input
      type="range" min="0" max="0.35" step="0.005"
      bind:value={volume}
      class="w-full h-1 bg-slate-700 rounded-lg appearance-none cursor-pointer
             [&::-webkit-slider-thumb]:appearance-none [&::-webkit-slider-thumb]:w-4 [&::-webkit-slider-thumb]:h-4
             [&::-webkit-slider-thumb]:rounded-full [&::-webkit-slider-thumb]:bg-slate-300
             [&::-moz-range-thumb]:w-4 [&::-moz-range-thumb]:h-4
             [&::-moz-range-thumb]:rounded-full [&::-moz-range-thumb]:bg-slate-300 [&::-moz-range-thumb]:border-0"
      aria-label="Volume"
    />
    <svg class="w-5 h-5 text-slate-500 shrink-0" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
      <path d="M11 5L6 9H2v6h4l5 4V5z"/>
      <path d="M19.07 4.93a10 10 0 010 14.14M15.54 8.46a5 5 0 010 7.08"/>
    </svg>
  </div>
</div>
