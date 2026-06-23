<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}
            </h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Item summary -->
            <div class="item-summary">
              <div class="item-info">
                <div class="item-name">{{ backlogItem.item_name }}</div>
                <div class="item-sku">SKU: {{ backlogItem.item_sku || backlogItem.sku }}</div>
              </div>
              <div v-if="mode === 'view' && backlogItem.purchase_order" class="po-status-badge" :class="backlogItem.purchase_order.status">
                {{ backlogItem.purchase_order.status }}
              </div>
            </div>

            <!-- Create mode: form -->
            <form v-if="mode === 'create'" @submit.prevent="submitPO" class="po-form">
              <div class="form-group">
                <label class="form-label">Supplier</label>
                <input
                  v-model="form.supplier"
                  type="text"
                  class="form-input"
                  placeholder="Enter supplier name"
                  required
                />
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label class="form-label">Quantity</label>
                  <input
                    v-model.number="form.quantity"
                    type="number"
                    class="form-input"
                    min="1"
                    required
                  />
                </div>

                <div class="form-group">
                  <label class="form-label">Unit Cost (USD)</label>
                  <input
                    v-model.number="form.unit_cost"
                    type="number"
                    class="form-input"
                    min="0"
                    step="0.01"
                    required
                  />
                </div>
              </div>

              <div class="form-group">
                <label class="form-label">Expected Delivery</label>
                <input
                  v-model="form.expected_delivery"
                  type="date"
                  class="form-input"
                  required
                />
              </div>

              <div v-if="form.quantity && form.unit_cost" class="total-estimate">
                <span class="total-label">Estimated Total</span>
                <span class="total-value">${{ (form.quantity * form.unit_cost).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</span>
              </div>
            </form>

            <!-- View mode: read-only PO details -->
            <div v-else-if="mode === 'view' && backlogItem.purchase_order" class="po-details">
              <div class="info-grid">
                <div class="info-item">
                  <div class="info-label">PO ID</div>
                  <div class="info-value po-id">{{ backlogItem.purchase_order.id }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Status</div>
                  <div class="info-value">
                    <span class="po-status-badge" :class="backlogItem.purchase_order.status">
                      {{ backlogItem.purchase_order.status }}
                    </span>
                  </div>
                </div>

                <div class="info-item">
                  <div class="info-label">Supplier</div>
                  <div class="info-value">{{ backlogItem.purchase_order.supplier || 'N/A' }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Quantity</div>
                  <div class="info-value">{{ backlogItem.purchase_order.quantity }} units</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Unit Cost</div>
                  <div class="info-value">${{ Number(backlogItem.purchase_order.unit_cost).toFixed(2) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Total Value</div>
                  <div class="info-value">
                    ${{ (backlogItem.purchase_order.quantity * backlogItem.purchase_order.unit_cost).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
                  </div>
                </div>

                <div class="info-item">
                  <div class="info-label">Expected Delivery</div>
                  <div class="info-value">{{ formatDate(backlogItem.purchase_order.expected_delivery) }}</div>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-cancel" type="button" @click="close">Cancel</button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              type="button"
              @click="submitPO"
              :disabled="!isFormValid"
            >
              Submit Purchase Order
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  isOpen: {
    type: Boolean,
    default: false
  },
  backlogItem: {
    type: Object,
    default: null
  },
  mode: {
    type: String,
    default: 'create' // 'create' | 'view'
  }
})

const emit = defineEmits(['close', 'po-created'])

// Form state for create mode
const form = ref({
  supplier: '',
  quantity: 0,
  unit_cost: 0,
  expected_delivery: ''
})

// Pre-fill form whenever the modal opens with a new backlog item
watch(() => [props.isOpen, props.backlogItem], ([isOpen, item]) => {
  if (isOpen && item && props.mode === 'create') {
    form.value = {
      supplier: '',
      quantity: item.quantity_needed || 0,
      unit_cost: item.estimated_cost || 0,
      expected_delivery: ''
    }
  }
}, { immediate: true })

const isFormValid = computed(() => {
  return form.value.supplier.trim() &&
    form.value.quantity > 0 &&
    form.value.unit_cost >= 0 &&
    form.value.expected_delivery
})

const submitPO = () => {
  if (!isFormValid.value) return

  const poData = {
    id: 'PO-' + Date.now(),
    backlog_item_id: props.backlogItem.id,
    status: 'pending',
    supplier: form.value.supplier.trim(),
    quantity: form.value.quantity,
    unit_cost: form.value.unit_cost,
    expected_delivery: form.value.expected_delivery
  }

  emit('po-created', poData)
}

const close = () => {
  emit('close')
}

const formatDate = (dateString) => {
  if (!dateString) return 'N/A'
  const date = new Date(dateString)
  if (isNaN(date.getTime())) return dateString
  return date.toLocaleDateString('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  })
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 10px;
  border: 1px solid #e2e8f0;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 560px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 1.5rem;
}

.item-summary {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 1.25rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  margin-bottom: 1.5rem;
}

.item-name {
  font-size: 0.938rem;
  font-weight: 600;
  color: #0f172a;
  margin-bottom: 0.25rem;
}

.item-sku {
  font-size: 0.813rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.form-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.form-input {
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  padding: 0.5rem 0.75rem;
  font-size: 0.875rem;
  color: #0f172a;
  font-family: inherit;
  transition: border-color 0.15s ease;
  outline: none;
}

.form-input:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.total-estimate {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.875rem 1.25rem;
  background: #eff6ff;
  border: 1px solid #bfdbfe;
  border-radius: 8px;
  margin-top: 0.25rem;
}

.total-label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #1e40af;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.total-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #1e40af;
}

.po-details {
  /* view mode details container */
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.25rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.info-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.info-value.po-id {
  font-family: 'Monaco', 'Courier New', monospace;
  color: #2563eb;
  font-size: 0.875rem;
}

.po-status-badge {
  display: inline-block;
  padding: 0.25rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.po-status-badge.pending {
  background: #fed7aa;
  color: #92400e;
}

.po-status-badge.approved {
  background: #d1fae5;
  color: #065f46;
}

.po-status-badge.shipped {
  background: #dbeafe;
  color: #1e40af;
}

.po-status-badge.delivered {
  background: #d1fae5;
  color: #065f46;
}

.po-status-badge.cancelled {
  background: #fecaca;
  color: #991b1b;
}

.modal-footer {
  padding: 1.25rem 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.btn-cancel {
  padding: 0.5rem 1.25rem;
  background: transparent;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
  color: #64748b;
  cursor: pointer;
  font-family: inherit;
  transition: all 0.15s ease;
}

.btn-cancel:hover {
  background: #f1f5f9;
  border-color: #cbd5e1;
  color: #334155;
}

.btn-primary {
  padding: 0.5rem 1.25rem;
  background: #2563eb;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  color: white;
  cursor: pointer;
  font-family: inherit;
  transition: all 0.15s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Modal transition animations */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
