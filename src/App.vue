<template>
  <div id="root">
    <div v-if="!isLoggedIn" class="login-shell">
      <div class="login-title">Admin</div>
      <div class="login-subtitle">Enter admin password to access users and messages.</div>

      <form @submit.prevent="onLogin">
        <div class="field">
          <label class="field-label">Admin password</label>
          <input
            v-model="password"
            type="password"
            class="input"
            autocomplete="current-password"
            required
          />
        </div>
        <button class="button" type="submit" :disabled="loggingIn || !password">
          <span v-if="loggingIn">Checking…</span>
          <span v-else>Log in</span>
        </button>
        <div v-if="loginError" class="error">{{ loginError }}</div>
      </form>

      <div class="login-footer">
        <span>Backend: <strong>{{ healthLabel }}</strong></span>
        <span class="pill">Base: eus.lat</span>
      </div>
    </div>

    <div v-else class="shell">
      <header class="shell-header">
        <div class="shell-title">Admin</div>
        <div style="display:flex;align-items:center;gap:12px;">
          <div class="badge">
            <span
              class="dot"
              :class="healthOk ? 'green' : 'red'"
            ></span>
            <span>{{ healthLabel }}</span>
          </div>
          <button class="button secondary" type="button" @click="onLogout">
            Logout
          </button>
        </div>
      </header>

      <div class="shell-main">
        <aside class="panel panel-left">
          <div class="field">
            <div class="field-label">User filters</div>
          </div>
          <div class="field">
            <label class="field-label">user_id</label>
            <input
              v-model="filters.user_id"
              class="input"
              inputmode="numeric"
              placeholder="e.g. 123"
            />
          </div>
          <div class="field">
            <label class="field-label">telegram_id</label>
            <input
              v-model="filters.telegram_id"
              class="input"
              inputmode="numeric"
              placeholder="e.g. 123456"
            />
          </div>
          <div class="field">
            <label class="field-label">limit</label>
            <input
              v-model.number="filters.limit"
              class="input"
              type="number"
              min="1"
              max="500"
            />
          </div>
          <button
            class="button"
            type="button"
            :disabled="usersLoading"
            @click="loadUsers"
          >
            <span v-if="usersLoading">Loading users…</span>
            <span v-else>Load users</span>
          </button>
          <div v-if="usersError" class="error">{{ usersError }}</div>
          <div style="font-size:11px;color:#6b7280;margin-top:auto;">
            {{ users.length }} of {{ usersTotal }} users
          </div>
        </aside>

        <main class="panel panel-right">
          <div style="display:flex;gap:16px;height:100%;">
            <div style="flex:1;min-width:0;">
              <div class="field-label" style="margin-bottom:6px;">Users</div>
              <div style="max-height:calc(100vh - 170px);overflow-y:auto;">
                <table class="table">
                  <thead>
                    <tr>
                      <th>id</th>
                      <th>telegram</th>
                      <th>username</th>
                      <th>name</th>
                      <th>locale</th>
                      <th>created</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr
                      v-for="u in users"
                      :key="u.id"
                      :class="[
                        'row-clickable',
                        selectedUserId === u.id ? 'row-active' : ''
                      ]"
                      @click="selectUser(u)"
                    >
                      <td>{{ u.id }}</td>
                      <td>{{ u.telegram_id }}</td>
                      <td>{{ u.telegram_username || '—' }}</td>
                      <td>{{ u.display_name || '—' }}</td>
                      <td>{{ u.locale }}</td>
                      <td>{{ u.created_at }}</td>
                    </tr>
                    <tr v-if="!usersLoading && !users.length">
                      <td colspan="6" style="font-size:12px;color:#6b7280;">
                        No users loaded yet.
                      </td>
                    </tr>
                  </tbody>
                </table>
              </div>
            </div>

            <div style="flex:1.2;min-width:0;display:flex;flex-direction:column;">
              <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:6px;">
                <div class="field-label">
                  Messages
                  <span v-if="selectedUserId">for user {{ selectedUserId }}</span>
                </div>
                <div v-if="messagesLoading" style="font-size:11px;color:#9ca3af;">
                  Loading…
                </div>
              </div>
              <div v-if="messagesError" class="error" style="margin-bottom:6px;">
                {{ messagesError }}
              </div>
              <div v-if="!selectedUserId" style="font-size:13px;color:#6b7280;margin-top:8px;">
                Select a user to see message history.
              </div>
              <div v-else class="messages">
                <div
                  v-for="m in messages"
                  :key="m.id"
                  class="message"
                  :class="m.role"
                >
                  <div class="message-meta">
                    <span>{{ m.role }}</span>
                    <span>{{ m.created_at }}</span>
                  </div>
                  <div class="message-content">
                    {{ m.content }}
                  </div>
                </div>
                <div v-if="!messagesLoading && !messages.length" style="font-size:12px;color:#6b7280;">
                  No messages for this user.
                </div>
              </div>
            </div>
          </div>
        </main>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted } from 'vue'

const BASE_URL = 'https://eus.lat'

const storedPassword = localStorage.getItem('adminPassword') || ''
const storedLoggedIn = localStorage.getItem('isLoggedIn') === '1'

const password = ref(storedPassword)
const isLoggedIn = ref(!!storedPassword && storedLoggedIn)
const loggingIn = ref(false)
const loginError = ref('')

const healthStatus = ref('unknown')

const filters = reactive({
  user_id: '',
  telegram_id: '',
  limit: 100
})

const users = ref([])
const usersTotal = ref(0)
const usersLoading = ref(false)
const usersError = ref('')

const selectedUserId = ref(null)
const messages = ref([])
const messagesLoading = ref(false)
const messagesError = ref('')

const healthOk = computed(() => healthStatus.value === 'ok')
const healthLabel = computed(() => {
  if (healthStatus.value === 'ok') return 'Online'
  if (healthStatus.value === 'down') return 'Offline'
  return 'Unknown'
})

function currentPassword() {
  return localStorage.getItem('adminPassword') || ''
}

async function fetchWithAuth(path, init) {
  const pw = currentPassword()
  if (!pw) throw new Error('missing password')
  const headers = {
    'Content-Type': 'application/json',
    ...(init && init.headers),
    'X-Admin-Password': pw
  }
  const res = await fetch(BASE_URL + path, { ...(init || {}), headers })
  if (res.status === 401) {
    onLogout()
    throw new Error('unauthorized')
  }
  return res
}

async function pingHealth() {
  try {
    const res = await fetch(BASE_URL + '/_health')
    if (res.ok) {
      healthStatus.value = 'ok'
    } else {
      healthStatus.value = 'down'
    }
  } catch {
    healthStatus.value = 'down'
  }
}

async function onLogin() {
  if (!password.value) return
  loggingIn.value = true
  loginError.value = ''
  try {
    const res = await fetch(BASE_URL + '/admin/api/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ password: password.value })
    })
    if (res.status === 401) {
      loginError.value = 'Invalid password'
      loggingIn.value = false
      return
    }
    if (!res.ok) {
      loginError.value = 'Login failed'
      loggingIn.value = false
      return
    }
    const data = await res.json()
    if (!data.ok) {
      loginError.value = 'Invalid password'
      loggingIn.value = false
      return
    }
    localStorage.setItem('adminPassword', password.value)
    localStorage.setItem('isLoggedIn', '1')
    isLoggedIn.value = true
    users.value = []
    messages.value = []
    selectedUserId.value = null
    usersError.value = ''
    messagesError.value = ''
  } catch {
    loginError.value = 'Network error'
  } finally {
    loggingIn.value = false
  }
}

function onLogout() {
  localStorage.removeItem('adminPassword')
  localStorage.removeItem('isLoggedIn')
  isLoggedIn.value = false
  password.value = ''
  users.value = []
  messages.value = []
  selectedUserId.value = null
}

async function loadUsers() {
  if (!isLoggedIn.value) return
  usersLoading.value = true
  usersError.value = ''
  try {
    const params = new URLSearchParams()
    if (filters.user_id) params.set('user_id', String(filters.user_id))
    if (filters.telegram_id) params.set('telegram_id', String(filters.telegram_id))
    if (filters.limit) params.set('limit', String(filters.limit))
    const qs = params.toString()
    const res = await fetchWithAuth('/admin/api/users' + (qs ? `?${qs}` : ''), {
      method: 'GET'
    })
    if (!res.ok) {
      usersError.value = 'Failed to load users'
      return
    }
    const data = await res.json()
    users.value = data.items || []
    usersTotal.value = data.total || 0
  } catch (e) {
    if (e.message !== 'unauthorized') usersError.value = 'Network error'
  } finally {
    usersLoading.value = false
  }
}

async function loadMessages(userId) {
  if (!isLoggedIn.value || !userId) return
  messagesLoading.value = true
  messagesError.value = ''
  try {
    const params = new URLSearchParams()
    params.set('user_id', String(userId))
    params.set('limit', '100')
    const res = await fetchWithAuth('/admin/api/messages?' + params.toString(), {
      method: 'GET'
    })
    if (!res.ok) {
      messagesError.value = 'Failed to load messages'
      return
    }
    const data = await res.json()
    const items = data.items || []
    messages.value = items.slice().reverse()
  } catch (e) {
    if (e.message !== 'unauthorized') messagesError.value = 'Network error'
  } finally {
    messagesLoading.value = false
  }
}

function selectUser(u) {
  if (!u || !u.id) return
  selectedUserId.value = u.id
  loadMessages(u.id)
}

onMounted(() => {
  pingHealth()
  if (isLoggedIn.value) {
    loadUsers()
  }
  setInterval(pingHealth, 30000)
})
</script>


