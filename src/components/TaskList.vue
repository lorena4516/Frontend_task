<template>
  <div class="row g-4">
    <!-- Crear nueva tarea -->
    <div class="col-12">
      <div class="card shadow-sm">
        <div class="card-body">
          <h2 class="h5 mb-3">Nueva tarea</h2>

          <div class="row gy-2">
            <div class="col-md-6">
              <input
                v-model.trim="newTitle"
                type="text"
                class="form-control"
                placeholder="Título de la tarea"
                :disabled="submitting"
              />
            </div>

            <div class="col-md-4">
              <select
                multiple
                class="form-select"
                v-model="selectedKeywords"
                :disabled="submitting"
              >
                <option v-for="k in keywords" :key="k.id" :value="k.id">
                  {{ k.name }}
                </option>
              </select>
              <div class="form-text">Mantén Ctrl para selección múltiple.</div>
            </div>

            <div class="col-md-2 d-grid">
              <button
                class="btn btn-primary"
                @click="createTask"
                :disabled="!newTitle || submitting"
              >
                {{ submitting ? 'Guardando...' : 'Crear' }}
              </button>
            </div>
          </div>

          <p v-if="formError" class="text-danger mt-2">{{ formError }}</p>
          <p v-if="successMsg" class="text-success mt-2">{{ successMsg }}</p>
        </div>
      </div>
    </div>

    <!-- Listado -->
    <div class="col-12">
      <div class="card shadow-sm">
        <div class="card-body">
          <div class="d-flex justify-content-between align-items-center mb-3">
            <h2 class="h5 mb-0">Tareas</h2>
            <span class="text-muted">
              Total: {{ tasks.length }} · Completadas: {{ completedCount }} · Pendientes: {{ pendingCount }}
            </span>
          </div>

          <div v-if="loading" class="py-3">Cargando...</div>
          <div v-else>
            <div v-if="tasks.length === 0" class="text-muted">No hay tareas aún.</div>

            <ul class="list-group">
              <li
                v-for="t in tasks"
                :key="t.id"
                class="list-group-item d-flex flex-column flex-md-row align-items-md-center justify-content-between gap-2"
              >
                <div class="me-3">
                  <div class="fw-semibold" :class="t.is_done ? 'text-decoration-line-through text-muted' : ''">
                    {{ t.title }}
                  </div>
                  <div class="mt-1">
                    <span v-for="k in t.keywords" :key="k.id" class="badge bg-secondary me-1">
                      {{ k.name }}
                    </span>
                  </div>
                </div>

                <div class="d-flex align-items-center gap-2">
                  <span class="badge" :class="t.is_done ? 'bg-success' : 'bg-warning text-dark'">
                    {{ t.is_done ? 'Completada' : 'Pendiente' }}
                  </span>
                  <button
                    class="btn btn-outline-primary btn-sm"
                    @click="toggleTask(t)"
                    :disabled="t._toggling"
                  >
                    {{ t.is_done ? 'Marcar pendiente' : 'Marcar completada' }}
                  </button>
                </div>
              </li>
            </ul>

            <p v-if="listError" class="text-danger mt-3">{{ listError }}</p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
  import { onMounted, ref, computed } from 'vue'
  import { api } from '../api'

  const tasks = ref([])
  const keywords = ref([])
  const loading = ref(false)
  const listError = ref('')

  const newTitle = ref('')
  const selectedKeywords = ref([])
  const submitting = ref(false)
  const formError = ref('')
  const successMsg = ref('')

  const completedCount = computed(() => tasks.value.filter(t => t.is_done).length)
  const pendingCount = computed(() => tasks.value.filter(t => !t.is_done).length)

  async function fetchAll() {
    loading.value = true
    listError.value = ''
    try {
      const [tRes, kRes] = await Promise.all([
        api.get('/tasks'),
        api.get('/keywords')
      ])
      tasks.value = tRes.data ?? []
      keywords.value = kRes.data ?? []
    } catch (e) {
      listError.value = 'No se pudo cargar la lista. Revisa el backend.'
    } finally {
      loading.value = false
    }
  }

  async function createTask() {
    submitting.value = true
    formError.value = ''
    successMsg.value = ''
    try {
      const payload = { title: newTitle.value, keywords: selectedKeywords.value }
      const { data } = await api.post('/tasks', payload)
      tasks.value.unshift(data)
      newTitle.value = ''
      selectedKeywords.value = []
      successMsg.value = 'Tarea creada correctamente.'
    } catch (e) {
      formError.value = e?.response?.data?.message || 'Error al crear la tarea.'
    } finally {
      submitting.value = false
      setTimeout(() => (successMsg.value = ''), 1500)
    }
  }

  async function toggleTask(t) {
    t._toggling = true
    try {
      const { data } = await api.patch(`/tasks/${t.id}/toggle`)
      // actualiza el item en memoria
      const idx = tasks.value.findIndex(x => x.id === t.id)
      if (idx >= 0) tasks.value[idx] = data
    } catch (e) {
      listError.value = 'No se pudo cambiar el estado.'
    } finally {
      t._toggling = false
    }
  }

  onMounted(fetchAll)
</script>
