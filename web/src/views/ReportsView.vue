<script setup lang="ts">
import { ref, onMounted, watch } from 'vue'
import api from '@/api/client'
import { useAuthStore } from '@/stores/auth'
import type { SummaryRow, ReportTotals, BudgetRow, UtilizationRow, Project } from '@/types'
import { ArrowDownTrayIcon } from '@heroicons/vue/24/outline'

const auth = useAuthStore()

type ReportType = 'summary' | 'budget' | 'utilization'
const reportType = ref<ReportType>('summary')
const groupBy = ref('project')
const dateFrom = ref(getFirstOfMonth())
const dateTo = ref(new Date().toISOString().split('T')[0]!)
const filterProjectId = ref('')

const summaryData = ref<SummaryRow[]>([])
const totals = ref<ReportTotals | null>(null)
const budgetData = ref<BudgetRow[]>([])
const utilizationData = ref<UtilizationRow[]>([])
const projects = ref<Project[]>([])
const loading = ref(false)

function fmt(d: Date): string {
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`
}

function getFirstOfMonth(): string {
  const d = new Date()
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-01`
}

type DateRange = 'this_week' | 'this_month' | 'last_month' | 'this_quarter' | 'this_year'
const activeRange = ref<DateRange | null>('this_month')
const dateRangeOptions: { key: DateRange; label: string }[] = [
  { key: 'this_week', label: 'This week' },
  { key: 'this_month', label: 'This month' },
  { key: 'last_month', label: 'Last month' },
  { key: 'this_quarter', label: 'This quarter' },
  { key: 'this_year', label: 'This year' },
]

function setRange(range: DateRange) {
  activeRange.value = range
  const today = new Date()
  const y = today.getFullYear()
  const m = today.getMonth()

  if (range === 'this_week') {
    const dow = today.getDay() // 0=Sun
    const monday = new Date(today)
    monday.setDate(today.getDate() - ((dow + 6) % 7))
    dateFrom.value = fmt(monday)
    dateTo.value = fmt(today)
  } else if (range === 'this_month') {
    dateFrom.value = `${y}-${String(m + 1).padStart(2, '0')}-01`
    dateTo.value = fmt(today)
  } else if (range === 'last_month') {
    const first = new Date(y, m - 1, 1)
    const last = new Date(y, m, 0)
    dateFrom.value = fmt(first)
    dateTo.value = fmt(last)
  } else if (range === 'this_quarter') {
    const q = Math.floor(m / 3)
    dateFrom.value = fmt(new Date(y, q * 3, 1))
    dateTo.value = fmt(today)
  } else if (range === 'this_year') {
    dateFrom.value = `${y}-01-01`
    dateTo.value = fmt(today)
  }
}

async function fetchReport() {
  loading.value = true
  try {
    if (reportType.value === 'summary') {
      const params: Record<string, string> = {
        date_from: dateFrom.value,
        date_to: dateTo.value,
        group_by: groupBy.value,
      }
      if (filterProjectId.value) params['filter[project_id]'] = filterProjectId.value
      const { data } = await api.get('/reports/summary', { params })
      summaryData.value = data.data
      totals.value = data.meta?.totals
    } else if (reportType.value === 'budget') {
      const { data } = await api.get('/reports/budget')
      budgetData.value = data.data
    } else if (reportType.value === 'utilization') {
      const { data } = await api.get('/reports/utilization', {
        params: { date_from: dateFrom.value, date_to: dateTo.value },
      })
      utilizationData.value = data.data
    }
  } finally {
    loading.value = false
  }
}

async function exportCsv() {
  const params = new URLSearchParams({
    date_from: dateFrom.value,
    date_to: dateTo.value,
    format: 'csv',
  })
  if (filterProjectId.value) params.set('filter[project_id]', filterProjectId.value)
  window.open(`/api/reports/export?${params.toString()}`, '_blank')
}

watch([reportType, groupBy, dateFrom, dateTo, filterProjectId], fetchReport)

function onDateInput() {
  activeRange.value = null
}

onMounted(async () => {
  const { data } = await api.get('/projects', { params: { per_page: 100 } })
  projects.value = data.data
  fetchReport()
})
</script>

<template>
  <div class="page-container">
    <div class="page-header">
      <h1 class="heading-1">Reports</h1>
      <button class="btn-secondary" @click="exportCsv">
        <ArrowDownTrayIcon class="btn-icon-sm" />
        Export CSV
      </button>
    </div>

    <!-- Report type tabs -->
    <div class="report-tabs">
      <button
        :class="reportType === 'summary' ? 'report-tab-active' : 'report-tab'"
        @click="reportType = 'summary'"
      >
        Summary
      </button>
      <button
        :class="reportType === 'budget' ? 'report-tab-active' : 'report-tab'"
        @click="reportType = 'budget'"
      >
        Budget
      </button>
      <button
        v-if="auth.isAdmin"
        :class="reportType === 'utilization' ? 'report-tab-active' : 'report-tab'"
        @click="reportType = 'utilization'"
      >
        Utilization
      </button>
    </div>

    <!-- Filters -->
    <div v-if="reportType !== 'budget'" class="filters-section">
      <div class="date-ranges">
        <button
          v-for="r in dateRangeOptions"
          :key="r.key"
          :class="activeRange === r.key ? 'range-btn-active' : 'range-btn'"
          @click="setRange(r.key)"
        >
          {{ r.label }}
        </button>
      </div>
      <div class="filters-bar">
      <div class="form-group">
        <label class="form-label">From</label>
        <input v-model="dateFrom" type="date" class="form-input" @change="onDateInput" />
      </div>
      <div class="form-group">
        <label class="form-label">To</label>
        <input v-model="dateTo" type="date" class="form-input" @change="onDateInput" />
      </div>
      <div v-if="reportType === 'summary'" class="form-group">
        <label class="form-label">Group by</label>
        <select v-model="groupBy" class="form-select">
          <option value="project">Project</option>
          <option value="client">Client</option>
          <option value="user">User</option>
          <option value="day">Day</option>
          <option value="week">Week</option>
          <option value="month">Month</option>
        </select>
      </div>
      <div v-if="reportType === 'summary'" class="form-group">
        <label class="form-label">Project</label>
        <select v-model="filterProjectId" class="form-select">
          <option value="">All</option>
          <option v-for="p in projects" :key="p.id" :value="p.id">{{ p.name }}</option>
        </select>
      </div>
      </div>
    </div>

    <!-- Totals -->
    <div v-if="totals && reportType === 'summary'" class="totals-bar">
      <div class="total-item">
        <span class="total-label">Total</span>
        <span class="total-value">{{ totals.total_hours }}h</span>
      </div>
      <div class="total-item">
        <span class="total-label">Billable</span>
        <span class="total-value total-value-green">{{ totals.billable_hours }}h</span>
      </div>
      <div class="total-item">
        <span class="total-label">Non-billable</span>
        <span class="total-value">{{ totals.non_billable_hours }}h</span>
      </div>
      <div class="total-item">
        <span class="total-label">Entries</span>
        <span class="total-value">{{ totals.entry_count }}</span>
      </div>
    </div>

    <!-- Summary Table -->
    <div v-if="reportType === 'summary'" class="card">
      <div v-if="loading" class="loading-center"><div class="loading-spinner"></div></div>
      <div v-else-if="summaryData.length === 0" class="empty-state">
        <p class="empty-state-text">No data for this period.</p>
      </div>
      <div v-else class="table-container">
        <table class="table">
          <thead class="table-header">
            <tr>
              <th class="table-th">
                {{ groupBy === 'project' ? 'Project' : groupBy === 'client' ? 'Client' : groupBy === 'user' ? 'User' : 'Period' }}
              </th>
              <th class="table-th">Total Hours</th>
              <th class="table-th">Billable Hours</th>
              <th class="table-th">Entries</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(row, i) in summaryData" :key="i" class="table-row">
              <td class="table-td">
                <div class="report-name-cell">
                  <span v-if="row.color" class="color-dot" :style="{ backgroundColor: row.color }"></span>
                  {{ row.project_name || row.client_name || row.user_name || row.period }}
                </div>
              </td>
              <td class="table-td table-td-mono">{{ row.total_hours }}h</td>
              <td class="table-td table-td-mono">{{ row.billable_hours }}h</td>
              <td class="table-td table-td-mono">{{ row.entry_count }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Budget Table -->
    <div v-if="reportType === 'budget'" class="card">
      <div v-if="loading" class="loading-center"><div class="loading-spinner"></div></div>
      <div v-else-if="budgetData.length === 0" class="empty-state">
        <p class="empty-state-text">No projects with budgets.</p>
      </div>
      <div v-else class="table-container">
        <table class="table">
          <thead class="table-header">
            <tr>
              <th class="table-th">Project</th>
              <th class="table-th">Client</th>
              <th class="table-th">Budget</th>
              <th class="table-th">Tracked</th>
              <th class="table-th">Remaining</th>
              <th class="table-th">Progress</th>
              <th class="table-th">Status</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="row in budgetData" :key="row.id" class="table-row">
              <td class="table-td">
                <div class="report-name-cell">
                  <span class="color-dot" :style="{ backgroundColor: row.color }"></span>
                  {{ row.project_name }}
                </div>
              </td>
              <td class="table-td">{{ row.client_name }}</td>
              <td class="table-td table-td-mono">{{ row.budget_hours }}h</td>
              <td class="table-td table-td-mono">{{ row.tracked_hours }}h</td>
              <td class="table-td table-td-mono">{{ row.remaining_hours }}h</td>
              <td class="table-td">
                <div class="budget-bar-inline">
                  <div class="budget-bar-bg-sm">
                    <div
                      class="budget-bar-fill-sm"
                      :class="{
                        'budget-bar-ok': row.status === 'on_track',
                        'budget-bar-warn': row.status === 'at_risk',
                        'budget-bar-over': row.status === 'over_budget',
                      }"
                      :style="{ width: Math.min(row.budget_used_percentage, 100) + '%' }"
                    ></div>
                  </div>
                  <span class="budget-bar-pct">{{ row.budget_used_percentage }}%</span>
                </div>
              </td>
              <td class="table-td">
                <span
                  :class="{
                    'badge-green': row.status === 'on_track',
                    'badge-yellow': row.status === 'at_risk',
                    'badge-red': row.status === 'over_budget',
                  }"
                >
                  {{ row.status.replace('_', ' ') }}
                </span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Utilization Table -->
    <div v-if="reportType === 'utilization'" class="card">
      <div v-if="loading" class="loading-center"><div class="loading-spinner"></div></div>
      <div v-else class="table-container">
        <table class="table">
          <thead class="table-header">
            <tr>
              <th class="table-th">Team Member</th>
              <th class="table-th">Total Hours</th>
              <th class="table-th">Billable</th>
              <th class="table-th">Non-billable</th>
              <th class="table-th">Billable %</th>
              <th class="table-th">Days Tracked</th>
              <th class="table-th">Avg h/day</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="row in utilizationData" :key="row.id" class="table-row">
              <td class="table-td font-medium">{{ row.name }}</td>
              <td class="table-td table-td-mono">{{ row.total_hours }}h</td>
              <td class="table-td table-td-mono">{{ row.billable_hours }}h</td>
              <td class="table-td table-td-mono">{{ row.non_billable_hours }}h</td>
              <td class="table-td table-td-mono">{{ row.billable_percentage }}%</td>
              <td class="table-td table-td-mono">{{ row.days_tracked }}</td>
              <td class="table-td table-td-mono">{{ row.avg_hours_per_day }}h</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<style scoped>
@reference "tailwindcss";
.page-header {
  @apply flex items-center justify-between mb-6;
}

.report-tabs {
  @apply flex gap-1 mb-6 bg-gray-100 rounded-lg p-1 w-fit;
}

.report-tab {
  @apply px-4 py-2 rounded-md text-sm font-medium text-gray-600 transition-colors hover:text-gray-900;
}

.report-tab-active {
  @apply px-4 py-2 rounded-md text-sm font-medium bg-white text-gray-900 shadow-sm;
}

.filters-section {
  @apply mb-6 space-y-3;
}

.date-ranges {
  @apply flex flex-wrap gap-2;
}

.range-btn {
  @apply px-3 py-1.5 rounded-md text-sm font-medium text-gray-600 border border-gray-200 bg-white hover:bg-gray-50 transition-colors;
}

.range-btn-active {
  @apply px-3 py-1.5 rounded-md text-sm font-medium text-blue-700 border border-blue-300 bg-blue-50;
}

.filters-bar {
  @apply grid grid-cols-2 sm:grid-cols-4 gap-4;
}

.totals-bar {
  @apply grid grid-cols-2 sm:grid-cols-4 gap-4 mb-6;
}

.total-item {
  @apply bg-white rounded-lg border border-gray-200 shadow-sm p-3 flex flex-col;
}

.total-label {
  @apply text-xs text-gray-500;
}

.total-value {
  @apply text-lg font-bold text-gray-900 tabular-nums;
}

.total-value-green {
  @apply text-green-600;
}

.loading-center {
  @apply flex justify-center py-12;
}

.report-name-cell {
  @apply flex items-center gap-2;
}

.table-td-mono {
  @apply font-mono text-sm tabular-nums;
}

.budget-bar-inline {
  @apply flex items-center gap-2;
}

.budget-bar-bg-sm {
  @apply w-20 h-2 rounded-full bg-gray-200 overflow-hidden;
}

.budget-bar-fill-sm {
  @apply h-full rounded-full transition-all;
}

.budget-bar-ok {
  @apply bg-green-500;
}

.budget-bar-warn {
  @apply bg-yellow-500;
}

.budget-bar-over {
  @apply bg-red-500;
}

.budget-bar-pct {
  @apply text-xs text-gray-500 tabular-nums;
}

.btn-icon-sm {
  @apply h-4 w-4;
}
</style>
