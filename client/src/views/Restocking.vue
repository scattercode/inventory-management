<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend items to restock based on demand forecasts within your budget</p>
    </div>

    <!-- Budget panel -->
    <div class="card budget-panel">
      <div class="card-header">
        <h3 class="card-title">Available Budget</h3>
        <span class="budget-value">${{ budget.toLocaleString() }}</span>
      </div>
      <div class="slider-row">
        <span class="range-label">$1,000</span>
        <input type="range" v-model.number="budget" min="1000" max="100000" step="1000" class="budget-slider" />
        <span class="range-label">$100,000</span>
      </div>
      <div class="budget-summary">
        <span>{{ selectedItems.length }} items selected</span>
        <span class="sep">·</span>
        <span>Total: ${{ totalCost.toLocaleString() }}</span>
        <span class="sep">·</span>
        <span>Remaining: ${{ remainingBudget.toLocaleString() }}</span>
      </div>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div v-if="orderSuccess" class="success-banner">
        <span>Order {{ lastOrderNumber }} placed successfully — expected delivery in 14 days.</span>
        <router-link to="/orders">View in Orders</router-link>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            Recommended Items
            <span class="count-chip">{{ selectedItems.length }} of {{ recommendations.length }} fit budget</span>
          </h3>
          <button
            class="btn-place-order"
            @click="placeOrder"
            :disabled="selectedItems.length === 0 || submitting"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
        <div class="table-container">
          <table class="orders-table">
            <thead>
              <tr>
                <th style="width:110px">SKU</th>
                <th>Item Name</th>
                <th style="width:130px">Current Demand</th>
                <th style="width:140px">Forecasted Demand</th>
                <th style="width:130px">Qty to Order</th>
                <th style="width:100px">Unit Cost</th>
                <th style="width:110px">Total Cost</th>
                <th style="width:130px">Status</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="isSelected(item) ? 'row-selected' : 'row-excluded'"
              >
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>{{ item.current_demand }}</td>
                <td>{{ item.forecasted_demand }}</td>
                <td><strong>{{ item.demand_gap }}</strong></td>
                <td>{{ item.unit_cost !== null ? '$' + item.unit_cost.toFixed(2) : '—' }}</td>
                <td>{{ item.item_cost !== null ? '$' + item.item_cost.toLocaleString() : '—' }}</td>
                <td>
                  <span v-if="isSelected(item)" class="badge success">Within Budget</span>
                  <span v-else-if="item.unit_cost === null" class="badge warning">No Cost Data</span>
                  <span v-else class="badge danger">Over Budget</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No items with increasing demand forecasts found.
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const orderSuccess = ref(false)
    const lastOrderNumber = ref('')
    const budget = ref(10000)
    const recommendations = ref([])

    // Greedy selection: pick items in demand_gap-desc order that fit remaining budget
    const selectedItems = computed(() => {
      let remaining = budget.value
      const selected = []
      for (const item of recommendations.value) {
        if (item.item_cost === null) continue
        if (item.item_cost <= remaining) {
          selected.push(item)
          remaining -= item.item_cost
        }
      }
      return selected
    })

    const totalCost = computed(() =>
      selectedItems.value.reduce((sum, item) => sum + item.item_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const isSelected = (item) => selectedItems.value.some(s => s.sku === item.sku)

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        recommendations.value = await api.getRestockingRecommendations()
      } catch (err) {
        error.value = 'Failed to load recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0 || submitting.value) return
      try {
        submitting.value = true
        orderSuccess.value = false
        const items = selectedItems.value.map(item => ({
          sku: item.sku,
          name: item.name,
          quantity: item.demand_gap,
          unit_price: item.unit_cost
        }))
        const order = await api.createRestockingOrder(items)
        lastOrderNumber.value = order.order_number
        orderSuccess.value = true
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => loadRecommendations())

    return {
      loading,
      error,
      submitting,
      orderSuccess,
      lastOrderNumber,
      budget,
      recommendations,
      selectedItems,
      totalCost,
      remainingBudget,
      isSelected,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-panel {
  margin-bottom: 1.5rem;
}
.budget-panel .card-header {
  margin-bottom: 1rem;
}
.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}
.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 0.75rem;
}
.range-label {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
}
.budget-slider {
  flex: 1;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
}
.budget-summary {
  font-size: 0.875rem;
  color: #64748b;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}
.sep {
  color: #cbd5e1;
}
.count-chip {
  display: inline-block;
  font-size: 0.75rem;
  font-weight: 500;
  background: #f1f5f9;
  color: #64748b;
  padding: 2px 8px;
  border-radius: 999px;
  margin-left: 0.5rem;
}
.btn-place-order {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-weight: 600;
  font-size: 0.875rem;
  cursor: pointer;
  transition: background 0.15s;
}
.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}
.btn-place-order:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}
.row-selected {
  background: #f0fdf4;
}
.row-excluded td {
  opacity: 0.45;
}
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 0.875rem;
}
.success-banner a {
  color: #065f46;
  font-weight: 600;
  text-decoration: underline;
}
.empty-state {
  text-align: center;
  padding: 2rem;
  color: #94a3b8;
  font-size: 0.875rem;
}
</style>
