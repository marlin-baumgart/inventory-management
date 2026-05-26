<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set your budget and submit a restocking order based on demand forecasts and stock urgency.</p>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="loadError" class="error">{{ loadError }}</div>
    <div v-else>
      <!-- Budget control card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-control">
          <div class="budget-slider-row">
            <input
              type="range"
              class="budget-slider"
              min="0"
              max="200000"
              step="1000"
              :value="budget"
              @input="onBudgetInput"
            />
            <span class="budget-value">{{ formatCurrency(budget) }}</span>
          </div>
          <div class="budget-stats">
            <div class="budget-stat">
              <span class="budget-stat-label">Selected total</span>
              <span class="budget-stat-value">{{ formatCurrency(selectedTotal) }}</span>
            </div>
            <div class="budget-stat">
              <span class="budget-stat-label">Remaining</span>
              <span class="budget-stat-value" :class="{ 'remaining-negative': remaining < 0 }">{{ formatCurrency(remaining) }}</span>
            </div>
            <div class="budget-stat">
              <span class="budget-stat-label">Items selected</span>
              <span class="budget-stat-value">{{ selectedSkus.size }}</span>
            </div>
          </div>
        </div>
      </div>

      <!-- Recommendations table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations ({{ recommendations.length }})</h3>
        </div>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-sku">SKU</th>
                <th class="col-item">Item</th>
                <th class="col-category">Category</th>
                <th class="col-urgency">Urgency</th>
                <th class="col-stock">On-hand / Reorder</th>
                <th class="col-forecast">Forecast</th>
                <th class="col-qty">Rec. Qty</th>
                <th class="col-unit-cost">Unit Cost</th>
                <th class="col-line-total">Line Total</th>
                <th class="col-lead">Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="rec in recommendations"
                :key="rec.sku"
                :class="{ 'row-selected': selectedSkus.has(rec.sku) }"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(rec.sku)"
                    @change="onCheckboxChange(rec.sku, $event.target.checked)"
                  />
                </td>
                <td class="col-sku"><code>{{ rec.sku }}</code></td>
                <td class="col-item">{{ rec.name }}</td>
                <td class="col-category">{{ rec.category }}</td>
                <td class="col-urgency">
                  <span :class="['badge', rec.urgency === 'critical' ? 'danger' : 'warning']">
                    {{ rec.urgency }}
                  </span>
                </td>
                <td class="col-stock">{{ rec.quantity_on_hand }} / {{ rec.reorder_point }}</td>
                <td class="col-forecast">
                  {{ rec.forecasted_demand }}
                  <span v-if="rec.forecast_source === 'reorder_proxy'" class="forecast-proxy">(est.)</span>
                </td>
                <td class="col-qty">{{ rec.recommended_quantity }}</td>
                <td class="col-unit-cost">{{ formatCurrency(rec.unit_cost) }}</td>
                <td class="col-line-total"><strong>{{ formatCurrency(rec.line_total) }}</strong></td>
                <td class="col-lead">{{ rec.lead_time_days }}d</td>
              </tr>
            </tbody>
          </table>
        </div>
        <p class="forecast-note">* (est.) indicates that the demand forecast is estimated from the reorder point rather than historical order data.</p>
      </div>

      <!-- Place order action -->
      <div class="card order-action-card">
        <div class="order-action-row">
          <div class="order-action-summary">
            <span v-if="selectedSkus.size > 0">
              {{ selectedSkus.size }} item{{ selectedSkus.size !== 1 ? 's' : '' }} selected &mdash; {{ formatCurrency(selectedTotal) }}
            </span>
            <span v-else class="muted">No items selected. Adjust budget or select items manually.</span>
          </div>
          <button
            class="btn-place-order"
            :disabled="selectedSkus.size === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
        <div v-if="orderSuccess" class="order-success">
          Order {{ orderSuccess.order_number }} placed. Expected delivery: {{ formatDate(orderSuccess.expected_delivery) }}
        </div>
        <div v-if="orderError" class="error">{{ orderError }}</div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const recommendations = ref([])
    const loading = ref(true)
    const loadError = ref(null)
    const budget = ref(50000)
    const submitting = ref(false)
    const orderSuccess = ref(null)
    const orderError = ref(null)

    // Tracks SKUs that the user has manually toggled. When the budget slider
    // moves, we reset manualOverrides so auto-fill can reassert itself — an
    // acceptable demo trade-off to keep the logic simple.
    const manualOverrides = ref({}) // { [sku]: true (force-on) | false (force-off) }

    // Greedy auto-fill: walk sorted recommendations top-down and check each
    // item whose line_total fits within the remaining budget. Manual overrides
    // take precedence over the auto-fill result for individual SKUs.
    const selectedSkus = computed(() => {
      const result = new Set()
      let spent = 0

      for (const rec of recommendations.value) {
        const hasOverride = rec.sku in manualOverrides.value
        if (hasOverride) {
          if (manualOverrides.value[rec.sku]) {
            result.add(rec.sku)
            spent += rec.line_total
          }
          // false override: skip regardless of budget
        } else {
          // Auto-fill: include if it fits
          if (spent + rec.line_total <= budget.value) {
            result.add(rec.sku)
            spent += rec.line_total
          }
        }
      }

      return result
    })

    const selectedTotal = computed(() => {
      return recommendations.value
        .filter(r => selectedSkus.value.has(r.sku))
        .reduce((sum, r) => sum + r.line_total, 0)
    })

    const remaining = computed(() => budget.value - selectedTotal.value)

    const formatCurrency = (value) => {
      return '$' + value.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })
    }

    const formatDate = (dateString) => {
      const d = new Date(dateString)
      if (isNaN(d.getTime())) return dateString
      return d.toLocaleDateString('en-US', { year: 'numeric', month: 'short', day: 'numeric' })
    }

    const onBudgetInput = (event) => {
      // Reset manual overrides so auto-fill takes over with the new budget
      manualOverrides.value = {}
      budget.value = Number(event.target.value)
    }

    const onCheckboxChange = (sku, checked) => {
      // Record a manual override for this SKU
      manualOverrides.value = { ...manualOverrides.value, [sku]: checked }
    }

    const loadRecommendations = async () => {
      loading.value = true
      loadError.value = null
      try {
        recommendations.value = await api.getRestockRecommendations()
      } catch (err) {
        loadError.value = 'Failed to load recommendations: ' + (err.response?.data?.detail || err.message)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedSkus.value.size === 0) return
      submitting.value = true
      orderSuccess.value = null
      orderError.value = null

      const items = recommendations.value
        .filter(r => selectedSkus.value.has(r.sku))
        .map(r => ({ sku: r.sku, quantity: r.recommended_quantity }))

      try {
        const created = await api.createRestockOrder({ items })
        orderSuccess.value = created
        // Clear selections by resetting overrides; auto-fill with same budget
        manualOverrides.value = {}
        await loadRecommendations()
      } catch (err) {
        orderError.value = 'Failed to place order: ' + (err.response?.data?.detail || err.message)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      recommendations,
      loading,
      loadError,
      budget,
      selectedSkus,
      selectedTotal,
      remaining,
      submitting,
      orderSuccess,
      orderError,
      formatCurrency,
      formatDate,
      onBudgetInput,
      onCheckboxChange,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

/* Budget card */
.budget-control {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.budget-slider-row {
  display: flex;
  align-items: center;
  gap: 1.5rem;
}

.budget-slider {
  flex: 1;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  min-width: 10ch;
  text-align: right;
}

.budget-stats {
  display: flex;
  gap: 2rem;
}

.budget-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.budget-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.budget-stat-value {
  font-size: 1rem;
  font-weight: 600;
  color: #0f172a;
}

.remaining-negative {
  color: #dc2626;
}

/* Recommendations table */
.restock-table {
  table-layout: auto;
  width: 100%;
}

.col-check {
  width: 36px;
}

.col-sku {
  width: 110px;
}

.col-item {
  min-width: 160px;
}

.col-category {
  width: 130px;
}

.col-urgency {
  width: 100px;
}

.col-stock {
  width: 120px;
  white-space: nowrap;
}

.col-forecast {
  width: 90px;
}

.col-qty {
  width: 90px;
}

.col-unit-cost {
  width: 100px;
}

.col-line-total {
  width: 110px;
}

.col-lead {
  width: 80px;
}

code {
  font-size: 0.8rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #334155;
}

.row-selected {
  background: #eff6ff;
}

.row-selected:hover {
  background: #dbeafe !important;
}

.forecast-proxy {
  font-size: 0.75rem;
  color: #94a3b8;
  margin-left: 0.25rem;
}

.forecast-note {
  margin-top: 0.875rem;
  font-size: 0.8rem;
  color: #94a3b8;
  padding-top: 0.875rem;
  border-top: 1px solid #f1f5f9;
}

/* Order action card */
.order-action-card {
  margin-bottom: 2rem;
}

.order-action-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.order-action-summary {
  font-size: 0.938rem;
  color: #334155;
}

.muted {
  color: #94a3b8;
}

.btn-place-order {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
  white-space: nowrap;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.order-success {
  margin-top: 0.875rem;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.75rem 1rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 500;
}
</style>
