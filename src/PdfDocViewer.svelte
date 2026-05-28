<script lang="ts">
  import type { DocumentState } from '@embedpdf/core';
  import { RenderLayer } from '@embedpdf/plugin-render/svelte';

  let {
    documentId,
    documentState,
  }: { documentId: string; documentState: DocumentState } = $props();

  let currentPageIndex = $state(0);

  const totalPages = $derived(documentState.document?.pageCount ?? 0);
  const currentPage = $derived(currentPageIndex + 1);
  const page = $derived(documentState.document?.pages[currentPageIndex] ?? null);

  function prev() {
    if (currentPageIndex > 0) currentPageIndex--;
  }

  function next() {
    if (currentPageIndex < totalPages - 1) currentPageIndex++;
  }

  let inputVal = $state('1');
  let inputFocused = $state(false);

  $effect(() => {
    if (!inputFocused) inputVal = String(currentPage);
  });

  function commitInput() {
    inputFocused = false;
    const n = parseInt(inputVal, 10);
    if (!isNaN(n) && n >= 1 && n <= totalPages) {
      currentPageIndex = n - 1;
    } else {
      inputVal = String(currentPage);
    }
  }
</script>

<div class="doc-wrap">
  <!-- Toolbar -->
  <div class="toolbar">
    <button class="nav-btn" disabled={currentPageIndex <= 0} onclick={prev}>
      <svg width="15" height="15" viewBox="0 0 16 16" fill="none">
        <path d="M10 12L6 8l4-4" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
      </svg>
      <span>Sebelumnya</span>
    </button>

    <div class="page-info">
      <input
        class="page-input"
        type="text"
        inputmode="numeric"
        value={inputVal}
        onfocus={() => { inputFocused = true; inputVal = String(currentPage); }}
        oninput={(e) => inputVal = (e.target as HTMLInputElement).value}
        onblur={commitInput}
        onkeydown={(e) => e.key === 'Enter' && (e.target as HTMLInputElement).blur()}
      />
      <span class="page-sep">/ {totalPages}</span>
    </div>

    <button class="nav-btn" disabled={currentPageIndex >= totalPages - 1} onclick={next}>
      <span>Berikutnya</span>
      <svg width="15" height="15" viewBox="0 0 16 16" fill="none">
        <path d="M6 4l4 4-4 4" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/>
      </svg>
    </button>
  </div>

  <!-- Page canvas area -->
  <div class="page-area">
    {#if page}
      {#key currentPageIndex}
        <div
          class="page-wrap"
          style="aspect-ratio:{page.size.width}/{page.size.height};"
        >
          <RenderLayer
            {documentId}
            pageIndex={currentPageIndex}
            scale={documentState.scale}
            style="width:100%;height:100%;display:block;"
          />
        </div>
      {/key}
    {/if}
  </div>

  <!-- Footer: page dots / mini progress -->
  <div class="footer">
    <div class="progress-track">
      <div
        class="progress-fill"
        style="width:{totalPages > 1 ? ((currentPage - 1) / (totalPages - 1)) * 100 : 100}%"
      ></div>
    </div>
    <span class="footer-label">Halaman {currentPage} dari {totalPages}</span>
  </div>
</div>

<style>
  .doc-wrap {
    display: flex;
    flex-direction: column;
    width: 100%;
    height: 100%;
    background: #11111a;
  }

  /* ── Toolbar ── */
  .toolbar {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 10px 16px;
    background: rgba(255,255,255,.035);
    border-bottom: 1px solid rgba(255,255,255,.07);
    user-select: none;
  }

  .nav-btn {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 7px 14px;
    border-radius: 8px;
    border: 1px solid rgba(124,92,252,.35);
    background: rgba(124,92,252,.1);
    color: rgba(255,255,255,.75);
    font-size: 13px;
    font-weight: 500;
    cursor: pointer;
    transition: background .15s, border-color .15s, color .15s, opacity .15s;
    white-space: nowrap;
    line-height: 1;
  }
  .nav-btn:hover:not(:disabled) {
    background: rgba(124,92,252,.26);
    border-color: rgba(124,92,252,.6);
    color: #fff;
  }
  .nav-btn:disabled { opacity: .28; cursor: not-allowed; }

  .page-info {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 13px;
  }
  .page-input {
    width: 42px;
    text-align: center;
    background: rgba(255,255,255,.07);
    border: 1px solid rgba(255,255,255,.12);
    border-radius: 6px;
    color: #fff;
    font-size: 13px;
    font-weight: 600;
    padding: 5px 4px;
    outline: none;
    transition: border-color .15s, background .15s;
  }
  .page-input:focus {
    border-color: rgba(124,92,252,.6);
    background: rgba(124,92,252,.1);
  }
  .page-sep { color: rgba(255,255,255,.4); font-weight: 500; }

  /* ── Page area ── */
  .page-area {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
    padding: 20px;
    background: #13131c;
  }

  .page-wrap {
    max-width: 100%;
    max-height: 100%;
    box-shadow: 0 8px 40px rgba(0,0,0,.7), 0 0 0 1px rgba(255,255,255,.05);
    border-radius: 3px;
    overflow: hidden;
    background: #fff;
  }

  /* ── Footer ── */
  .footer {
    flex-shrink: 0;
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 8px 16px;
    background: rgba(255,255,255,.025);
    border-top: 1px solid rgba(255,255,255,.06);
  }
  .progress-track {
    flex: 1;
    height: 3px;
    background: rgba(255,255,255,.08);
    border-radius: 99px;
    overflow: hidden;
  }
  .progress-fill {
    height: 100%;
    background: linear-gradient(90deg, #7c5cfc, #38bdf8);
    border-radius: 99px;
    transition: width .25s ease;
  }
  .footer-label {
    flex-shrink: 0;
    font-size: 11px;
    color: rgba(255,255,255,.35);
    white-space: nowrap;
  }
</style>
