<script lang="ts">
  import { onMount } from 'svelte';
  import type { PdfEngine } from '@embedpdf/models';
  import type { PluginRegistry } from '@embedpdf/core';
  import { EmbedPDF } from '@embedpdf/core/svelte';
  import { createPdfiumDirectEngine, DEFAULT_PDFIUM_WASM_URL } from '@embedpdf/engines';
  import { DocumentManagerPluginPackage } from '@embedpdf/plugin-document-manager/svelte';
  import type { DocumentManagerPlugin } from '@embedpdf/plugin-document-manager';
  import { RenderPluginPackage } from '@embedpdf/plugin-render/svelte';
  import { DocumentContent } from '@embedpdf/plugin-document-manager/svelte';
  import PdfDocViewer from './PdfDocViewer.svelte';

  let { pdfUrl = `${import.meta.env.BASE_URL}sample-local-pdf.pdf` }: { pdfUrl?: string } = $props();

  let engine: PdfEngine | null = $state(null);
  let engineError = $state('');
  let pdfBuffer: ArrayBuffer | null = null;
  let fetchError = $state('');

  function cookieKey(url: string) {
    return 'pdf_page_' + url.replace(/[^a-z0-9]/gi, '_');
  }

  function getSavedPage(url: string): number {
    const key = cookieKey(url);
    const match = document.cookie.split(';').map(c => c.trim()).find(c => c.startsWith(key + '='));
    if (!match) return 0;
    const val = parseInt(match.split('=')[1], 10);
    return isNaN(val) ? 0 : Math.max(0, val);
  }

  const initialPage = $derived(getSavedPage(pdfUrl));

  const plugins = [
    { package: DocumentManagerPluginPackage },
    { package: RenderPluginPackage },
  ];

  async function onInitialized(registry: PluginRegistry) {
    const docManager = registry.getPlugin<DocumentManagerPlugin>('document-manager');
    if (!docManager || !pdfBuffer) return;
    docManager.provides().openDocumentBuffer({ buffer: pdfBuffer, name: 'ebook.pdf', autoActivate: true });
  }

  onMount(async () => {
    try {
      const [eng, res] = await Promise.all([
        createPdfiumDirectEngine(DEFAULT_PDFIUM_WASM_URL),
        fetch(pdfUrl),
      ]);

      if (!res.ok) {
        fetchError = `Fetch gagal: ${res.status} ${res.statusText}`;
        return;
      }

      pdfBuffer = await res.arrayBuffer();
      engine = eng;
    } catch (e: any) {
      engineError = e?.message ?? 'Gagal menginisialisasi PDF viewer';
    }
  });
</script>

<div class="pdf-viewer-wrap">
  {#if engineError || fetchError}
    <div class="pdf-error">⚠️ {engineError || fetchError}</div>
  {:else if !engine}
    <div class="pdf-loading">
      <div class="pdf-spinner"></div>
      <span>Memuat PDF engine…</span>
    </div>
  {:else}
    <EmbedPDF {engine} {plugins} {onInitialized}>
      {#snippet children({ activeDocumentId, documentStates })}
        {#if activeDocumentId}
          <DocumentContent documentId={activeDocumentId}>
            {#snippet children({ isLoaded, isLoading, isError, documentState })}
              {#if isLoading}
                <div class="pdf-loading">
                  <div class="pdf-spinner"></div>
                  <span>Memuat dokumen…</span>
                </div>
              {:else if isError}
                <div class="pdf-error">
                  ⚠️ {documentState.error ?? 'Gagal memuat dokumen PDF.'}
                </div>
              {:else if isLoaded}
                <PdfDocViewer documentId={activeDocumentId} {documentState} {initialPage} pdfKey={pdfUrl} />
              {/if}
            {/snippet}
          </DocumentContent>
        {:else if documentStates.length === 0}
          <div class="pdf-loading">
            <div class="pdf-spinner"></div>
            <span>Memuat dokumen…</span>
          </div>
        {/if}
      {/snippet}
    </EmbedPDF>
  {/if}
</div>

<style>
  .pdf-viewer-wrap {
    width: 100%;
    height: 680px;
    background: #0d0d12;
    border-radius: 12px;
    overflow: hidden;
    display: flex;
    flex-direction: column;
  }

  .pdf-loading {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 14px;
    color: rgba(255,255,255,.5);
    font-size: 14px;
  }

  .pdf-spinner {
    width: 36px;
    height: 36px;
    border: 3px solid rgba(124,92,252,.2);
    border-top-color: #7c5cfc;
    border-radius: 50%;
    animation: spin .8s linear infinite;
  }

  @keyframes spin {
    to { transform: rotate(360deg); }
  }

  .pdf-error {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    color: rgba(248,113,113,.8);
    font-size: 14px;
    padding: 24px;
    text-align: center;
  }
</style>
