<template>
  <main class="page">
    <section class="hero">
      <p class="eyebrow">OCI Object Storage + API Gateway</p>
      <h1>Static Vue app pronta para build e publicação pública</h1>
      <p class="lead">
        Esta base foi pensada para gerar um <strong>build estático</strong>, subir em bucket e
        validar o acesso via API Gateway e DNS da OCI.
      </p>
    </section>

    <section class="card">
      <h2>Status do deploy</h2>
      <div class="grid">
        <div>
          <span class="label">Build</span>
          <strong>vite build</strong>
        </div>
        <div>
          <span class="label">Endpoint API</span>
          <strong>{{ apiBaseUrl }}</strong>
        </div>
      </div>
    </section>

    <section class="card">
      <h2>Teste do API Gateway</h2>
      <p class="muted">
        O botão abaixo chama <code>/health</code> no mesmo deployment do site.
      </p>
      <div class="actions">
        <button :disabled="loading" @click="testGateway">
          {{ loading ? 'Testando...' : 'Testar gateway' }}
        </button>
        <button class="secondary" @click="reset">Limpar</button>
      </div>
      <pre class="result">{{ output }}</pre>
    </section>

    <section class="card">
      <h2>Arquitetura sugerida</h2>
      <ol>
        <li>Build da SPA com Vite.</li>
        <li>Upload do conteúdo de <code>dist/</code> no bucket.</li>
        <li>API Gateway apontando para o objeto estático e/ou backend de teste.</li>
        <li>DNS público apontando para o domínio do gateway.</li>
      </ol>
    </section>
  </main>
</template>

<script setup lang="ts">
import { computed, ref } from 'vue';

const configuredApiBaseUrl = import.meta.env.VITE_API_BASE_URL?.replace(/\/$/, '') ?? '';
const apiBaseUrl = configuredApiBaseUrl || getCurrentDeploymentBaseUrl();
const loading = ref(false);
const output = ref('Aguardando teste...');

const defaultHealthUrl = computed(() => `${apiBaseUrl}/health`);

function getCurrentDeploymentBaseUrl() {
  const path = window.location.pathname;

  if (path.endsWith('/')) {
    return `${window.location.origin}${path.replace(/\/$/, '')}`;
  }

  if (path.includes('.')) {
    return `${window.location.origin}${path.slice(0, path.lastIndexOf('/'))}`;
  }

  return `${window.location.origin}${path}`;
}

async function testGateway() {
  loading.value = true;
  output.value = `Chamando ${defaultHealthUrl.value}...`;

  try {
    const response = await fetch(defaultHealthUrl.value, {
      headers: { Accept: 'application/json' },
    });

    const contentType = response.headers.get('content-type') ?? '';
    const body = contentType.includes('application/json')
      ? JSON.stringify(await response.json(), null, 2)
      : await response.text();

    output.value = [
      `HTTP ${response.status} ${response.statusText}`,
      body,
    ].join('\n\n');
  } catch (error) {
    output.value = `Falha ao chamar o gateway: ${error instanceof Error ? error.message : String(error)}`;
  } finally {
    loading.value = false;
  }
}

function reset() {
  output.value = 'Aguardando teste...';
}
</script>
