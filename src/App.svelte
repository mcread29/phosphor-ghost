<script lang="ts">
  import CrtScreen from "./CrtScreen.svelte";

  const defaults = {
    bloom: 0.98,
    scanlineSize: 5,
    scanlineDepth: 0.46,
    scanlineSpeed: 8,
    flicker: 0.02,
    warp: 0.075,
    chromatic: 1.35,
    glitchIntensity: 0.7,
    glitchFrequency: 4,
  };

  const settingsKey = "signal-os-crt-defaults";
  const isDev = import.meta.env.DEV;

  function loadSettings() {
    if (!isDev) return { ...defaults };
    try {
      const saved = JSON.parse(localStorage.getItem(settingsKey) ?? "null") as Partial<typeof defaults> | null;
      return saved ? { ...defaults, ...saved } : { ...defaults };
    } catch {
      return { ...defaults };
    }
  }

  let armed = $state(false);
  let channel = $state(4);
  let controlsOpen = $state(false);
  let saveLabel = $state("SAVE DEFAULT");
  let settings = $state(loadSettings());

  function saveSettings() {
    localStorage.setItem(settingsKey, JSON.stringify(settings));
    saveLabel = "SAVED";
    window.setTimeout(() => (saveLabel = "SAVE DEFAULT"), 1200);
  }
  const controls = [
    { key: "bloom", label: "Bloom", min: 0, max: 2, step: 0.01 },
    { key: "scanlineSize", label: "Line size", min: 2, max: 12, step: 0.25 },
    { key: "scanlineDepth", label: "Line depth", min: 0, max: 0.8, step: 0.01 },
    { key: "scanlineSpeed", label: "Line speed", min: -20, max: 20, step: 0.5 },
    { key: "flicker", label: "Flicker", min: 0, max: 0.12, step: 0.002 },
    { key: "warp", label: "Screen warp", min: 0, max: 0.2, step: 0.0025 },
    { key: "chromatic", label: "RGB split", min: 0, max: 6, step: 0.05 },
    { key: "glitchIntensity", label: "Glitch strength", min: 0, max: 2, step: 0.02 },
    { key: "glitchFrequency", label: "Glitch interval", min: 0.5, max: 12, step: 0.1 },
  ] as const;
  const logs = [
    "Carrier handshake accepted",
    "Remote archive mounted",
    "Telemetry stream nominal",
    "Awaiting operator command",
  ];
  const waveform = Array.from({ length: 42 }, (_, index) => ({
    id: index,
    height: 18 + ((index * 31) % 72),
  }));
</script>

<CrtScreen {...settings}>
  <main class="terminal">
    <header class="topbar">
      <a class="brand" href="#top" aria-label="Signal OS home"><span class="brand-mark">S/</span> SIGNAL.OS</a>
      <div class="status"><i></i> UPLINK STABLE · CH {channel}</div>
    </header>
    <section class="hero" id="top">
      <div class="eyebrow">RESTRICTED SYSTEM // NODE 07</div>
      <h1>THE SIGNAL<br /><span>IS ALIVE.</span></h1>
      <p class="lede">A live interface rendered through an experimental WebGL phosphor display. The DOM stays real. The signal does not.</p>
      <div class="actions">
        <button class={['primary', { armed }]} onclick={() => (armed = !armed)}>{armed ? "LINK ARMED" : "ESTABLISH LINK"}</button>
        <button class="ghost" onclick={() => (channel = (channel % 9) + 1)}>SCAN CHANNELS</button>
      </div>
    </section>
    <section class="dashboard" aria-label="System telemetry">
      <article class="panel signal-panel">
        <div class="panel-heading"><span>WAVEFORM</span><b>LIVE</b></div>
        <div class="waveform" aria-hidden="true">
          {#each waveform as bar (bar.id)}<i style:height={`${bar.height}%`}></i>{/each}
        </div>
        <div class="metrics"><span>12.048 <small>MHz</small></span><span>-42 <small>dBm</small></span></div>
      </article>
      <article class="panel log-panel">
        <div class="panel-heading"><span>EVENT LOG</span><b>04 ITEMS</b></div>
        <ol>{#each logs as log, index (log)}<li><time>00:0{index + 2}:1{index}</time>{log}</li>{/each}</ol>
      </article>
    </section>
    <footer><span>BUILD 4.12.88</span><span>WEBGL CRT PROTOCOL</span><span>© 2088 SIGNAL INDUSTRIES</span></footer>
  </main>
</CrtScreen>

{#if isDev}
  <div class="dev-overlay">
    <button class="tune-toggle" onclick={() => (controlsOpen = !controlsOpen)} aria-expanded={controlsOpen}>
      {controlsOpen ? "CLOSE" : "TUNE CRT"}
    </button>
    {#if controlsOpen}
      <aside class="tune-panel" aria-label="CRT post-processing controls">
        <div class="tune-heading">
          <strong>POST PROCESSING</strong>
          <div><button onclick={() => (settings = { ...defaults })}>RESET</button><button onclick={saveSettings}>{saveLabel}</button></div>
        </div>
        {#each controls as control (control.key)}
          <label>
            <span>{control.label}<output>{settings[control.key].toFixed(control.step < 0.01 ? 3 : 2)}</output></span>
            <input type="range" min={control.min} max={control.max} step={control.step} bind:value={settings[control.key]} />
          </label>
        {/each}
      </aside>
    {/if}
  </div>
{/if}
