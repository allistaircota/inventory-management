<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Controls card -->
      <div class="card controls-card">
        <!-- Budget slider row -->
        <div class="slider-row">
          <label class="slider-label">
            {{ t('restocking.budgetLabel') }}:
            <strong>{{ currencySymbol }}{{ budget.toLocaleString() }}</strong>
          </label>
          <input type="range" v-model.number="budget" :min="0" :max="maxBudget" :step="100" class="slider" />
          <div class="slider-range">
            <span>{{ currencySymbol }}0</span>
            <span>{{ currencySymbol }}{{ maxBudget.toLocaleString() }}</span>
          </div>
        </div>

        <!-- Lead time slider row -->
        <div class="slider-row">
          <label class="slider-label">
            {{ t('restocking.leadTimeLabel') }}:
            <strong>{{ leadTimeDays }} {{ t('restocking.days') }}</strong>
          </label>
          <input type="range" v-model.number="leadTimeDays" :min="7" :max="30" :step="1" class="slider" />
          <div class="slider-range">
            <span>7 {{ t('restocking.days') }}</span>
            <span>30 {{ t('restocking.days') }}</span>
          </div>
        </div>

        <!-- Summary + Place Order button -->
        <div class="order-summary">
          <div class="summary-stat">
            <span class="summary-stat-label">{{ t('restocking.itemsSelected') }}</span>
            <span class="summary-stat-value">{{ recommendedItems.length }}</span>
          </div>
          <div class="summary-stat">
            <span class="summary-stat-label">{{ t('restocking.totalCost') }}</span>
            <span class="summary-stat-value">{{ currencySymbol }}{{ formatCurrency(totalSelectedCost) }}</span>
          </div>
          <button
            class="btn-primary"
            @click="placeOrder"
            :disabled="submitting || recommendedItems.length === 0"
          >
            {{ submitting ? t('common.loading') : t('restocking.placeOrder') }}
          </button>
        </div>
      </div>

      <!-- Alerts -->
      <div v-if="orderSuccess" class="alert alert-success">{{ t('restocking.orderSuccess') }}</div>
      <div v-if="orderError" class="alert alert-danger">{{ orderError }}</div>

      <!-- Low stock items table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.lowStockItems') }} ({{ lowStockItems.length }})</h3>
        </div>
        <div v-if="lowStockItems.length === 0" class="no-data">
          {{ t('restocking.noLowStock') }}
        </div>
        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.name') }}</th>
                <th>{{ t('restocking.table.category') }}</th>
                <th class="col-num">{{ t('restocking.table.onHand') }}</th>
                <th class="col-num">{{ t('restocking.table.reorderPoint') }}</th>
                <th class="col-num">{{ t('restocking.table.restockQty') }}</th>
                <th class="col-num">{{ t('restocking.table.unitCost') }}</th>
                <th class="col-num">{{ t('restocking.table.restockCost') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.status') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in lowStockItems"
                :key="item.id"
                :class="isSelected(item) ? 'row-selected' : 'row-excluded'"
              >
                <td><code class="sku-code">{{ item.sku }}</code></td>
                <td>{{ item.name }}</td>
                <td>{{ item.category }}</td>
                <td class="col-num">{{ item.quantity_on_hand }}</td>
                <td class="col-num">{{ item.reorder_point }}</td>
                <td class="col-num"><strong>{{ item.restockQty }}</strong></td>
                <td class="col-num">{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
                <td class="col-num"><strong>{{ currencySymbol }}{{ formatCurrency(item.restockCost) }}</strong></td>
                <td>
                  <span :class="['badge', 'trend-' + item.trend]">{{ item.trend }}</span>
                </td>
                <td>
                  <span v-if="isSelected(item)" class="badge info">{{ t('restocking.withinBudget') }}</span>
                  <span v-else class="badge muted">{{ t('restocking.overBudget') }}</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const inventoryItems = ref([])
    const demandForecasts = ref([])
    const budget = ref(0)
    const leadTimeDays = ref(14)
    const submitting = ref(false)
    const orderSuccess = ref(false)
    const orderError = ref(null)

    // Items below reorder point, enriched and sorted
    const lowStockItems = computed(() => {
      const TREND_PRIORITY = { increasing: 0, stable: 1, decreasing: 2 }
      return inventoryItems.value
        .filter(item => item.quantity_on_hand < item.reorder_point)
        .map(item => {
          const restockQty = item.reorder_point - item.quantity_on_hand
          const restockCost = restockQty * item.unit_cost
          const shortfallPct = restockQty / item.reorder_point
          const forecast = demandForecasts.value.find(f => f.item_sku === item.sku)
          const trend = forecast ? forecast.trend : 'stable'
          return { ...item, restockQty, restockCost, shortfallPct, trend }
        })
        .sort((a, b) => {
          const trendDiff = (TREND_PRIORITY[a.trend] ?? 1) - (TREND_PRIORITY[b.trend] ?? 1)
          if (trendDiff !== 0) return trendDiff
          return b.shortfallPct - a.shortfallPct
        })
    })

    // Total cost to restock all low-stock items (slider max)
    const maxBudget = computed(() =>
      lowStockItems.value.reduce((sum, item) => sum + item.restockCost, 0)
    )

    // Greedy selection: walk sorted list, include while budget allows
    const recommendedItems = computed(() => {
      let remaining = budget.value
      return lowStockItems.value.filter(item => {
        if (item.restockCost <= remaining) {
          remaining -= item.restockCost
          return true
        }
        return false
      })
    })

    // Total cost of selected items
    const totalSelectedCost = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.restockCost, 0)
    )

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    // Initialize budget to maxBudget once data loads so all items start selected
    watch(maxBudget, (newMax) => {
      if (budget.value === 0 && newMax > 0) {
        budget.value = newMax
      }
    }, { immediate: true })

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [inv, forecasts] = await Promise.all([
          api.getInventory(),
          api.getDemandForecasts()
        ])
        inventoryItems.value = inv
        demandForecasts.value = forecasts
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const isSelected = (item) => recommendedItems.value.some(r => r.id === item.id)

    const formatCurrency = (value) => {
      return value.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 })
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0) return
      submitting.value = true
      orderSuccess.value = false
      orderError.value = null
      try {
        const orderItems = recommendedItems.value.map(item => ({
          sku: item.sku,
          name: item.name,
          quantity: item.restockQty,
          unit_price: item.unit_cost
        }))
        await api.createOrder({
          customer: 'Internal Restock',
          items: orderItems,
          total_value: totalSelectedCost.value,
          lead_time_days: leadTimeDays.value
        })
        orderSuccess.value = true
      } catch (err) {
        orderError.value = t('restocking.orderError')
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      t,
      loading,
      error,
      inventoryItems,
      demandForecasts,
      budget,
      leadTimeDays,
      submitting,
      orderSuccess,
      orderError,
      lowStockItems,
      maxBudget,
      recommendedItems,
      totalSelectedCost,
      currencySymbol,
      isSelected,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
.controls-card {
  margin-bottom: 1.5rem;
}

.slider-row {
  margin-bottom: 1.5rem;
}

.slider-row:last-child {
  margin-bottom: 0;
}

.slider-label {
  display: block;
  font-size: 0.875rem;
  font-weight: 600;
  color: #374151;
  margin-bottom: 0.5rem;
}

.slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
  display: block;
}

.slider-range {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-top: 0.375rem;
}

.order-summary {
  display: flex;
  align-items: center;
  gap: 2rem;
  padding-top: 1.25rem;
  margin-top: 1.25rem;
  border-top: 1px solid #e2e8f0;
  flex-wrap: wrap;
}

.summary-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.summary-stat-label {
  font-size: 0.72rem;
  font-weight: 700;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.summary-stat-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.btn-primary {
  margin-left: auto;
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.alert {
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.875rem;
  font-weight: 500;
}

.alert-success {
  background: #d1fae5;
  color: #065f46;
  border: 1px solid #6ee7b7;
}

.alert-danger {
  background: #fef2f2;
  color: #991b1b;
  border: 1px solid #fecaca;
}

.no-data {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.875rem;
}

.restocking-table {
  width: 100%;
  border-collapse: collapse;
}

.restocking-table th,
.restocking-table td {
  padding: 0.625rem 0.875rem;
  text-align: left;
  font-size: 0.875rem;
  border-bottom: 1px solid #f1f5f9;
}

.restocking-table th {
  font-size: 0.72rem;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  background: #f8fafc;
}

.col-num {
  text-align: right;
}

.row-selected {
  background: #eff6ff;
  transition: background 0.15s;
}

.row-selected:hover {
  background: #dbeafe;
}

.row-excluded {
  opacity: 0.45;
}

.sku-code {
  font-family: "SF Mono", "Fira Code", Consolas, monospace;
  font-size: 0.8rem;
  background: #f1f5f9;
  padding: 2px 6px;
  border-radius: 4px;
  color: #334155;
}

.badge.trend-increasing { background: #dcfce7; color: #166534; }
.badge.trend-stable     { background: #f1f5f9; color: #475569; }
.badge.trend-decreasing { background: #fef2f2; color: #991b1b; }
.badge.muted            { background: #f1f5f9; color: #94a3b8; }
</style>
