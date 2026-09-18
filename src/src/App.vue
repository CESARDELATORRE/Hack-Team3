<script setup>
import { computed, onMounted, ref, watch } from 'vue'

const endpoint = import.meta.env.VITE_FOUNDRY_ENDPOINT
const apiKey = import.meta.env.VITE_FOUNDRY_API_KEY
const apiVersion = import.meta.env.VITE_FOUNDRY_API_VERSION || '2025-05-15-preview'

const locale = ref('es')
const catalog = [
  {
    id: 'paella-mariscos',
    name: 'Paella de mariscos',
    type: 'plato',
    category: 'marisco',
    price: 24,
    dietary: ['marisco'],
    tags: ['mariscos', 'paella', 'maridaje', 'principal'],
    description: 'Arroz con marisco, azafrán y verduras tostadas.',
    available: true,
    featured: true,
  },
  {
    id: 'ensalada-iberica',
    name: 'Ensalada Ibérica',
    type: 'plato',
    category: 'vegetariano',
    price: 16,
    dietary: ['vegetariano', 'sin gluten'],
    tags: ['vegetariana', 'fresca', 'ensalada', 'ligera'],
    description: 'Mix de hojas, tomate, aguacate y queso artesanal.',
    available: true,
    featured: true,
  },
  {
    id: 'solomillo-iberostar',
    name: 'Solomillo Iberostar',
    type: 'plato',
    category: 'carne',
    price: 29,
    dietary: ['sin gluten'],
    tags: ['carne', 'principal', 'premium', 'fuertes'],
    description: 'Solomillo con reducción de vino y patatas al romero.',
    available: true,
    featured: true,
  },
  {
    id: 'cava-brut',
    name: 'Cava Brut',
    type: 'vino',
    category: 'cava',
    price: 12,
    dietary: ['sin gluten'],
    tags: ['vino', 'cava', 'brut', 'aperitivo'],
    description: 'Burbuja fina, ideal para aperitivos y maridajes ligeros.',
    available: true,
    featured: false,
  },
  {
    id: 'vino-rosado',
    name: 'Rosado de la casa',
    type: 'vino',
    category: 'rosado',
    price: 15,
    dietary: ['sin gluten'],
    tags: ['vino', 'rosado', 'maridaje', 'verano'],
    description: 'Vino de cuerpo medio para platos de pescado y ensaladas.',
    available: true,
    featured: false,
  },
  {
    id: 'postre-turron',
    name: 'Turrón de chocolate',
    type: 'postre',
    category: 'dulce',
    price: 10,
    dietary: ['vegano', 'sin gluten'],
    tags: ['postre', 'dulce', 'chocolate', 'final'],
    description: 'Postre elegante con textura cremosa y toque salado.',
    available: true,
    featured: false,
  },
]

const browserVoice = ref(null)
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
const selectedItems = ref([])
const orderHistory = ref([])
const recommendations = ref(catalog.slice(0, 3))

const canSend = computed(() => input.value.trim().length > 0 && !loading.value)
const orderTotal = computed(() =>
  selectedItems.value.reduce((sum, item) => sum + Number(item.price || 0), 0),
)
const voiceSupported = computed(() => !!browserVoice.value)

function normalizeText(value) {
  return (value || '')
    .toLowerCase()
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '')
}

function localeText(valueEs, valueEn) {
  return locale.value === 'en' ? valueEn : valueEs
}

function getRecommendationMatches(query = '') {
  const normalizedQuery = normalizeText(query)

  const scored = catalog
    .map((item) => {
      let score = 0
      const haystack = normalizeText(`${item.name} ${item.description} ${item.category} ${item.tags.join(' ')} ${item.dietary.join(' ')}`)

      if (!normalizedQuery) {
        score = item.featured ? 10 : 2
      } else {
        if (haystack.includes(normalizedQuery)) score += 7
        if (item.name.toLowerCase().includes(normalizedQuery)) score += 5
        if (normalizedQuery.includes('vegetariano') && item.dietary.some((entry) => entry.includes('vegetariano'))) score += 4
        if (normalizedQuery.includes('barato') && item.price <= 16) score += 4
        if (normalizedQuery.includes('premium') && item.price >= 24) score += 4
        if (normalizedQuery.includes('vino') && item.type === 'vino') score += 4
        if (normalizedQuery.includes('maridaje') && item.tags.some((tag) => tag.includes('maridaje'))) score += 4
      }

      return { item, score }
    })
    .filter((entry) => entry.score > 0)
    .sort((a, b) => b.score - a.score)
    .map((entry) => entry.item)

  return scored.slice(0, 4)
}

function buildLocalResponse(messageText) {
  const matches = getRecommendationMatches(messageText)
  const intro = locale.value === 'en'
    ? 'I found options for you:'
    : 'He encontrado opciones para ti:'

  if (!matches.length) {
    return locale.value === 'en'
      ? 'I do not have a clear match in the current catalog. I can suggest a light option, a premium recommendation or a pairing based on what you need.'
      : 'No tengo una coincidencia clara en el catálogo actual. Te puedo sugerir una opción ligera, una recomendación premium o un maridaje según lo que busques.'
  }

  const lines = matches
    .map((item) => `• ${item.name} (${item.price}€): ${item.description}`)
    .join('\n')

  return `${intro}\n${lines}\n${locale.value === 'en' ? 'If you want, I can narrow it down by budget, diet or pairing.' : 'Si quieres, te puedo afinar más por presupuesto, dieta o maridaje.'}`
}

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
    return buildLocalResponse(messageText)
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

  try {
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
      buildLocalResponse(messageText)

    return typeof content === 'string' ? content : JSON.stringify(content)
  } catch (err) {
    console.warn('Falling back to local catalog recommendation:', err)
    return buildLocalResponse(messageText)
  }
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
    recommendations.value = getRecommendationMatches(text)
  } catch (err) {
    messages.value.push({
      role: 'assistant',
      text: localeText(
        'No he podido contactar con el asistente. Reviso el catálogo local para sugerirte opciones disponibles.',
        'I could not reach the assistant. I am checking the local catalog to suggest available options.',
      ),
    })
    recommendations.value = getRecommendationMatches(text)
    error.value = err.message
  } finally {
    loading.value = false
  }
}

function addItem(item) {
  if (!selectedItems.value.some((selected) => selected.id === item.id)) {
    selectedItems.value.push(item)
  }
}

function removeItem(itemId) {
  selectedItems.value = selectedItems.value.filter((item) => item.id !== itemId)
}

function confirmOrder() {
  if (!selectedItems.value.length) {
    error.value = localeText(
      'Selecciona al menos un plato o vino antes de confirmar el pedido de prueba.',
      'Select at least one dish or wine before confirming the test order.',
    )
    return
  }

  const orderId = `ORD-${Math.floor(Date.now() / 1000)}`
  const order = {
    id: orderId,
    status: 'created',
    createdAt: new Date().toISOString(),
    items: selectedItems.value.map((item) => ({
      id: item.id,
      name: item.name,
      price: item.price,
    })),
    total: orderTotal.value,
  }

  orderHistory.value.unshift(order)
  messages.value.push({
    role: 'assistant',
    text: localeText(
      `Pedido de prueba registrado con ID ${orderId}. Estado: creado. Puedes revisarlo en el backoffice de demo.`,
      `Test order registered with ID ${orderId}. Status: created. You can review it in the demo backoffice.`,
    ),
  })
  selectedItems.value = []
  error.value = ''
}

function toggleLocale(nextLocale) {
  locale.value = nextLocale
  const recognition = browserVoice.value
  if (recognition) {
    recognition.lang = nextLocale === 'en' ? 'en-US' : 'es-ES'
  }
}

function startVoiceInput() {
  if (!browserVoice.value) {
    error.value = localeText(
      'La voz no está disponible en este navegador.',
      'Voice input is not available in this browser.',
    )
    return
  }

  browserVoice.value.start()
}

onMounted(() => {
  const storageKey = 'iberostar-order-history'
  const storedHistory = localStorage.getItem(storageKey)
  if (storedHistory) {
    orderHistory.value = JSON.parse(storedHistory)
  }

  const storedItems = localStorage.getItem('iberostar-selected-items')
  if (storedItems) {
    selectedItems.value = JSON.parse(storedItems)
  }

  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition
  if (SpeechRecognition) {
    const recognition = new SpeechRecognition()
    recognition.lang = locale.value === 'en' ? 'en-US' : 'es-ES'
    recognition.interimResults = false
    recognition.continuous = false

    recognition.onresult = (event) => {
      const transcript = Array.from(event.results)
        .map((result) => result[0]?.transcript || '')
        .join(' ')
        .trim()

      if (transcript) {
        input.value = transcript
      }
    }

    recognition.onerror = () => {
      error.value = localeText(
        'No se pudo reconocer la voz. Prueba con texto.',
        'Voice recognition did not work. Try typing instead.',
      )
    }

    browserVoice.value = recognition
  }

  recommendations.value = getRecommendationMatches('')
  if (!endpoint || !apiKey) {
    error.value = localeText(
      'Configuración de Azure Foundry no encontrada; la app usa el catálogo local como fallback funcional.',
      'Azure Foundry configuration not found; the app uses the local catalog as a functional fallback.',
    )
  }
})

watch(
  selectedItems,
  (items) => {
    localStorage.setItem('iberostar-selected-items', JSON.stringify(items))
  },
  { deep: true },
)

watch(
  orderHistory,
  (items) => {
    localStorage.setItem('iberostar-order-history', JSON.stringify(items))
  },
  { deep: true },
)
</script>

<template>
  <main class="app-shell">
    <section class="experience">
      <div class="chat-panel">
        <header class="header">
          <div>
            <p class="eyebrow">Iberostar Selection</p>
            <h1>{{ localeText('Asistente gastronómico', 'Restaurant assistant') }}</h1>
          </div>
          <div class="header-actions">
            <div class="language-switcher" aria-label="Language selector">
              <button type="button" :class="{ active: locale === 'es' }" @click="toggleLocale('es')">ES</button>
              <button type="button" :class="{ active: locale === 'en' }" @click="toggleLocale('en')">EN</button>
            </div>
            <span class="badge">POC</span>
          </div>
        </header>

        <div class="status-row" v-if="sessionId">
          <span>{{ localeText('Sesión', 'Session') }}</span>
          <strong>{{ sessionId }}</strong>
        </div>

        <div class="messages" aria-live="polite">
          <article
            v-for="(message, index) in messages"
            :key="`${message.role}-${index}`"
            :class="['message', message.role]"
          >
            <span class="author">{{ message.role === 'assistant' ? localeText('Asistente', 'Assistant') : localeText('Tú', 'You') }}</span>
            <p>{{ message.text }}</p>
          </article>
        </div>

        <div v-if="error" class="error-box">{{ error }}</div>

        <form class="composer" @submit.prevent="sendMessage">
          <div class="composer-row">
            <textarea
              v-model="input"
              rows="3"
              :placeholder="localeText('Pregunta por un plato, un vino o un maridaje...', 'Ask for a dish, wine or pairing...')"
              :disabled="loading"
            ></textarea>
            <button v-if="voiceSupported" type="button" class="voice-button" @click="startVoiceInput" :disabled="loading">
              {{ localeText('Micrófono', 'Voice') }}
            </button>
          </div>
          <button type="submit" :disabled="!canSend">
            {{ loading ? localeText('Pensando...', 'Thinking...') : localeText('Enviar', 'Send') }}
          </button>
        </form>
      </div>

      <aside class="side-panel">
        <section class="panel-block">
          <h2>{{ localeText('Recomendaciones', 'Recommendations') }}</h2>
          <div class="recommendation-list">
            <article v-for="item in recommendations" :key="item.id" class="recommendation-card">
              <div class="title-row">
                <h3>{{ item.name }}</h3>
                <span>{{ item.price }}€</span>
              </div>
              <p>{{ item.description }}</p>
              <div class="meta-row">
                <span>{{ item.type }}</span>
                <span>{{ item.category }}</span>
              </div>
              <button class="secondary" type="button" @click="addItem(item)">{{ localeText('Añadir', 'Add') }}</button>
            </article>
          </div>
        </section>

        <section class="panel-block">
          <h2>{{ localeText('Pedido de prueba', 'Test order') }}</h2>
          <div v-if="selectedItems.length === 0" class="empty-state">{{ localeText('Aún no tienes elementos seleccionados.', 'No items selected yet.') }}</div>
          <ul v-else class="order-list">
            <li v-for="item in selectedItems" :key="item.id">
              <div>
                <strong>{{ item.name }}</strong>
                <small>{{ item.price }}€</small>
              </div>
              <button type="button" class="remove-button" @click="removeItem(item.id)">{{ localeText('Quitar', 'Remove') }}</button>
            </li>
          </ul>

          <div class="summary-row" v-if="selectedItems.length">
            <strong>{{ localeText('Total estimado:', 'Estimated total:') }}</strong>
            <span>{{ orderTotal }}€</span>
          </div>

          <button type="button" class="confirm-button" :disabled="selectedItems.length === 0" @click="confirmOrder">
            {{ localeText('Confirmar pedido', 'Confirm order') }}
          </button>
        </section>

        <section class="panel-block">
          <h2>{{ localeText('Backoffice de demo', 'Demo backoffice') }}</h2>
          <div v-if="orderHistory.length === 0" class="empty-state">{{ localeText('Sin pedidos confirmados todavía.', 'No confirmed orders yet.') }}</div>
          <ul v-else class="history-list">
            <li v-for="order in orderHistory.slice(0, 3)" :key="order.id">
              <strong>{{ order.id }}</strong>
              <span>{{ order.status }}</span>
              <small>{{ order.items.length }} {{ localeText('artículos', 'items') }} · {{ order.total }}€</small>
            </li>
          </ul>
        </section>
      </aside>
    </section>
  </main>
</template>

<style scoped>
.app-shell {
  min-height: 100vh;
  background: linear-gradient(135deg, #f4efe7 0%, #ebe6d7 100%);
  padding: 24px;
}

.experience {
  max-width: 1400px;
  margin: 0 auto;
  display: grid;
  grid-template-columns: 1.6fr 1fr;
  gap: 24px;
}

.chat-panel,
.panel-block {
  background: rgba(255, 255, 255, 0.96);
  border-radius: 20px;
  box-shadow: 0 20px 50px rgba(18, 29, 41, 0.1);
}

.chat-panel {
  overflow: hidden;
}

.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
  padding: 20px 24px;
  background: #123a59;
  color: white;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 12px;
}

.language-switcher {
  display: inline-flex;
  background: rgba(255, 255, 255, 0.12);
  border-radius: 999px;
  padding: 4px;
}

.language-switcher button {
  background: transparent;
  color: white;
  border: none;
  border-radius: 999px;
  padding: 6px 10px;
  font-weight: 700;
}

.language-switcher .active {
  background: rgba(255, 255, 255, 0.2);
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
  font-size: clamp(1.5rem, 2.5vw, 2.3rem);
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

.composer-row {
  display: flex;
  gap: 12px;
  align-items: flex-start;
}

textarea {
  flex: 1;
  width: 100%;
  resize: vertical;
  border: 1px solid #d7dfe4;
  border-radius: 12px;
  padding: 14px 16px;
  font: inherit;
  min-height: 80px;
  box-sizing: border-box;
}

.voice-button {
  min-width: 110px;
  background: #e8f1f9;
  color: #123a59;
}

button {
  border: none;
  border-radius: 12px;
  padding: 10px 14px;
  background: #d7b56d;
  color: #1c1c1c;
  font-weight: 700;
  cursor: pointer;
}

button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.side-panel {
  display: grid;
  gap: 20px;
}

.panel-block {
  padding: 18px 18px 20px;
}

.panel-block h2 {
  margin: 0 0 16px;
  font-size: 1.2rem;
  color: #183f5e;
}

.recommendation-list,
.order-list,
.history-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: grid;
  gap: 12px;
}

.recommendation-card {
  border: 1px solid #e5e7eb;
  border-radius: 14px;
  padding: 12px;
  background: #fbfaf7;
}

.title-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
}

.title-row h3 {
  margin: 0;
  font-size: 1rem;
}

.title-row span {
  font-weight: 700;
  color: #123a59;
}

.recommendation-card p {
  margin: 8px 0;
  color: #3f4a58;
  line-height: 1.5;
}

.meta-row {
  display: flex;
  justify-content: space-between;
  gap: 8px;
  font-size: 0.72rem;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  color: #667085;
  margin-bottom: 10px;
}

.secondary,
.confirm-button,
.remove-button {
  width: 100%;
  margin-top: 4px;
}

.order-list li,
.history-list li {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  border: 1px solid #ebedf0;
  border-radius: 12px;
  padding: 10px 12px;
  background: #f9fafb;
}

.order-list li div,
.history-list li {
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 16px;
  font-size: 1rem;
  color: #123a59;
}

.empty-state {
  color: #667085;
  border: 1px dashed #d7dfe4;
  background: #f9fafb;
  border-radius: 12px;
  padding: 12px;
}

.remove-button {
  width: auto;
  background: #f7efe3;
  color: #4f3d18;
}

.confirm-button {
  margin-top: 12px;
}

@media (max-width: 980px) {
  .experience {
    grid-template-columns: 1fr;
  }

  .composer-row {
    flex-direction: column;
  }
}
</style>
