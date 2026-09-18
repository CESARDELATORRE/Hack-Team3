<script setup>
import { computed, onMounted, ref } from 'vue'

const endpoint = import.meta.env.VITE_FOUNDRY_ENDPOINT
const apiKey = import.meta.env.VITE_FOUNDRY_API_KEY
const apiVersion = import.meta.env.VITE_FOUNDRY_API_VERSION || '2025-05-15-preview'

const messages = ref([
  {
    role: 'assistant',
    text: 'Hola, soy tu asistente gastronómico de Iberostar. ¿Qué te gustaría comer o beber hoy?',
  },
])
const input = ref('')
const loading = ref(false)
const error = ref('')
const sessionId = ref(`session-${Date.now()}`)

const canSend = computed(() => input.value.trim().length > 0 && !loading.value)

function formatSystemPrompt() {
  return [
    'Eres un asistente gastronómico experto para Iberostar.',
    'Responde en español o inglés según el mensaje del usuario.',
    'Usa solo información del catálogo disponible.',
    'Si faltan datos críticos, informa la limitación y sugiere hablar con el personal.',
    'Nunca inventes ingredientes, alérgenos, precios o disponibilidad.',
    'Si hay dudas, pide aclaración antes de confirmar una recomendación.',
    'En el POC, registra solo pedidos de prueba y no hagas cobros ni integraciones reales.',
  ].join(' ')
}

async function callFoundry(messageText) {
  if (!endpoint || !apiKey) {
    throw new Error('Falta la configuración del endpoint o la API key de Azure Foundry.')
  }

  const payload = {
    input: [
      {
        type: 'message',
        role: 'system',
        content: formatSystemPrompt(),
      },
      {
        type: 'message',
        role: 'user',
        content: messageText,
      },
    ],
  }

  const requestUrl = new URL(endpoint)
  requestUrl.searchParams.set('api-version', apiVersion)

  const response = await fetch(requestUrl, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'api-key': apiKey,
      Accept: 'application/json',
    },
    body: JSON.stringify(payload),
  })

  if (!response.ok) {
    const body = await response.text()
    throw new Error(`Azure Foundry error (${response.status}): ${body || response.statusText}`)
  }

  const data = await response.json()

  const content =
    data?.output?.[0]?.content?.[0]?.text ||
    data?.output_text ||
    data?.choices?.[0]?.message?.content ||
    'No pude obtener una respuesta útil del asistente.'

  return typeof content === 'string' ? content : JSON.stringify(content)
}

async function sendMessage() {
  const text = input.value.trim()
  if (!text) return

  error.value = ''
  loading.value = true

  messages.value.push({ role: 'user', text })
  input.value = ''

  try {
    const reply = await callFoundry(text)
    messages.value.push({ role: 'assistant', text: reply })
  } catch (err) {
    messages.value.push({
      role: 'assistant',
      text: 'No he podido contactar con el asistente. Revisa la configuración de Azure Foundry o inténtalo más tarde.',
    })
    error.value = err.message
  } finally {
    loading.value = false
  }
}

onMounted(() => {
  if (!endpoint || !apiKey) {
    error.value = 'Configuración de Azure Foundry no encontrada. Revisa el archivo .env.'
  }
})
</script>

<template>
  <main class="app-shell">
    <section class="chat-card">
      <header class="header">
        <div>
          <p class="eyebrow">Iberostar Selection</p>
          <h1>Asistente gastronómico</h1>
        </div>
        <span class="badge">POC</span>
      </header>

      <div class="status-row" v-if="sessionId">
        <span>Sesión</span>
        <strong>{{ sessionId }}</strong>
      </div>

      <div class="messages" aria-live="polite">
        <article
          v-for="(message, index) in messages"
          :key="`${message.role}-${index}`"
          :class="['message', message.role]"
        >
          <span class="author">{{ message.role === 'assistant' ? 'Asistente' : 'Tú' }}</span>
          <p>{{ message.text }}</p>
        </article>
      </div>

      <div v-if="error" class="error-box">{{ error }}</div>

      <form class="composer" @submit.prevent="sendMessage">
        <textarea
          v-model="input"
          rows="3"
          placeholder="Pregunta por un plato, un vino o un maridaje..."
          :disabled="loading"
        ></textarea>
        <button type="submit" :disabled="!canSend">
          {{ loading ? 'Pensando...' : 'Enviar' }}
        </button>
      </form>
    </section>
  </main>
</template>

<style scoped>
.app-shell {
  min-height: 100vh;
  display: grid;
  place-items: center;
  background: linear-gradient(135deg, #f4efe7 0%, #ebe6d7 100%);
  padding: 24px;
}

.chat-card {
  width: min(100%, 760px);
  background: rgba(255, 255, 255, 0.96);
  border-radius: 20px;
  box-shadow: 0 20px 50px rgba(18, 29, 41, 0.12);
  overflow: hidden;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 24px;
  background: #123a59;
  color: white;
}

.eyebrow {
  margin: 0 0 6px;
  font-size: 12px;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  opacity: 0.72;
}

h1 {
  margin: 0;
  font-size: clamp(1.4rem, 3vw, 2rem);
}

.badge {
  background: rgba(255, 255, 255, 0.12);
  padding: 8px 10px;
  border-radius: 999px;
  font-weight: 700;
}

.status-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 24px;
  background: #f5f5f5;
  border-bottom: 1px solid #e8e8e8;
  font-size: 0.85rem;
}

.messages {
  display: flex;
  flex-direction: column;
  gap: 14px;
  padding: 20px 24px 12px;
  max-height: 60vh;
  overflow-y: auto;
  background: #fafaf8;
}

.message {
  max-width: 82%;
  border-radius: 16px;
  padding: 12px 14px;
  line-height: 1.45;
}

.message.user {
  align-self: flex-end;
  background: #183f5e;
  color: white;
}

.message.assistant {
  align-self: flex-start;
  background: #eef4f6;
  color: #12283b;
}

.author {
  display: block;
  font-size: 0.74rem;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  opacity: 0.75;
  margin-bottom: 4px;
}

.message p {
  margin: 0;
  white-space: pre-wrap;
}

.error-box {
  margin: 0 24px 10px;
  padding: 10px 12px;
  border-radius: 12px;
  background: #fdecea;
  color: #a8322d;
  border: 1px solid #f2c9c7;
}

.composer {
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 12px 24px 24px;
  background: white;
}

textarea {
  width: 100%;
  resize: vertical;
  border: 1px solid #d7dfe4;
  border-radius: 12px;
  padding: 14px 16px;
  font: inherit;
  min-height: 80px;
  box-sizing: border-box;
}

button {
  border: none;
  border-radius: 12px;
  padding: 12px 18px;
  background: #d7b56d;
  color: #1c1c1c;
  font-weight: 700;
  cursor: pointer;
}

button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
</style>
