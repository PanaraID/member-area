<script lang="ts">
  import { onMount } from 'svelte';
  import PdfViewer from './PdfViewer.svelte';

  const videos = [
    {id:'T-NCfLfHLVA',title:'Video 1 – Playlist Member Area',desc:'Konten eksklusif dari playlist Member Area.'},
  ];

  const tools = [
    {icon:'🔍',name:'Google',url:'https://www.google.com/webhp?igu=1'},
    {icon:'📚',name:'Panara Course',url:'https://panara.id'},
    {icon:'📰',name:'Berita',url:'https://news.google.com/foryou?hl=id&gl=ID&ceid=ID:id'},
    {icon:'📊',name:'Canva',url:'https://www.canva.com'},
    {icon:'🎨',name:'Remove.bg',url:'https://www.remove.bg'},
    {icon:'📝',name:'Notion',url:'https://www.notion.so'},
  ];

  let ebookStarted = $state(false);

  let curIdx     = $state(-1);
  let nowTitle   = $state('Pilih video dari playlist di bawah ↓');
  let nowDesc    = $state('Klik salah satu video untuk mulai menonton.');
  let toolUrl    = $state('— Pilih tool di atas —');
  let toolSrc    = $state('about:blank');
  let toolFrameVisible = $state(false);
  let activeTool = $state(-1);

  // ── YouTube Player API ──────────────────────────────────────────────
  let ytPlayer: any = null;
  let playerReady  = $state(false);
  let isPlaying    = $state(false);
  let currentTime  = $state(0);
  let duration     = $state(0);
  let volume       = $state(100);
  let isMuted      = $state(false);
  let isFullscreen = $state(false);
  let progressInterval: ReturnType<typeof setInterval> | null = null;
  let availableQualities = $state<string[]>([]);
  let currentQuality     = $state('auto');
  let qualityOpen        = $state(false);

  const qualityLabel: Record<string, string> = {
    hd2160: '4K', hd1440: '1440p', hd1080: '1080p',
    hd720: '720p', large: '480p', medium: '360p',
    small: '240p', tiny: '144p', auto: 'Auto',
  };

  function createPlayer(videoId: string) {
    const win = window as any;
    if (!win.YT?.Player) return;
    ytPlayer = new win.YT.Player('ytPlayerContainer', {
      host: 'https://www.youtube-nocookie.com',
      videoId,
      playerVars: {
        autoplay: 1, rel: 0, modestbranding: 1,
        iv_load_policy: 3, playsinline: 1,
        controls: 0, disablekb: 1, fs: 0,
      },
      events: { onReady: onPlayerReady, onStateChange: onPlayerStateChange },
    });
  }

  function onPlayerReady() {
    playerReady = true;
    volume   = ytPlayer.getVolume?.()  ?? 100;
    isMuted  = ytPlayer.isMuted?.()    ?? false;
    duration = ytPlayer.getDuration?.() ?? 0;
    refreshQualities();
  }

  function refreshQualities() {
    const q: string[] = ytPlayer?.getAvailableQualityLevels?.() ?? [];
    availableQualities = q.length ? q : [];
    currentQuality = ytPlayer?.getPlaybackQuality?.() ?? 'auto';
  }

  function setQuality(q: string) {
    if (!ytPlayer || !playerReady) return;
    ytPlayer.setPlaybackQuality(q);
    currentQuality = q;
    qualityOpen = false;
  }

  function onPlayerStateChange(event: any) {
    const s = event.data;
    isPlaying = s === 1;
    if (s === 1) {
      startProgress();
      refreshQualities();
    } else {
      stopProgress();
      if (s === 0 && curIdx < videos.length - 1) {
        setTimeout(() => playVideo(curIdx + 1), 1200);
      }
    }
  }

  function startProgress() {
    stopProgress();
    progressInterval = setInterval(() => {
      if (ytPlayer && playerReady) {
        currentTime = ytPlayer.getCurrentTime?.() ?? 0;
        if (!duration || duration < 1) duration = ytPlayer.getDuration?.() ?? 0;
      }
    }, 500);
  }

  function stopProgress() {
    if (progressInterval) { clearInterval(progressInterval); progressInterval = null; }
  }

  function togglePlay() {
    if (!ytPlayer || !playerReady) return;
    isPlaying ? ytPlayer.pauseVideo() : ytPlayer.playVideo();
  }

  function stopVideo() {
    if (!ytPlayer || !playerReady) return;
    ytPlayer.stopVideo();
    currentTime = 0;
    isPlaying = false;
  }

  function playPrev() { if (curIdx > 0) playVideo(curIdx - 1); }
  function playNext() { if (curIdx < videos.length - 1) playVideo(curIdx + 1); }

  function seekTo(e: MouseEvent) {
    if (!ytPlayer || !playerReady || !duration) return;
    const bar  = e.currentTarget as HTMLElement;
    const rect = bar.getBoundingClientRect();
    const pct  = Math.max(0, Math.min(1, (e.clientX - rect.left) / rect.width));
    ytPlayer.seekTo(pct * duration, true);
    currentTime = pct * duration;
  }

  function handleVolume(e: Event) {
    if (!ytPlayer || !playerReady) return;
    const val = parseInt((e.target as HTMLInputElement).value);
    volume = val;
    ytPlayer.setVolume(val);
    if (val === 0) { ytPlayer.mute(); isMuted = true; }
    else if (isMuted) { ytPlayer.unMute(); isMuted = false; }
  }

  function toggleMute() {
    if (!ytPlayer || !playerReady) return;
    if (isMuted) {
      ytPlayer.unMute(); isMuted = false;
      if (volume === 0) { volume = 50; ytPlayer.setVolume(50); }
    } else {
      ytPlayer.mute(); isMuted = true;
    }
  }

  function toggleFullscreen() {
    const box = document.querySelector('.video-player-box') as HTMLElement;
    if (!box) return;
    if (!document.fullscreenElement) box.requestFullscreen?.();
    else document.exitFullscreen?.();
  }

  function fmt(s: number) {
    if (!s || isNaN(s)) return '0:00';
    return `${Math.floor(s / 60)}:${String(Math.floor(s % 60)).padStart(2,'0')}`;
  }

  function playVideo(idx: number) {
    curIdx   = idx;
    const v  = videos[idx];
    nowTitle = v.title;
    nowDesc  = v.desc;
    currentTime = 0;
    duration    = 0;
    isPlaying   = false;

    if (ytPlayer && playerReady) {
      ytPlayer.loadVideoById(v.id);
    } else if (!ytPlayer) {
      playerReady = false;
      createPlayer(v.id);
    }

    setTimeout(() => {
      document.querySelector('.video-player-box')?.scrollIntoView({behavior:'smooth',block:'start'});
    }, 50);
  }

  function selectTool(idx: number) {
    activeTool = idx;
    toolUrl    = tools[idx].url;
    toolSrc    = tools[idx].url;
    toolFrameVisible = true;
    setTimeout(() => {
      document.querySelector('.tools-frame-box')?.scrollIntoView({behavior:'smooth',block:'start'});
    }, 50);
  }

  onMount(() => {
    const win = window as any;
    win.onYouTubeIframeAPIReady = () => { /* API ready; createPlayer called on first playVideo */ };

    const tag = document.createElement('script');
    tag.src = 'https://www.youtube.com/iframe_api';
    document.head.appendChild(tag);

    document.addEventListener('fullscreenchange', () => {
      isFullscreen = !!document.fullscreenElement;
    });

    document.addEventListener('contextmenu', e => e.preventDefault());
    document.addEventListener('keydown', (e: KeyboardEvent) => {
      if (
        e.key === 'F12' ||
        (e.ctrlKey && ['s','u','p'].includes(e.key.toLowerCase())) ||
        (e.ctrlKey && e.shiftKey && ['i','j','c'].includes(e.key.toLowerCase()))
      ) e.preventDefault();
    });
    document.addEventListener('dragstart', e => e.preventDefault());

    const obs = new IntersectionObserver(entries => {
      entries.forEach(e => {
        if (e.isIntersecting) { e.target.classList.add('visible'); obs.unobserve(e.target); }
      });
    }, {threshold: 0.06});
    document.querySelectorAll('.reveal').forEach(el => obs.observe(el));

    return () => { obs.disconnect(); stopProgress(); };
  });
</script>

<!-- NAVBAR -->
<nav class="navbar">
  <div class="nav-brand">
    <div class="nav-dot"></div>
    Member Area
  </div>
  <a href="https://demo.sheetflare.net/member" class="nav-dashboard">
    🏠 Dashboard
  </a>
</nav>

<!-- HERO -->
<div class="hero wrap">
  <div class="hero-badge">✦ Exclusive Access</div>
  <h1>Selamat Datang<br>di Member Area</h1>
  <p>Akses penuh ke video, ebook, dan tools eksklusif. Semua tersedia langsung di halaman ini.</p>
</div>

<!-- WRAP -->
<div class="wrap">

  <!-- VIDEO -->
  <div class="sec-label reveal">
    <div class="sec-icon" style="background:rgba(124,92,252,.15);">▶️</div>
    <div>
      <div class="sec-title">Video Playlist</div>
      <div class="sec-sub">Tonton langsung di sini – tidak bisa dibuka di YouTube</div>
    </div>
  </div>
  <div class="sec-divider"></div>

  <div class="protect-note reveal">
    🔒 <strong>Konten Dilindungi</strong> — Video hanya dapat diputar di halaman ini. Download via IDM maupun tool lain diblokir secara teknis.
  </div>

  <div class="video-player-box reveal">
    <!-- ── Video Area ─────────────────────────────────────── -->
    <div class="video-ratio">
      <!-- Shield: blocks all direct iframe interaction -->
      <!-- svelte-ignore a11y_no_static_element_interactions -->
      <div class="vid-shield"
        oncontextmenu={(e) => e.preventDefault()}
        ondragstart={(e) => e.preventDefault()}
        onmousedown={(e) => e.preventDefault()}>
      </div>

      <!-- YT IFrame API injects iframe here -->
      <div id="ytPlayerContainer"></div>

      <!-- Placeholder before first video -->
      {#if curIdx === -1}
        <div class="yt-placeholder">
          <div class="yt-placeholder-icon">▶</div>
          <p>Pilih video dari playlist di bawah</p>
        </div>
      {/if}
    </div>

    <!-- ── Custom Controls ────────────────────────────────── -->
    <div class="yt-controls" class:disabled={curIdx === -1}>

      <!-- Seekbar -->
      <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
      <div class="yt-seekbar" onclick={seekTo} role="slider" aria-label="Seekbar" tabindex="0" aria-valuenow={duration ? Math.round(currentTime/duration*100) : 0} aria-valuemin={0} aria-valuemax={100}>
        <div class="yt-seekbar-track">
          <div class="yt-seekbar-fill" style="width:{duration ? (currentTime/duration*100) : 0}%"></div>
          <div class="yt-seekbar-thumb" style="left:{duration ? (currentTime/duration*100) : 0}%"></div>
        </div>
      </div>

      <!-- Buttons Row -->
      <div class="yt-ctrl-row">
        <!-- Left -->
        <div class="yt-ctrl-left">
          <!-- svelte-ignore a11y_consider_explicit_label -->
          <button class="yt-btn" onclick={playPrev} disabled={curIdx <= 0} title="Video Sebelumnya">
            <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor"><path d="M6 6h2v12H6zm3.5 6 8.5 6V6z"/></svg>
          </button>
          <!-- svelte-ignore a11y_consider_explicit_label -->
          <button class="yt-btn yt-btn-play" onclick={togglePlay} title={isPlaying ? 'Pause' : 'Play'}>
            {#if isPlaying}
              <svg width="22" height="22" viewBox="0 0 24 24" fill="currentColor"><path d="M6 19h4V5H6v14zm8-14v14h4V5h-4z"/></svg>
            {:else}
              <svg width="22" height="22" viewBox="0 0 24 24" fill="currentColor"><path d="M8 5v14l11-7z"/></svg>
            {/if}
          </button>
          <!-- svelte-ignore a11y_consider_explicit_label -->
          <button class="yt-btn" onclick={stopVideo} title="Stop">
            <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor"><path d="M6 6h12v12H6z"/></svg>
          </button>
          <!-- svelte-ignore a11y_consider_explicit_label -->
          <button class="yt-btn" onclick={playNext} disabled={curIdx >= videos.length - 1} title="Video Berikutnya">
            <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor"><path d="M6 18l8.5-6L6 6v12zm2.5-6L12 9.5V14zM16 6h2v12h-2z"/></svg>
          </button>

          <!-- Volume -->
          <div class="yt-vol-wrap">
            <!-- svelte-ignore a11y_consider_explicit_label -->
            <button class="yt-btn" onclick={toggleMute} title={isMuted ? 'Unmute' : 'Mute'}>
              {#if isMuted || volume === 0}
                <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor"><path d="M16.5 12A4.5 4.5 0 0 0 14 7.97V10.18l2.45 2.45c.03-.2.05-.41.05-.63zm2.5 0c0 .94-.2 1.82-.54 2.64l1.51 1.51C20.63 14.91 21 13.5 21 12c0-4.28-2.99-7.86-7-8.77v2.06c2.89.86 5 3.54 5 6.71zM4.27 3 3 4.27 7.73 9H3v6h4l5 5v-6.73l4.25 4.25c-.67.52-1.42.93-2.25 1.18v2.06c1.38-.31 2.63-.95 3.69-1.81L19.73 21 21 19.73l-9-9L4.27 3zM12 4 9.91 6.09 12 8.18V4z"/></svg>
              {:else if volume < 50}
                <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor"><path d="M18.5 12A4.5 4.5 0 0 0 16 7.97v8.05c1.48-.73 2.5-2.25 2.5-4.02zM5 9v6h4l5 5V4L9 9H5z"/></svg>
              {:else}
                <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor"><path d="M3 9v6h4l5 5V4L7 9H3zm13.5 3A4.5 4.5 0 0 0 14 7.97v8.05c1.48-.73 2.5-2.25 2.5-4.02zM14 3.23v2.06c2.89.86 5 3.54 5 6.71s-2.11 5.85-5 6.71v2.06c4.01-.91 7-4.49 7-8.77s-2.99-7.86-7-8.77z"/></svg>
              {/if}
            </button>
            <input type="range" class="yt-vol-slider" min="0" max="100"
              value={isMuted ? 0 : volume} oninput={handleVolume} title="Volume">
          </div>

          <!-- Time -->
          <div class="yt-time">{fmt(currentTime)} / {fmt(duration)}</div>
        </div>

        <!-- Right -->
        <div class="yt-ctrl-right">
          <!-- Quality Selector -->
          {#if availableQualities.length > 0}
            <div class="yt-quality-wrap">
              <!-- svelte-ignore a11y_consider_explicit_label -->
              <button class="yt-btn yt-quality-btn" onclick={() => qualityOpen = !qualityOpen}
                title="Kualitas Video">
                <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor"><path d="M19.14 12.94c.04-.3.06-.61.06-.94s-.02-.64-.07-.94l2.03-1.58a.49.49 0 0 0 .12-.61l-1.92-3.32a.488.488 0 0 0-.59-.22l-2.39.96c-.5-.38-1.03-.7-1.62-.94l-.36-2.54a.484.484 0 0 0-.48-.41h-3.84c-.24 0-.43.17-.47.41l-.36 2.54c-.59.24-1.13.57-1.62.94l-2.39-.96c-.22-.08-.47 0-.59.22L2.74 8.87a.48.48 0 0 0 .12.61l2.03 1.58c-.05.3-.07.61-.07.94s.02.64.07.94l-2.03 1.58a.49.49 0 0 0-.12.61l1.92 3.32c.12.22.37.29.59.22l2.39-.96c.5.38 1.03.7 1.62.94l.36 2.54c.05.24.24.41.48.41h3.84c.24 0 .44-.17.47-.41l.36-2.54c.59-.24 1.13-.56 1.62-.94l2.39.96c.22.08.47 0 .59-.22l1.92-3.32a.49.49 0 0 0-.12-.61l-2.03-1.58zM12 15.6a3.6 3.6 0 1 1 0-7.2 3.6 3.6 0 0 1 0 7.2z"/></svg>
                <span class="yt-quality-label">{qualityLabel[currentQuality] ?? currentQuality}</span>
              </button>
              {#if qualityOpen}
                <!-- svelte-ignore a11y_no_static_element_interactions -->
                <div class="yt-quality-menu" onmouseleave={() => qualityOpen = false}>
                  {#each availableQualities as q}
                    <!-- svelte-ignore a11y_click_events_have_key_events -->
                    <div class="yt-quality-item" class:active={currentQuality === q}
                      onclick={() => setQuality(q)} role="menuitem" tabindex="0">
                      {qualityLabel[q] ?? q}
                    </div>
                  {/each}
                </div>
              {/if}
            </div>
          {/if}

          <!-- Fullscreen -->
          <!-- svelte-ignore a11y_consider_explicit_label -->
          <button class="yt-btn" onclick={toggleFullscreen} title={isFullscreen ? 'Keluar Fullscreen' : 'Fullscreen'}>
            {#if isFullscreen}
              <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor"><path d="M5 16h3v3h2v-5H5v2zm3-8H5v2h5V5H8v3zm6 11h2v-3h3v-2h-5v5zm2-11V5h-2v5h5V8h-3z"/></svg>
            {:else}
              <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor"><path d="M7 14H5v5h5v-2H7v-3zm-2-4h2V7h3V5H5v5zm12 7h-3v2h5v-5h-2v3zM14 5v2h3v3h2V5h-5z"/></svg>
            {/if}
          </button>
        </div>
      </div>
    </div>

    <!-- Video Meta -->
    <div class="video-meta">
      <div class="now-pill"><span class="blink"></span> Now Playing</div>
      <div class="video-title">{nowTitle}</div>
      <div class="video-desc">{nowDesc}</div>
    </div>
  </div>

  <div class="plist-box reveal mt-8">
    <div class="plist-head">
      <span class="plist-head-t">🎵 Daftar Video</span>
      <span class="count-pill">{videos.length} Video</span>
    </div>
    <div>
      {#each videos as v, i}
        <!-- svelte-ignore a11y_click_events_have_key_events -->
        <div class="pitem" class:active={curIdx === i} onclick={() => playVideo(i)} role="button" tabindex="0">
          <div class="p-num">{String(i+1).padStart(2,'0')}</div>
          <div class="p-thumb">
            <img src="https://img.youtube.com/vi/{v.id}/mqdefault.jpg" alt="" loading="lazy">
            <div class="p-ov">{curIdx === i ? '⏸' : '▶'}</div>
          </div>
          <div class="p-info">
            <div class="p-title">{v.title}</div>
          </div>
        </div>
      {/each}
    </div>
  </div>

  <!-- EBOOK -->
  <div class="sec-label reveal" style="margin-top:56px;">
    <div class="sec-icon" style="background:rgba(56,189,248,.1);">📚</div>
    <div>
      <div class="sec-title">E-Book Library</div>
      <div class="sec-sub">Baca langsung di halaman ini tanpa download</div>
    </div>
  </div>
  <div class="sec-divider"></div>

  <div class="ebook-card reveal">
    <div class="ebook-banner">
      <div class="ebook-cover-big">📖</div>
      <div class="ebook-banner-text">
        <div class="ebook-tag">Novel / Ebook</div>
        <div class="ebook-banner-title">Ebook Novel</div>
      </div>
    </div>
    <div class="ebook-actions">
      <div class="ebook-meta">Format: PDF · Baca langsung di halaman ini</div>
    </div>
    <div class="ebook-frame-wrap" style="display:block;">
      {#if ebookStarted}
        <PdfViewer pdfUrl="{import.meta.env.BASE_URL}sample-local-pdf.pdf" />
      {:else}
        <div class="ebook-start-wrap">
          <button class="ebook-start-btn" onclick={() => ebookStarted = true}>
            📖 Baca Ebook
          </button>
        </div>
      {/if}
    </div>
  </div>

  <!-- TOOLS -->
  <div class="sec-label reveal" style="margin-top:56px;">
    <div class="sec-icon" style="background:rgba(250,204,21,.1);">🛠️</div>
    <div>
      <div class="sec-title">Tools &amp; Download</div>
      <div class="sec-sub">Akses tools pilihan – tetap di dalam halaman ini</div>
    </div>
  </div>
  <div class="sec-divider"></div>

  <div class="tools-tabs reveal">
    {#each tools as t, i}
      <!-- svelte-ignore a11y_click_events_have_key_events -->
      <div class="tool-tab" class:active={activeTool === i} onclick={() => selectTool(i)} role="tab" tabindex="0">
        {t.icon} {t.name}
      </div>
    {/each}
  </div>

  <div class="tools-frame-box reveal">
    <div class="tools-bar">
      <div class="win-dots">
        <div class="win-dot" style="background:#FF5F57;"></div>
        <div class="win-dot" style="background:#FEBC2E;"></div>
        <div class="win-dot" style="background:#28C840;"></div>
      </div>
      <div class="tools-url">{toolUrl}</div>
    </div>
    <div>
      {#if !toolFrameVisible}
        <div class="tools-empty">
          <div class="big">🛠️</div>
          <p>Pilih tool dari tab di atas untuk mulai</p>
        </div>
      {:else}
        <iframe
          class="tools-iframe"
          src={toolSrc}
          sandbox="allow-scripts allow-same-origin allow-forms"
          referrerpolicy="no-referrer"
          title="Tool Frame"
        ></iframe>
      {/if}
    </div>
  </div>

</div><!-- /wrap -->

<footer>
  <p>© 2025 Member Area · Dibuat eksklusif untuk <span>Member</span></p>
  <p style="margin-top:6px;">Semua konten dilindungi. Dilarang menyebarkan tanpa izin.</p>
</footer>
