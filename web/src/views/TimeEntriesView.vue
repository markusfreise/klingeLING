<script setup lang="ts">
import { ref, onMounted, watch } from 'vue'
import api from '@/api/client'
import type { TimeEntry, Project, Task } from '@/types'
import { PencilIcon, TrashIcon, PlusIcon } from '@heroicons/vue/24/outline'
import ManualEntryModal from '@/components/ManualEntryModal.vue'

const entries = ref<TimeEntry[]>([])
const projects = ref<Project[]>([])
const tasks = ref<Task[]>([])
const loading = ref(true)
const showManualEntry = ref(false)
const meta = ref({ current_page: 1, last_page: 1, total: 0 })

// Filters
const dateFrom = ref(getLast30Days())
const dateTo = ref(new Date().toISOString().split('T')[0]!)
const filterProjectId = ref('')

function getLast30Days(): string {
  const d = new Date()
  d.setDate(d.getDate() - 30)
  return d.toISOString().split('T')[0]!
}

async function fetchEntries(page = 1) {
  loading.value = true
  try {
    const params: Record<string, string | number> = {
      'filter[date_from]': dateFrom.value,
      'filter[date_to]': dateTo.value,
      per_page: 50,
      page,
      sort: '-started_at',
    }
    if (filterProjectId.value) {
      params['filter[project_id]'] = filterProjectId.value
    }
    const { data } = await api.get('/time-entries', { params })
    entries.value = data.data
    meta.value = data.meta
  } finally {
    loading.value = false
  }
}

async function deleteEntry(id: string) {
  if (!confirm('Delete this time entry?')) return
  await api.delete(`/time-entries/${id}`)
  entries.value = entries.value.filter((e) => e.id !== id)
}

function formatTime(iso: string): string {
  return new Date(iso).toLocaleTimeString('de-DE', { hour: '2-digit', minute: '2-digit' })
}

function formatDate(iso: string): string {
  return new Date(iso).toLocaleDateString('de-DE', {
    weekday: 'short',
    day: '2-digit',
    month: '2-digit',
  })
}

function onEntryCreated() {
  showManualEntry.value = false
  fetchEntries()
}

watch([dateFrom, dateTo, filterProjectId], () => fetchEntries())

onMounted(async () => {
  const [, projectsRes, tasksRes] = await Promise.all([
    fetchEntries(),
    api.get('/projects', { params: { 'filter[is_active]': true, per_page: 100 } }),
    api.get('/tasks'),
  ])
  projects.value = projectsRes.data.data
  tasks.value = tasksRes.data.data
})
</script>

<template>
  <div class="page-container">
    <div class="page-header">
      <h1 class="heading-1">Time Entries</h1>
      <button class="btn-primary" @click="showManualEntry = true">
        <PlusIcon class="btn-icon-sm" />
        Manual Entry
      </button>
    </div>

    <!-- Filters -->
    <div class="filters-bar">
      <div class="form-group">
        <label class="form-label">From</label>
        <input v-model="dateFrom" type="date" class="form-input" />
      </div>
      <div class="form-group">
        <label class="form-label">To</label>
        <input v-model="dateTo" type="date" class="form-input" />
      </div>
      <div class="form-group">
        <label class="form-label">Project</label>
        <select v-model="filterProjectId" class="form-select">
          <option value="">All projects</option>
          <option v-for="p in projects" :key="p.id" :value="p.id">{{ p.name }}</option>
        </select>
      </div>
    </div>

    <!-- Table -->
    <div class="card">
      <div v-if="loading" class="card-body loading-container">
        <div class="loading-spinner"></div>
      </div>
      <div v-else-if="entries.length === 0" class="empty-state">
        <p class="empty-state-text">No time entries for this period.</p>
      </div>
      <div v-else class="table-container">
        <table class="table">
          <thead class="table-header">
            <tr>
              <th class="table-th">Date</th>
              <th class="table-th">Project</th>
              <th class="table-th">Task</th>
              <th class="table-th">Description</th>
              <th class="table-th">Start</th>
              <th class="table-th">Stop</th>
              <th class="table-th">Duration</th>
              <th class="table-th">Billable</th>
              <th class="table-th">Actions</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="entry in entries" :key="entry.id" class="table-row">
              <td class="table-td">{{ formatDate(entry.started_at) }}</td>
              <td class="table-td">
                <div class="project-cell">
                  <span class="color-dot" :style="{ backgroundColor: entry.project?.color }"></span>
                  {{ entry.project?.name }}
                </div>
              </td>
              <td class="table-td">{{ entry.task?.name || '—' }}</td>
              <td class="table-td table-td-desc">{{ entry.description || '—' }}</td>
              <td class="table-td table-td-mono">{{ formatTime(entry.started_at) }}</td>
              <td class="table-td table-td-mono">
                {{ entry.stopped_at ? formatTime(entry.stopped_at) : '—' }}
              </td>
              <td class="table-td table-td-mono">
                <span :class="entry.is_running ? 'timer-running' : ''">
                  {{ entry.duration_human }}
                </span>
              </td>
              <td class="table-td">
                <span :class="entry.is_billable ? 'badge-green' : 'badge-gray'">
                  {{ entry.is_billable ? 'Yes' : 'No' }}
                </span>
              </td>
              <td class="table-td">
                <div class="action-btns">
                  <button class="btn-ghost btn-icon btn-sm" @click="deleteEntry(entry.id)">
                    <TrashIcon class="action-icon" />
                  </button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Pagination -->
      <div v-if="meta.last_page > 1" class="pagination-bar">
        <span class="text-muted">{{ meta.total }} entries</span>
        <div class="pagination-btns">
          <button
            class="btn-secondary btn-sm"
            :disabled="meta.current_page <= 1"
            @click="fetchEntries(meta.current_page - 1)"
          >
            Previous
          </button>
          <span class="text-muted">{{ meta.current_page }} / {{ meta.last_page }}</span>
          <button
            class="btn-secondary btn-sm"
            :disabled="meta.current_page >= meta.last_page"
            @click="fetchEntries(meta.current_page + 1)"
          >
            Next
          </button>
        </div>
      </div>
    </div>

    <ManualEntryModal
      v-if="showManualEntry"
      :projects="projects"
      :tasks="tasks"
      @close="showManualEntry = false"
      @created="onEntryCreated"
    />
  </div>
</template>

<style scoped>
@reference "tailwindcss";
.page-header {
  @apply flex items-center justify-between mb-6;
}

.filters-bar {
  @apply grid grid-cols-1 sm:grid-cols-3 gap-4 mb-6;
}

.loading-container {
  @apply flex justify-center py-12;
}

.project-cell {
  @apply flex items-center gap-2;
}

.table-td-desc {
  @apply max-w-xs truncate;
}

.table-td-mono {
  @apply font-mono text-xs tabular-nums;
}

.action-btns {
  @apply flex gap-1;
}

.action-icon {
  @apply h-4 w-4;
}

.btn-icon-sm {
  @apply h-4 w-4;
}

.pagination-bar {
  @apply flex items-center justify-between px-6 py-3 border-t border-gray-200;
}

.pagination-btns {
  @apply flex items-center gap-3;
}
</style>
