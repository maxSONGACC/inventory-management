<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div v-if="submitResult" class="submit-success">
        <div class="submit-success-text">
          {{ t('restocking.orderPlaced', { orderNumber: submitResult.order_number }) }}
        </div>
        <router-link to="/orders" class="submit-success-link">
          {{ t('restocking.viewInOrders') }}
        </router-link>
      </div>
      <div v-if="submitError" class="error">{{ submitError }}</div>

      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget') }}</h3>
          <span class="budget-amount">{{ formatMoney(budget) }}</span>
        </div>
        <input
          v-model.number="budget"
          type="range"
          class="budget-slider"
          min="0"
          :max="maxBudget"
          :step="budgetStep"
          :aria-label="t('restocking.budget')"
        >
        <div class="budget-scale">
          <span>{{ formatMoney(0) }}</span>
          <span>{{ formatMoney(maxBudget) }}</span>
        </div>
        <p class="budget-help">{{ t('restocking.budgetHelp') }}</p>
      </div>

      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.allocated') }}</div>
          <div class="stat-value">{{ formatMoney(allocatedTotal) }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.remaining') }}</div>
          <div class="stat-value">{{ formatMoney(remainingBudget) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('restocking.itemsSelected') }}</div>
          <div class="stat-value">{{ recommendations.length }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('restocking.totalUnits') }}</div>
          <div class="stat-value">{{ totalUnits.toLocaleString() }}</div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t('restocking.recommended') }} ({{ recommendations.length }})
          </h3>
        </div>
        <p class="card-help">{{ t('restocking.recommendedHelp') }}</p>

        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th class="num">{{ t('restocking.table.shortfall') }}</th>
                <th class="num">{{ t('restocking.table.orderQty') }}</th>
                <th class="num">{{ t('restocking.table.unitCost') }}</th>
                <th class="num">{{ t('restocking.table.lineTotal') }}</th>
                <th class="num">{{ t('restocking.table.leadTime') }}</th>
                <th>{{ t('restocking.table.coverage') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.item_sku">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ translateProductName(item.item_name) }}</td>
                <td class="num">{{ item.shortfall.toLocaleString() }}</td>
                <td class="num"><strong>{{ item.quantity.toLocaleString() }}</strong></td>
                <td class="num">{{ formatUnitCost(item.unit_cost) }}</td>
                <td class="num">{{ formatMoney(item.line_total) }}</td>
                <td class="num">{{ t('orders.leadTimeDays', { days: item.lead_time_days }) }}</td>
                <td>
                  <span :class="['badge', item.isPartial ? 'warning' : 'success']">
                    {{ item.isPartial ? t('restocking.partial') : t('restocking.full') }}
                  </span>
                  <span class="coverage-pct">{{ item.coveragePct }}%</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="order-actions">
          <button
            class="place-order-btn"
            :disabled="recommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency, formatCurrencyWithDecimals } from '../utils/currency'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])

    const budget = ref(0)
    const submitting = ref(false)
    const submitResult = ref(null)
    const submitError = ref(null)

    const formatMoney = (amount) => formatCurrency(amount, currentCurrency.value)
    // Unit costs need cents to stay meaningful; totals are rounded like the rest of the app
    const formatUnitCost = (amount) => formatCurrencyWithDecimals(amount, currentCurrency.value, 2)

    // Items whose forecast exceeds current demand are the only restock candidates.
    // A 'decreasing' item has no shortfall and is excluded by this filter.
    const shortfallItems = computed(() => {
      return forecasts.value
        .map(f => ({ ...f, shortfall: f.forecasted_demand - f.current_demand }))
        .filter(f => f.shortfall > 0)
    })

    // Cost of fully covering every shortfall - the slider's upper bound
    const fullCoverageCost = computed(() => {
      return shortfallItems.value.reduce((sum, f) => sum + f.shortfall * f.unit_cost, 0)
    })

    const budgetStep = 1000

    // Round the ceiling up to a whole step so the slider can always reach full coverage
    const maxBudget = computed(() => {
      if (fullCoverageCost.value === 0) return budgetStep
      return Math.ceil(fullCoverageCost.value / budgetStep) * budgetStep
    })

    // Greedy allocation: highest shortfall first, filling each item's full need
    // while the budget allows, then partially funding the next one that fits.
    const recommendations = computed(() => {
      const ranked = [...shortfallItems.value].sort((a, b) => {
        if (b.shortfall !== a.shortfall) return b.shortfall - a.shortfall
        // Tie-break toward items whose demand is still climbing
        const aRising = a.trend === 'increasing' ? 0 : 1
        const bRising = b.trend === 'increasing' ? 0 : 1
        return aRising - bRising
      })

      let remaining = budget.value
      const picked = []

      for (const item of ranked) {
        if (remaining < item.unit_cost) continue

        const fullCost = item.shortfall * item.unit_cost
        const quantity = fullCost <= remaining
          ? item.shortfall
          : Math.floor(remaining / item.unit_cost)

        if (quantity < 1) continue

        const lineTotal = Math.round(quantity * item.unit_cost * 100) / 100
        remaining -= lineTotal

        picked.push({
          item_sku: item.item_sku,
          item_name: item.item_name,
          quantity,
          unit_cost: item.unit_cost,
          line_total: lineTotal,
          lead_time_days: item.lead_time_days,
          shortfall: item.shortfall,
          coveragePct: Math.round((quantity / item.shortfall) * 100),
          isPartial: quantity < item.shortfall
        })
      }

      return picked
    })

    const allocatedTotal = computed(() => {
      return Math.round(recommendations.value.reduce((sum, i) => sum + i.line_total, 0) * 100) / 100
    })

    const remainingBudget = computed(() => {
      return Math.round((budget.value - allocatedTotal.value) * 100) / 100
    })

    const totalUnits = computed(() => {
      return recommendations.value.reduce((sum, i) => sum + i.quantity, 0)
    })

    const loadForecasts = async () => {
      try {
        loading.value = true
        error.value = null
        forecasts.value = await api.getDemandForecasts()
        // Open on half the full-coverage cost so the page lands on a useful default
        budget.value = Math.round(maxBudget.value / 2 / budgetStep) * budgetStep
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendations.value.length === 0 || submitting.value) return

      try {
        submitting.value = true
        submitError.value = null
        submitResult.value = null

        submitResult.value = await api.createRestockingOrder({
          budget: budget.value,
          items: recommendations.value.map(item => ({
            item_sku: item.item_sku,
            quantity: item.quantity
          }))
        })
      } catch (err) {
        const detail = err.response && err.response.data && err.response.data.detail
        submitError.value = t('restocking.orderFailed') + (detail ? ': ' + detail : '')
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      t,
      translateProductName,
      loading,
      error,
      budget,
      budgetStep,
      maxBudget,
      recommendations,
      allocatedTotal,
      remainingBudget,
      totalUnits,
      submitting,
      submitResult,
      submitError,
      placeOrder,
      formatMoney,
      formatUnitCost
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-amount {
  font-size: 1.5rem;
  font-weight: 700;
  color: var(--ink);
  letter-spacing: -0.03em;
}

.budget-slider {
  width: 100%;
  margin: 0.75rem 0 0.25rem;
  accent-color: var(--purple);
  cursor: pointer;
}

.budget-scale {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: var(--gray);
}

.budget-help {
  margin: 0.75rem 0 0;
  font-size: 0.875rem;
  color: var(--gray);
}

.card-help {
  margin: 0 0 1rem;
  font-size: 0.875rem;
  color: var(--gray);
}

.num {
  text-align: right;
}

.coverage-pct {
  margin-left: 0.5rem;
  font-size: 0.75rem;
  color: var(--gray);
}

.empty-state {
  padding: 2.5rem 1rem;
  text-align: center;
  color: var(--gray);
  font-size: 0.938rem;
  background: var(--line-soft);
  border: 1px dashed var(--line);
  border-radius: var(--radius);
}

.order-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 1.25rem;
}

/* Brand wipe applied to the element itself rather than an ::before overlay,
   so it can never paint over the button label. */
.place-order-btn {
  background-color: var(--purple);
  background-image: linear-gradient(90deg, var(--purple-deep) 50%, var(--purple) 0);
  background-position-x: 100%;
  background-size: 200% 200%;
  color: #fff;
  border: none;
  border-radius: var(--radius);
  padding: 0.875rem 1.75rem;
  font-family: inherit;
  font-size: 0.938rem;
  font-weight: 500;
  letter-spacing: -0.005em;
  cursor: pointer;
  transition: background-position-x 550ms var(--ease-brand);
}

.place-order-btn:hover:not(:disabled) {
  background-position-x: 0%;
}

.place-order-btn:disabled {
  background-color: var(--line);
  background-image: none;
  color: var(--gray-warm);
  cursor: not-allowed;
}

.submit-success {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  background: #f4e8ff;
  border: 1px solid var(--purple-pale);
  border-left: 3px solid var(--purple);
  color: var(--purple-dark);
  padding: 1rem;
  border-radius: var(--radius);
  margin-bottom: 1rem;
}

.submit-success-text {
  font-size: 0.938rem;
  font-weight: 600;
}

.submit-success-link {
  color: var(--purple-dark);
  font-size: 0.875rem;
  font-weight: 600;
  text-decoration: underline;
  white-space: nowrap;
}
</style>
