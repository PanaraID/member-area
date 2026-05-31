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

  let curIdx = $state(-1);
  let nowTitle = $state('Pilih video dari playlist di bawah ↓');
  let nowDesc = $state('Klik salah satu video untuk mulai menonton.');
  let playerSrc = $state('about:blank');
  let toolUrl = $state('— Pilih tool di atas —');
  let toolSrc = $state('about:blank');
  let toolFrameVisible = $state(false);
  let activeTool = $state(-1);

  function playVideo(idx: number) {
    curIdx = idx;
    const v = videos[idx];
    const origin = encodeURIComponent(window.location.origin);
    playerSrc = `https://www.youtube-nocookie.com/embed/${v.id}?autoplay=1&rel=0&modestbranding=1&iv_load_policy=3&playsinline=1&origin=${origin}`;
    nowTitle = v.title;
    nowDesc = v.desc;
    setTimeout(() => {
      document.querySelector('.video-player-box')?.scrollIntoView({behavior:'smooth',block:'start'});
    }, 50);
  }

  function selectTool(idx: number) {
    activeTool = idx;
    toolUrl = tools[idx].url;
    toolSrc = tools[idx].url;
    toolFrameVisible = true;
    setTimeout(() => {
      document.querySelector('.tools-frame-box')?.scrollIntoView({behavior:'smooth',block:'start'});
    }, 50);
  }

  onMount(() => {
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
        if (e.isIntersecting) {
          e.target.classList.add('visible');
          obs.unobserve(e.target);
        }
      });
    }, {threshold: 0.06});
    document.querySelectorAll('.reveal').forEach(el => obs.observe(el));

    return () => obs.disconnect();
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
    <div class="video-ratio">
      <!-- svelte-ignore a11y_no_static_element_interactions -->
      <div class="vid-shield"
        oncontextmenu={(e) => e.preventDefault()}
        ondragstart={(e) => e.preventDefault()}
        onmousedown={(e) => e.preventDefault()}>
      </div>
      <iframe id="ytPlayer" src={playerSrc} allow="autoplay; encrypted-media" title="Video Player"></iframe>
    </div>
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
        <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
        <div class="pitem" class:active={curIdx === i} onclick={() => playVideo(i)}>
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
      <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_static_element_interactions -->
      <div class="tool-tab" class:active={activeTool === i} onclick={() => selectTool(i)}>
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
