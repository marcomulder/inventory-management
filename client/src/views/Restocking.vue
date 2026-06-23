<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error && !successMessage" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card budget-card">
        <div class="budget-value">{{ formatBudget(selectedBudget) }}</div>
        <div class="budget-label">{{ t('restocking.budgetLabel') }}</div>
        <input
          type="range"
          min="0"
          max="250000"
          step="1000"
          v-model.number="selectedBudget"
          class="budget-slider"
        />
        <div class="budget-ticks">
          <span>$0</span>
          <span>$50k</span>
          <span>$100k</span>
          <span>$150k</span>
          <span>$200k</span>
          <span>$250k</span>
        </div>
      </div>

      <!-- Summary Bar -->
      <div class="summary-bar">
        <div class="summary-stat">
          <span class="summary-stat-label">{{ t('restocking.recommendedItems') }}</span>
          <span class="summary-stat-value">{{ recommendations.length }} items</span>
        </div>
        <div class="summary-stat">
          <span class="summary-stat-label">{{ t('restocking.totalCost') }}</span>
          <span class="summary-stat-value">{{ formatCurrency(totalCost) }}</span>
        </div>
        <div class="summary-stat">
          <span class="summary-stat-label">{{ t('restocking.remaining') }}</span>
          <span class="summary-stat-value" :class="{ danger: remainingBudget === 0 }">{{ formatCurrency(remainingBudget) }}</span>
        </div>
        <button
          class="place-order-btn"
          :disabled="recommendations.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? t('restocking.submitting') : t('restocking.placeOrder') }}
        </button>
      </div>

      <!-- Success Banner -->
      <div v-if="successMessage" class="success-banner">
        <span>{{ successMessage }}</span>
        <button class="success-dismiss" @click="successMessage = null">&times;</button>
      </div>

      <!-- Error Banner (shown alongside success or after submit failure) -->
      <div v-if="error && successMessage === null && !loading" class="error">{{ error }}</div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendedItems') }}</h3>
          <span class="badge info">{{ recommendations.length }}</span>
        </div>
        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noItemsInBudget') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.name') }}</th>
                <th>{{ t('restocking.table.quantity') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.lineCost') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.sku">
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>{{ item.quantity_to_order }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.line_cost) }}</strong></td>
                <td>
                  <span :class="['badge', trendBadgeClass(item.trend)]">
                    {{ trendLabel(item.trend) }}
                  </span>
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
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref(null)
    const forecasts = ref([])
    const inventoryItems = ref([])
    const selectedBudget = ref(50000)

    // Load both data sources in parallel on mount
    onMounted(async () => {
      try {
        loading.value = true
        const [forecastData, invData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        forecasts.value = forecastData
        inventoryItems.value = invData
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    })

    // Greedy cheapest-first algorithm to fill budget with recommended items
    const recommendations = computed(() => {
      // Build SKU → inventory item lookup
      const inventoryBySku = {}
      inventoryItems.value.forEach(item => { inventoryBySku[item.sku] = item })

      // Map forecasts to candidate items, skip decreasing trend and unmatched SKUs
      const candidates = forecasts.value
        .filter(f => f.trend !== 'decreasing')
        .map(f => {
          const inv = inventoryBySku[f.item_sku]
          if (!inv) return null
          return {
            sku: f.item_sku,
            name: inv.name,
            quantity_to_order: f.forecasted_demand,
            unit_cost: inv.unit_cost,
            line_cost: f.forecasted_demand * inv.unit_cost,
            trend: f.trend,
          }
        })
        .filter(Boolean)

      // Sort cheapest first, then greedily fill within budget
      candidates.sort((a, b) => a.line_cost - b.line_cost)
      let remaining = selectedBudget.value
      return candidates.filter(item => {
        if (item.line_cost <= remaining) { remaining -= item.line_cost; return true }
        return false
      })
    })

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, i) => sum + i.line_cost, 0)
    )

    const remainingBudget = computed(() => selectedBudget.value - totalCost.value)

    const placeOrder = async () => {
      if (!recommendations.value.length || submitting.value) return
      try {
        submitting.value = true
        error.value = null
        successMessage.value = null
        await api.createRestockingOrder({
          items: recommendations.value.map(r => ({
            sku: r.sku,
            name: r.name,
            quantity_to_order: r.quantity_to_order,
            unit_cost: r.unit_cost
          }))
        })
        successMessage.value = t('restocking.orderPlaced')
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    const formatCurrency = (value) =>
      '$' + value.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })

    const formatBudget = (value) =>
      '$' + Number(value).toLocaleString('en-US', { maximumFractionDigits: 0 })

    const trendBadgeClass = (trend) =>
      ({ increasing: 'success', stable: 'info', decreasing: 'danger' }[trend] || 'info')

    const trendLabel = (trend) => t(`restocking.trend.${trend}`) || trend

    return {
      t,
      loading,
      error,
      submitting,
      successMessage,
      selectedBudget,
      recommendations,
      totalCost,
      remainingBudget,
      placeOrder,
      formatCurrency,
      formatBudget,
      trendBadgeClass,
      trendLabel
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-value {
  font-size: 2.25rem;
  font-weight: 700;
  color: #2563eb;
  margin-bottom: 0.5rem;
  line-height: 1;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.75rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
  margin-bottom: 0.5rem;
}

.budget-ticks {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
}

.summary-bar {
  display: flex;
  align-items: center;
  gap: 2rem;
  background: #eff6ff;
  border: 1px solid #bfdbfe;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
}

.summary-stat {
  display: flex;
  flex-direction: column;
  gap: 0.2rem;
}

.summary-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.summary-stat-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.summary-stat-value.danger {
  color: #dc2626;
}

.place-order-btn {
  margin-left: auto;
  padding: 0.75rem 1.75rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
  cursor: pointer;
  transition: background 0.2s;
  white-space: nowrap;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.success-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.5rem;
  font-weight: 500;
}

.success-dismiss {
  background: none;
  border: none;
  color: #065f46;
  cursor: pointer;
  font-size: 1.25rem;
  padding: 0;
  line-height: 1;
}

.empty-state {
  padding: 3rem 1.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}
</style>
