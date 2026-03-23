<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Auto-select items to restock within your available budget</p>
    </div>

    <!-- Success Banner -->
    <div v-if="successOrder" class="success-banner">
      Order <strong>{{ successOrder.id }}</strong> submitted — estimated delivery by {{ formatDate(successOrder.estimated_delivery) }}
    </div>

    <!-- Budget Card -->
    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Available Budget</h3>
        <span class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
      </div>
      <div class="slider-wrap">
        <span class="slider-label">{{ currencySymbol }}0</span>
        <input
          type="range"
          min="0"
          max="50000"
          step="500"
          v-model.number="budget"
          class="budget-slider"
        />
        <span class="slider-label">{{ currencySymbol }}50,000</span>
      </div>
      <div class="budget-meta">
        <span class="budget-remaining" :class="{ 'over-budget': selectedCost > budget }">
          Budget remaining: {{ currencySymbol }}{{ Math.max(0, budget - selectedCost).toLocaleString() }}
        </span>
        <span class="budget-used">
          Selected: {{ currencySymbol }}{{ selectedCost.toLocaleString() }}
        </span>
      </div>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            Recommended Items
            <span class="count-badge">{{ selectedItems.length }} / {{ recommendations.length }} selected</span>
          </h3>
        </div>
        <div class="table-container">
          <table class="rec-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th class="col-item">Item</th>
                <th class="col-cat">Category</th>
                <th class="col-trend">Trend</th>
                <th class="col-qty">Forecast Qty</th>
                <th class="col-cost">Unit Cost</th>
                <th class="col-total">Total Cost</th>
                <th class="col-lead">Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="{ 'row-selected': isSelected(item.sku), 'row-disabled': !isSelected(item.sku) && wouldExceedBudget(item) }"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="isSelected(item.sku)"
                    @change="toggleItem(item)"
                    :disabled="!isSelected(item.sku) && wouldExceedBudget(item)"
                  />
                </td>
                <td class="col-item">
                  <span class="item-name">{{ item.name }}</span>
                  <span class="item-sku">{{ item.sku }}</span>
                </td>
                <td class="col-cat">{{ item.category }}</td>
                <td class="col-trend">
                  <span :class="['badge', trendClass(item.trend)]">{{ item.trend }}</span>
                </td>
                <td class="col-qty">{{ item.restock_quantity.toLocaleString() }}</td>
                <td class="col-cost">{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
                <td class="col-total"><strong>{{ currencySymbol }}{{ item.total_cost.toLocaleString() }}</strong></td>
                <td class="col-lead">{{ item.lead_time_days }} days</td>
              </tr>
            </tbody>
          </table>
        </div>
        <div v-if="recommendations.length === 0" class="empty-state">
          No restocking recommendations available.
        </div>
      </div>

      <!-- Order Summary -->
      <div v-if="selectedItems.length > 0" class="card order-summary">
        <div class="card-header">
          <h3 class="card-title">Order Summary</h3>
          <span class="summary-total">{{ currencySymbol }}{{ selectedCost.toLocaleString() }}</span>
        </div>
        <div class="summary-items">
          <div v-for="item in selectedItems" :key="item.sku" class="summary-row">
            <span class="summary-name">{{ item.name }}</span>
            <span class="summary-detail">{{ item.restock_quantity.toLocaleString() }} units @ {{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</span>
            <span class="summary-cost">{{ currencySymbol }}{{ item.total_cost.toLocaleString() }}</span>
          </div>
        </div>
        <div class="summary-footer">
          <div class="summary-total-row">
            <span>Total</span>
            <strong>{{ currencySymbol }}{{ selectedCost.toLocaleString() }}</strong>
          </div>
          <button
            class="place-order-btn"
            :disabled="submitting || selectedItems.length === 0"
            @click="placeOrder"
          >
            {{ submitting ? 'Submitting...' : 'Place Order' }}
          </button>
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
    const { currentCurrency, currentLocale } = useI18n()

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const loading = ref(true)
    const error = ref(null)
    const recommendations = ref([])
    const budget = ref(25000)
    const selectedSkus = ref(new Set())
    const submitting = ref(false)
    const successOrder = ref(null)
    let successTimer = null

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        recommendations.value = await api.getRestockingRecommendations()
        autoSelect()
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // Greedy auto-selection: pick items in priority order until budget would be exceeded
    const autoSelect = () => {
      const newSelected = new Set()
      let remaining = budget.value
      for (const item of recommendations.value) {
        if (item.total_cost <= remaining) {
          newSelected.add(item.sku)
          remaining -= item.total_cost
        }
      }
      selectedSkus.value = newSelected
    }

    // Re-run auto-selection when budget changes
    watch(budget, () => {
      autoSelect()
      // Clear success banner when budget changes
      if (successTimer) clearTimeout(successTimer)
      successOrder.value = null
    })

    const selectedItems = computed(() =>
      recommendations.value.filter(r => selectedSkus.value.has(r.sku))
    )

    const selectedCost = computed(() =>
      selectedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
    )

    const isSelected = (sku) => selectedSkus.value.has(sku)

    const wouldExceedBudget = (item) => {
      return selectedCost.value + item.total_cost > budget.value
    }

    const toggleItem = (item) => {
      const next = new Set(selectedSkus.value)
      if (next.has(item.sku)) {
        next.delete(item.sku)
      } else {
        next.add(item.sku)
      }
      selectedSkus.value = next
    }

    const trendClass = (trend) => {
      return { increasing: 'increasing', stable: 'info', decreasing: 'danger' }[trend] || 'info'
    }

    const formatDate = (dateStr) => {
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      return new Date(dateStr).toLocaleDateString(locale, { year: 'numeric', month: 'short', day: 'numeric' })
    }

    const placeOrder = async () => {
      if (submitting.value || selectedItems.value.length === 0) return
      submitting.value = true
      try {
        const order = await api.submitRestockingOrder({
          budget: budget.value,
          items: selectedItems.value.map(item => ({
            sku: item.sku,
            name: item.name,
            category: item.category,
            quantity: item.restock_quantity,
            unit_cost: item.unit_cost,
          }))
        })
        successOrder.value = order
        // Auto-dismiss after 8 seconds
        if (successTimer) clearTimeout(successTimer)
        successTimer = setTimeout(() => { successOrder.value = null }, 8000)
        // Reset selections to auto-select for current budget
        autoSelect()
      } catch (err) {
        error.value = 'Failed to submit order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      loading,
      error,
      recommendations,
      budget,
      selectedItems,
      selectedCost,
      submitting,
      successOrder,
      currencySymbol,
      isSelected,
      wouldExceedBudget,
      toggleItem,
      trendClass,
      formatDate,
      placeOrder,
    }
  }
}
</script>

<style scoped>
.budget-card .card-header {
  align-items: center;
}

.budget-display {
  font-size: 1.75rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.slider-wrap {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin: 1rem 0 0.75rem;
}

.budget-slider {
  flex: 1;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.slider-label {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
}

.budget-meta {
  display: flex;
  justify-content: space-between;
  font-size: 0.875rem;
}

.budget-remaining {
  font-weight: 600;
  color: #059669;
}

.budget-remaining.over-budget {
  color: #dc2626;
}

.budget-used {
  color: #64748b;
}

.count-badge {
  margin-left: 0.625rem;
  font-size: 0.75rem;
  font-weight: 600;
  background: #eff6ff;
  color: #1d4ed8;
  padding: 0.25rem 0.625rem;
  border-radius: 20px;
}

.rec-table {
  table-layout: fixed;
  width: 100%;
}

.col-check { width: 40px; }
.col-item  { width: 220px; }
.col-cat   { width: 130px; }
.col-trend { width: 110px; }
.col-qty   { width: 110px; }
.col-cost  { width: 100px; }
.col-total { width: 110px; }
.col-lead  { width: 100px; }

.item-name {
  display: block;
  font-weight: 500;
  color: #0f172a;
  font-size: 0.875rem;
}

.item-sku {
  display: block;
  font-size: 0.75rem;
  color: #94a3b8;
  font-family: monospace;
}

.row-selected td {
  background: #f0f9ff;
}

.row-disabled {
  opacity: 0.45;
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #94a3b8;
}

/* Order Summary */
.order-summary .card-header {
  align-items: center;
}

.summary-total {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.summary-items {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.summary-row {
  display: grid;
  grid-template-columns: 1fr auto auto;
  gap: 1rem;
  align-items: center;
  padding: 0.5rem 0;
  border-bottom: 1px solid #f1f5f9;
  font-size: 0.875rem;
}

.summary-name {
  color: #0f172a;
  font-weight: 500;
}

.summary-detail {
  color: #64748b;
}

.summary-cost {
  font-weight: 600;
  color: #0f172a;
  text-align: right;
}

.summary-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 0.875rem;
  border-top: 2px solid #e2e8f0;
}

.summary-total-row {
  display: flex;
  align-items: baseline;
  gap: 0.75rem;
  font-size: 1rem;
  color: #64748b;
}

.summary-total-row strong {
  font-size: 1.375rem;
  font-weight: 700;
  color: #0f172a;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.625rem 1.75rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Success Banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
}
</style>
