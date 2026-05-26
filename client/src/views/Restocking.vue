<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Set your available budget to receive recommendations based on demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget control card -->
      <div class="card" style="margin-bottom: 1.25rem;">
        <div class="card-header">
          <div class="budget-header-row">
            <label class="budget-label">Available Budget</label>
            <span class="budget-amount">${{ budget.toLocaleString() }}</span>
          </div>
        </div>
        <div class="budget-slider-container">
          <input
            type="range"
            :min="0"
            :max="sliderMax"
            :step="5000"
            v-model.number="budget"
            class="budget-slider"
          />
          <div class="slider-labels">
            <span>$0</span>
            <span>${{ sliderMax.toLocaleString() }}</span>
          </div>
        </div>
      </div>

      <!-- Recommendations card -->
      <div class="card" style="margin-bottom: 1.25rem;">
        <div class="card-header">
          <div>
            <h3 class="card-title">Recommended Items</h3>
            <p v-if="recommendedItems.length > 0" class="card-subtitle">(increasing demand, within budget)</p>
          </div>
        </div>

        <div v-if="budget === 0 || recommendedItems.length === 0" class="empty-state">
          Increase your budget to see restocking recommendations
        </div>

        <div v-else>
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>Item Name</th>
                  <th>SKU</th>
                  <th>Trend</th>
                  <th>Qty to Order</th>
                  <th>Unit Cost</th>
                  <th>Total Cost</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in recommendedItems" :key="item.item_sku">
                  <td>{{ item.item_name }}</td>
                  <td>{{ item.item_sku }}</td>
                  <td><span :class="['badge', item.trend]">{{ item.trend }}</span></td>
                  <td>{{ item.forecasted_demand }}</td>
                  <td>${{ item.unit_cost.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
                  <td>${{ item.total_cost.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
                </tr>
              </tbody>
            </table>
          </div>

          <!-- Budget summary bar -->
          <div class="budget-summary">
            <div class="budget-summary-row">
              <span class="budget-summary-text">
                Using ${{ budgetUsed.toLocaleString() }} of ${{ budget.toLocaleString() }} budget
              </span>
              <span class="budget-summary-pct">{{ ((budgetUsed / budget) * 100).toFixed(0) }}%</span>
            </div>
            <div class="budget-bar-track">
              <div
                class="budget-bar-fill"
                :style="{ width: ((budgetUsed / budget) * 100).toFixed(0) + '%' }"
              ></div>
            </div>
          </div>
        </div>

        <!-- Success alert -->
        <div v-if="orderSubmitted" class="success-alert">
          <span>Order submitted — expected delivery in 14 days.</span>
          <button class="reset-link" @click="resetOrder">Place another order</button>
        </div>

        <!-- Error alert -->
        <div v-if="submitError" class="error-alert">{{ submitError }}</div>

        <!-- Place Order button -->
        <div class="order-button-container">
          <button
            class="place-order-btn"
            :class="{ disabled: recommendedItems.length === 0 || budget === 0 || orderSubmitted }"
            :disabled="recommendedItems.length === 0 || budget === 0 || orderSubmitted"
            @click="placeOrder"
          >
            Place Order
          </button>
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
    const budget = ref(0)
    const allRecommendations = ref([])
    const loading = ref(true)
    const error = ref(null)
    const orderSubmitted = ref(false)
    const submitError = ref(null)

    // Only increasing-trend items are candidates
    const increasingItems = computed(() =>
      allRecommendations.value.filter(i => i.trend === 'increasing')
    )

    // Slider max = sum of all increasing items' total_cost, rounded up to nearest $10k
    const sliderMax = computed(() => {
      const total = increasingItems.value.reduce((sum, i) => sum + i.total_cost, 0)
      return Math.ceil(total / 10000) * 10000
    })

    // Greedy fill: add increasing items (largest total_cost first) until budget exhausted
    const recommendedItems = computed(() => {
      const sorted = [...increasingItems.value].sort((a, b) => b.total_cost - a.total_cost)
      const result = []
      let running = 0
      for (const item of sorted) {
        if (running + item.total_cost <= budget.value) {
          result.push(item)
          running += item.total_cost
        }
      }
      return result
    })

    const budgetUsed = computed(() =>
      recommendedItems.value.reduce((sum, i) => sum + i.total_cost, 0)
    )

    const placeOrder = async () => {
      try {
        submitError.value = null
        await api.createRestockingOrder(recommendedItems.value)
        orderSubmitted.value = true
      } catch (err) {
        submitError.value = 'Failed to submit order: ' + err.message
      }
    }

    const resetOrder = () => {
      orderSubmitted.value = false
      budget.value = 0
    }

    onMounted(async () => {
      try {
        allRecommendations.value = await api.getRestockingRecommendations()
      } catch (err) {
        error.value = 'Failed to load restocking recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    })

    return {
      budget,
      allRecommendations,
      loading,
      error,
      orderSubmitted,
      submitError,
      increasingItems,
      sliderMax,
      recommendedItems,
      budgetUsed,
      placeOrder,
      resetOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 2rem;
}

.budget-header-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: 100%;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 500;
  color: #64748b;
}

.budget-amount {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-slider-container {
  padding: 0.75rem 1.5rem 1rem;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  margin-top: 0.375rem;
  font-size: 0.75rem;
  color: #64748b;
}

.card-subtitle {
  font-size: 0.813rem;
  color: #64748b;
  margin-top: 0.125rem;
}

.empty-state {
  padding: 2rem 1.5rem;
  color: #64748b;
  font-size: 0.875rem;
}

.budget-summary {
  padding: 1rem 1.5rem 0.5rem;
}

.budget-summary-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
}

.budget-summary-text {
  font-size: 0.875rem;
  color: #64748b;
}

.budget-summary-pct {
  font-size: 0.875rem;
  font-weight: 600;
  color: #0f172a;
}

.budget-bar-track {
  width: 100%;
  height: 6px;
  background: #e2e8f0;
  border-radius: 3px;
  overflow: hidden;
}

.budget-bar-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 3px;
  transition: width 0.2s ease;
}

.order-button-container {
  padding: 1rem 1.5rem 1.5rem;
}

.place-order-btn {
  width: 100%;
  padding: 0.875rem 2rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease;
}

.place-order-btn:hover:not(.disabled) {
  background: #1d4ed8;
}

.place-order-btn.disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-alert {
  margin: 0 1.5rem 0.75rem;
  padding: 0.875rem 1rem;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 6px;
  color: #065f46;
  font-size: 0.875rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.reset-link {
  background: none;
  border: none;
  color: #065f46;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  text-decoration: underline;
  padding: 0;
  white-space: nowrap;
}

.reset-link:hover {
  color: #047857;
}

.error-alert {
  margin: 0 1.5rem 0.75rem;
  padding: 0.875rem 1rem;
  background: #fee2e2;
  border: 1px solid #fca5a5;
  border-radius: 6px;
  color: #991b1b;
  font-size: 0.875rem;
}
</style>
