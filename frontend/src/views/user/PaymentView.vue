<template>
  <AppLayout>
    <div class="relative mx-auto max-w-7xl pb-8">
      <div class="pointer-events-none absolute inset-0 -z-10 opacity-40 [background-image:linear-gradient(rgba(226,232,240,0.5)_1px,transparent_1px),linear-gradient(90deg,rgba(226,232,240,0.5)_1px,transparent_1px)] [background-size:44px_44px]"></div>

      <div v-if="loading" class="flex items-center justify-center py-20">
        <div class="h-8 w-8 animate-spin rounded-full border-4 border-primary-500 border-t-transparent"></div>
      </div>

      <template v-else>
        <template v-if="paymentPhase === 'paying'">
          <PaymentStatusPanel
            :order-id="paymentState.orderId"
            :qr-code="paymentState.qrCode"
            :expires-at="paymentState.expiresAt"
            :payment-type="paymentState.paymentType"
            :pay-url="paymentState.payUrl"
            :order-type="paymentState.orderType"
            @done="onPaymentDone"
            @success="onPaymentSuccess"
          />
        </template>

        <template v-else-if="paymentPhase === 'stripe'">
          <StripePaymentInline
            :order-id="paymentState.orderId"
            :amount="paymentState.amount"
            :client-secret="paymentState.clientSecret"
            :order-type="paymentState.orderType || undefined"
            :publishable-key="checkout.stripe_publishable_key"
            :pay-amount="paymentState.payAmount"
            @success="onPaymentSuccess"
            @done="onStripeDone"
            @back="resetPayment"
            @redirect="onStripeRedirect"
          />
        </template>

        <template v-else>
          <section class="overflow-hidden rounded-[26px] border border-orange-100/80 bg-[linear-gradient(135deg,rgba(255,255,255,0.96),rgba(255,247,237,0.92))] px-6 py-5 shadow-[0_18px_40px_rgba(15,23,42,0.05)] dark:border-dark-700 dark:bg-[linear-gradient(135deg,rgba(17,24,39,0.96),rgba(28,25,23,0.92))]">
            <div class="flex flex-col gap-4 lg:flex-row lg:items-start lg:justify-between">
              <h1 class="text-3xl font-semibold tracking-tight text-slate-900 dark:text-white">{{ t('payment.purchaseHeroTitle') }}</h1>
              <div class="flex flex-wrap items-center gap-x-6 gap-y-2 text-sm text-slate-700 dark:text-slate-200">
                <p>{{ t('payment.currentUserEmail') }}: <span class="font-semibold text-slate-900 dark:text-white">{{ userIdentity }}</span></p>
                <p>{{ t('payment.currentBalanceLabel') }}: <span class="font-semibold text-slate-900 dark:text-white">{{ user?.balance?.toFixed(2) || '0.00' }}</span></p>
              </div>
            </div>
            <div class="my-5 border-t border-orange-100/80 dark:border-dark-700"></div>
            <div class="space-y-3">
              <p class="text-base font-semibold text-slate-900 dark:text-white">{{ t('payment.purchaseNotes') }}</p>
              <ol class="space-y-2 text-sm leading-6 text-slate-700 dark:text-slate-300">
                <li v-for="(note, idx) in purchaseNoteItems" :key="`${idx}-${note}`">{{ idx + 1 }}. {{ note }}</li>
              </ol>
            </div>
          </section>

          <div class="mt-10 grid gap-8 xl:grid-cols-[minmax(0,5fr)_minmax(320px,2fr)]">
            <section class="space-y-6">
              <div class="flex min-h-[84px] flex-col justify-end">
                <p class="text-sm font-semibold uppercase tracking-[0.28em] text-orange-500">{{ t('payment.plansLabel') }}</p>
                <h2 class="mt-2 text-3xl font-semibold tracking-tight text-slate-900 dark:text-white">{{ t('payment.plansTitle') }}</h2>
              </div>

              <div class="overflow-hidden rounded-[24px] border border-slate-200/90 bg-white/95 shadow-[0_14px_28px_rgba(15,23,42,0.04)] dark:border-dark-700 dark:bg-dark-800">
                <div v-if="tabs.length > 1" class="border-b border-slate-200 px-5 py-4 dark:border-dark-700">
                  <div class="inline-flex rounded-2xl bg-slate-100 p-1 dark:bg-dark-900">
                    <button
                      v-for="tab in tabs"
                      :key="tab.key"
                      type="button"
                      class="rounded-xl px-4 py-2 text-sm font-medium transition"
                      :class="activeTab === tab.key
                        ? 'bg-white text-slate-900 shadow-sm dark:bg-dark-700 dark:text-white'
                        : 'text-slate-500 hover:text-slate-700 dark:text-slate-400 dark:hover:text-slate-200'"
                      @click="activeTab = tab.key"
                    >
                      {{ tab.label }}
                    </button>
                  </div>
                </div>

                <div class="p-5">
                  <div v-if="activeTab === 'recharge' && !checkout.balance_disabled" class="space-y-5">
                    <div>
                      <p class="text-sm font-semibold text-slate-500 dark:text-slate-400">{{ t('payment.rechargeIntro') }}</p>
                      <h3 class="mt-2 text-2xl font-semibold tracking-tight text-slate-900 dark:text-white">{{ t('payment.topUpByAmount') }}</h3>
                      <p class="mt-3 text-sm leading-6 text-slate-600 dark:text-slate-300">{{ t('payment.topUpDescription') }}</p>
                    </div>
                    <div class="space-y-1 text-sm text-slate-700 dark:text-slate-300">
                      <p>{{ t('payment.exchangeRateLabel') }}: ¥1 = ${{ balanceRechargeMultiplier.toFixed(2) }}</p>
                      <p>{{ t('payment.actualCreditLabel') }}: ${{ creditedAmount.toFixed(2) }}</p>
                    </div>
                    <div class="flex flex-wrap gap-2.5">
                      <button v-for="quick in quickAmounts" :key="quick" type="button" class="rounded-xl border px-4 py-2 text-base font-semibold transition-colors" :class="amount === quick ? 'border-orange-300 bg-orange-50 text-orange-500 dark:border-orange-500/50 dark:bg-orange-500/10 dark:text-orange-300' : 'border-orange-100 bg-white text-slate-700 hover:border-orange-200 hover:text-orange-500 dark:border-dark-700 dark:bg-dark-900 dark:text-slate-300 dark:hover:border-orange-500/30'" @click="amount = quick">¥{{ quick.toFixed(2) }}</button>
                  </div>
                    <div class="grid gap-3 lg:grid-cols-[minmax(0,1fr)_160px]">
                      <input :value="amountInputValue" type="number" min="0" class="h-12 rounded-2xl border border-slate-200 bg-white px-4 text-base text-slate-900 outline-none transition focus:border-orange-300 dark:border-dark-700 dark:bg-dark-900 dark:text-white" :placeholder="t('payment.enterAmount')" @input="onAmountInput" />
                      <button type="button" class="inline-flex h-12 items-center justify-center rounded-2xl bg-orange-500 px-5 text-base font-semibold text-white shadow-[0_12px_24px_rgba(249,115,22,0.22)] transition hover:bg-orange-600 disabled:cursor-not-allowed disabled:bg-slate-300 disabled:shadow-none dark:disabled:bg-slate-700" :disabled="!canSubmit || submitting" @click="handleSubmitRecharge">
                        <span v-if="submitting" class="flex items-center justify-center gap-2"><span class="h-4 w-4 animate-spin rounded-full border-2 border-white border-t-transparent"></span>{{ t('common.processing') }}</span>
                        <span v-else>{{ t('payment.topUpNow') }}</span>
                      </button>
                    </div>
                    <div class="space-y-3">
                      <div v-if="enabledMethods.length > 0" class="space-y-2">
                        <p class="text-sm font-medium text-slate-600 dark:text-slate-300">{{ t('payment.paymentMethod') }}</p>
                        <div class="flex flex-wrap gap-2">
                          <button v-for="option in methodOptions" :key="option.type" type="button" class="rounded-full border px-3 py-1.5 text-sm font-medium transition-colors" :class="selectedMethod === option.type ? 'border-orange-300 bg-orange-50 text-orange-600 dark:border-orange-500/50 dark:bg-orange-500/10 dark:text-orange-300' : 'border-slate-200 bg-white text-slate-600 hover:border-orange-200 dark:border-dark-700 dark:bg-dark-900 dark:text-slate-300'" :disabled="!option.available" @click="selectedMethod = option.type">{{ t('payment.methods.' + option.type, option.type) }}</button>
                        </div>
                      </div>
                      <p v-if="amountError" class="text-sm text-amber-600 dark:text-amber-300">{{ amountError }}</p>
                      <p v-if="errorMessage && activeTab === 'recharge'" class="text-sm text-red-600 dark:text-red-400">{{ errorMessage }}</p>
                    </div>
                  </div>

                  <div v-if="activeTab === 'subscription'" class="space-y-4">
                    <div v-if="enabledMethods.length > 0" class="mb-2 space-y-2">
                      <p class="text-sm font-medium text-slate-600 dark:text-slate-300">{{ t('payment.paymentMethod') }}</p>
                      <div class="flex flex-wrap gap-2">
                        <button v-for="option in subscriptionMethodOptions" :key="option.type" type="button" class="rounded-full border px-3 py-1.5 text-sm font-medium transition-colors" :class="selectedMethod === option.type ? 'border-orange-300 bg-orange-50 text-orange-600 dark:border-orange-500/50 dark:bg-orange-500/10 dark:text-orange-300' : 'border-slate-200 bg-white text-slate-600 hover:border-orange-200 dark:border-dark-700 dark:bg-dark-900 dark:text-slate-300'" :disabled="!option.available" @click="selectedMethod = option.type">{{ t('payment.methods.' + option.type, option.type) }}</button>
                      </div>
                    </div>
                    <p v-if="errorMessage && activeTab === 'subscription'" class="text-sm text-red-600 dark:text-red-400">{{ errorMessage }}</p>
                  </div>

                  <div v-if="activeTab === 'subscription' && checkout.plans.length === 0" class="rounded-[20px] border border-dashed border-slate-300 bg-white/90 px-6 py-20 text-center dark:border-dark-700 dark:bg-dark-800/90">
                    <span class="inline-flex rounded-full border border-slate-200 px-3 py-1 text-xs font-medium text-slate-500 dark:border-dark-700 dark:text-slate-400">{{ t('payment.noPlansBadge') }}</span>
                    <h3 class="mt-5 text-3xl font-semibold tracking-tight text-slate-900 dark:text-white">{{ t('payment.noPlansTitle') }}</h3>
                    <p class="mt-3 text-base text-slate-500 dark:text-slate-400">{{ t('payment.noPlansDescription') }}</p>
                  </div>
                  <div v-else-if="activeTab === 'subscription'" :class="planGridClass">
                    <SubscriptionPlanCard v-for="plan in checkout.plans" :key="plan.id" :plan="plan" :active-subscriptions="activeSubscriptions" @select="selectPlan" />
                  </div>
                </div>
              </div>
            </section>

            <aside class="space-y-4 xl:sticky xl:top-6 xl:self-start">
              <div class="flex min-h-[84px] flex-col justify-end">
                <p class="text-sm font-semibold uppercase tracking-[0.28em] text-orange-500">{{ t('payment.accountLabel') }}</p>
                <h2 class="mt-2 text-3xl font-semibold tracking-tight text-slate-900 dark:text-white">{{ t('payment.accountCenter') }}</h2>
              </div>
              <div class="overflow-hidden rounded-[24px] border border-slate-200/90 bg-white/95 shadow-[0_14px_28px_rgba(15,23,42,0.04)] dark:border-dark-700 dark:bg-dark-800">
                <div class="border-b border-slate-200 px-5 py-4 dark:border-dark-700">
                  <div class="inline-flex rounded-2xl bg-slate-100 p-1 dark:bg-dark-900">
                    <button type="button" class="rounded-xl px-4 py-2 text-sm font-medium transition" :class="accountPanel === 'subscriptions' ? 'bg-white text-slate-900 shadow-sm dark:bg-dark-700 dark:text-white' : 'text-slate-500 hover:text-slate-700 dark:text-slate-400 dark:hover:text-slate-200'" @click="accountPanel = 'subscriptions'">{{ t('nav.mySubscriptions') }}</button>
                    <button type="button" class="rounded-xl px-4 py-2 text-sm font-medium transition" :class="accountPanel === 'orders' ? 'bg-white text-slate-900 shadow-sm dark:bg-dark-700 dark:text-white' : 'text-slate-500 hover:text-slate-700 dark:text-slate-400 dark:hover:text-slate-200'" @click="accountPanel = 'orders'">{{ t('nav.myOrders') }}</button>
                  </div>
                </div>
                <div class="p-5">
                  <div v-if="accountPanel === 'orders'" class="space-y-4">
                    <input v-model.trim="orderSearch" type="text" class="h-11 w-full rounded-2xl border border-slate-200 bg-white px-4 text-sm text-slate-900 outline-none transition focus:border-orange-300 dark:border-dark-700 dark:bg-dark-900 dark:text-white" :placeholder="t('payment.searchOrdersPlaceholder')" />
                    <div class="flex flex-wrap gap-2">
                      <button v-for="filter in orderFilters" :key="filter.value" type="button" class="rounded-full border px-3 py-1.5 text-sm font-medium transition-colors" :class="orderStatusFilter === filter.value ? 'border-orange-300 bg-orange-50 text-orange-600 dark:border-orange-500/50 dark:bg-orange-500/10 dark:text-orange-300' : 'border-slate-200 bg-white text-slate-500 hover:border-slate-300 dark:border-dark-700 dark:bg-dark-900 dark:text-slate-300'" @click="orderStatusFilter = filter.value">{{ filter.label }}</button>
                    </div>
                    <div v-if="recentOrdersLoading" class="flex justify-center py-10"><div class="h-6 w-6 animate-spin rounded-full border-2 border-primary-500 border-t-transparent"></div></div>
                    <div v-else-if="filteredOrders.length > 0" class="space-y-3">
                      <div v-for="order in filteredOrders" :key="order.id" class="rounded-[20px] border border-slate-200/90 bg-white p-4 dark:border-dark-700 dark:bg-dark-900">
                        <div class="flex items-start justify-between gap-3">
                          <div class="min-w-0">
                            <h4 class="truncate text-lg font-semibold text-slate-900 dark:text-white">{{ orderCardTitle(order) }}</h4>
                            <p class="mt-2 break-all text-sm text-slate-500 dark:text-slate-400">{{ t('payment.orders.orderNo') }}: {{ order.out_trade_no }}</p>
                          </div>
                          <OrderStatusBadge :status="order.status" />
                        </div>
                        <div class="mt-3 space-y-1.5 text-sm text-slate-600 dark:text-slate-300">
                          <p>{{ t('payment.orders.createdAt') }}: {{ formatDateTime(order.created_at) }}</p>
                          <p>{{ t('payment.orders.paymentMethod') }}: {{ t('payment.methods.' + order.payment_type, order.payment_type) }}</p>
                          <p>{{ t('payment.amountLabel') }}: ¥{{ order.amount.toFixed(2) }}</p>
                        </div>
                        <div class="mt-4 border-t border-slate-200 pt-4 dark:border-dark-700">
                          <div class="flex items-center justify-between gap-3">
                            <span class="text-base font-semibold text-slate-900 dark:text-white">{{ t('payment.actualPay') }}</span>
                            <span class="text-3xl font-semibold tracking-tight text-slate-900 dark:text-white">¥{{ order.pay_amount.toFixed(2) }}</span>
                          </div>
                        </div>
                      </div>
                    </div>
                    <div v-else class="rounded-[20px] border border-dashed border-slate-300 px-5 py-12 text-center text-slate-500 dark:border-dark-700 dark:text-slate-400">{{ t('payment.noRecentOrders') }}</div>
                  </div>
                  <div v-else class="space-y-4">
                    <div class="flex flex-wrap gap-2">
                      <button v-for="filter in subscriptionFilters" :key="filter.value" type="button" class="rounded-full border px-3 py-1.5 text-sm font-medium transition-colors" :class="subscriptionStatusFilter === filter.value ? 'border-orange-300 bg-orange-50 text-orange-600 dark:border-orange-500/50 dark:bg-orange-500/10 dark:text-orange-300' : 'border-slate-200 bg-white text-slate-500 hover:border-slate-300 dark:border-dark-700 dark:bg-dark-900 dark:text-slate-300'" @click="subscriptionStatusFilter = filter.value">{{ filter.label }}</button>
                    </div>
                    <div v-if="subscriptionsLoading" class="flex justify-center py-10"><div class="h-6 w-6 animate-spin rounded-full border-2 border-primary-500 border-t-transparent"></div></div>
                    <div v-else-if="filteredSubscriptions.length > 0" class="space-y-3">
                      <div v-for="sub in filteredSubscriptions" :key="sub.id" class="rounded-[20px] border border-slate-200/90 bg-white p-3.5 dark:border-dark-700 dark:bg-dark-900">
                        <div class="flex items-start justify-between gap-3">
                          <div class="min-w-0">
                            <div class="flex flex-wrap items-center gap-2">
                              <h4 class="truncate text-base font-semibold text-slate-900 dark:text-white">{{ sub.group?.name || `Group #${sub.group_id}` }}</h4>
                              <span :class="['rounded-full px-2 py-0.5 text-[10px] font-medium', platformBadgeLightClass(sub.group?.platform || '')]">{{ platformLabel(sub.group?.platform || '') }}</span>
                            </div>
                            <p class="mt-1.5 text-xs text-slate-500 dark:text-slate-400">{{ sub.expires_at ? formatDateTime(sub.expires_at) : t('userSubscriptions.noExpiration') }}</p>
                          </div>
                          <span class="shrink-0 whitespace-nowrap rounded-full px-2 py-0.5 text-[11px] font-medium leading-4" :class="sub.status === 'active' ? 'bg-emerald-50 text-emerald-600 dark:bg-emerald-500/10 dark:text-emerald-300' : 'bg-slate-100 text-slate-500 dark:bg-dark-700 dark:text-slate-300'">{{ t(`userSubscriptions.status.${sub.status}`) }}</span>
                        </div>
                        <div class="mt-3.5 flex items-center justify-between gap-3">
                          <p class="text-xs text-slate-600 dark:text-slate-300">
                            <template v-if="sub.expires_at">{{ t('userSubscriptions.daysRemaining', { days: getDaysRemaining(sub.expires_at) }) }}</template>
                            <template v-else>{{ t('userSubscriptions.noExpiration') }}</template>
                          </p>
                          <button v-if="sub.status === 'active'" type="button" :class="['rounded-xl px-3 py-1.5 text-[11px] font-semibold text-white transition-colors', platformButtonClass(sub.group?.platform || '')]" @click="openRenewal(sub.group_id)">{{ t('payment.renewNow') }}</button>
                        </div>
                      </div>
                    </div>
                    <div v-else class="rounded-[20px] border border-dashed border-slate-300 px-5 py-12 text-center text-slate-500 dark:border-dark-700 dark:text-slate-400">{{ t('payment.noSubscriptionForFilter') }}</div>
                  </div>
                </div>
              </div>
            </aside>
          </div>
        </template>
      </template>
    </div>

    <Teleport to="body">
      <Transition name="modal">
        <div v-if="showRenewalModal" class="fixed inset-0 z-50 flex items-center justify-center bg-black/60 backdrop-blur-sm p-4" @click.self="closeRenewalModal">
          <div class="relative w-full max-w-lg rounded-2xl border border-gray-200 bg-white p-6 shadow-2xl dark:border-dark-700 dark:bg-dark-900">
            <button class="absolute right-4 top-4 rounded-lg p-1 text-gray-400 transition-colors hover:bg-gray-100 hover:text-gray-600 dark:hover:bg-dark-700 dark:hover:text-gray-200" @click="closeRenewalModal">
              <svg class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="M6 18L18 6M6 6l12 12" /></svg>
            </button>
            <h3 class="mb-4 text-lg font-semibold text-gray-900 dark:text-white">{{ t('payment.selectPlan') }}</h3>
            <div class="space-y-4">
              <SubscriptionPlanCard v-for="plan in renewalPlans" :key="plan.id" :plan="plan" :active-subscriptions="activeSubscriptions" @select="selectPlanFromModal" />
            </div>
          </div>
        </div>
      </Transition>
    </Teleport>

    <Teleport to="body">
      <Transition name="modal">
        <div v-if="previewImage" class="fixed inset-0 z-[60] flex items-center justify-center bg-black/70 backdrop-blur-sm" @click="previewImage = ''">
          <img :src="previewImage" alt="" class="max-h-[85vh] max-w-[90vw] rounded-xl object-contain shadow-2xl" />
        </div>
      </Transition>
    </Teleport>
  </AppLayout>
</template>

<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { useRoute } from 'vue-router'
import { useAuthStore } from '@/stores/auth'
import { usePaymentStore } from '@/stores/payment'
import { useSubscriptionStore } from '@/stores/subscriptions'
import { useAppStore } from '@/stores'
import { paymentAPI } from '@/api/payment'
import subscriptionsAPI from '@/api/subscriptions'
import { extractApiErrorMessage } from '@/utils/apiError'
import { isMobileDevice } from '@/utils/device'
import type { UserSubscription } from '@/types'
import type { SubscriptionPlan, CheckoutInfoResponse, OrderType, PaymentOrder } from '@/types/payment'
import AppLayout from '@/components/layout/AppLayout.vue'
import SubscriptionPlanCard from '@/components/payment/SubscriptionPlanCard.vue'
import PaymentStatusPanel from '@/components/payment/PaymentStatusPanel.vue'
import StripePaymentInline from '@/components/payment/StripePaymentInline.vue'
import OrderStatusBadge from '@/components/payment/OrderStatusBadge.vue'
import { METHOD_ORDER, POPUP_WINDOW_FEATURES } from '@/components/payment/providerConfig'
import { platformBadgeLightClass, platformButtonClass, platformLabel } from '@/utils/platformColors'
import type { PaymentMethodOption } from '@/components/payment/PaymentMethodSelector.vue'
const { t } = useI18n()
const route = useRoute()
const authStore = useAuthStore()
const paymentStore = usePaymentStore()
const subscriptionStore = useSubscriptionStore()
const appStore = useAppStore()

const quickAmounts = [10, 20, 50, 100, 200, 500]
const user = computed(() => authStore.user)
const userIdentity = computed(() => user.value?.email || user.value?.username || t('common.notSet'))
const activeSubscriptions = computed(() => subscriptionStore.activeSubscriptions)

const loading = ref(true)
const recentOrdersLoading = ref(false)
const subscriptionsLoading = ref(false)
const submitting = ref(false)
const errorMessage = ref('')
const activeTab = ref<'recharge' | 'subscription'>('recharge')
const amount = ref<number | null>(null)
const selectedMethod = ref('')
const previewImage = ref('')
const accountPanel = ref<'subscriptions' | 'orders'>('subscriptions')
const orderStatusFilter = ref<'all' | 'PENDING' | 'COMPLETED' | 'REFUNDED'>('all')
const subscriptionStatusFilter = ref<'all' | 'active' | 'expired'>('active')
const orderSearch = ref('')
const recentOrders = ref<PaymentOrder[]>([])
const allSubscriptions = ref<UserSubscription[]>([])

const paymentPhase = ref<'select' | 'paying' | 'stripe'>('select')
const paymentState = ref<{ orderId: number; amount: number; qrCode: string; expiresAt: string; paymentType: string; payUrl: string; clientSecret: string; payAmount: number; orderType: OrderType | '' }>({ orderId: 0, amount: 0, qrCode: '', expiresAt: '', paymentType: '', payUrl: '', clientSecret: '', payAmount: 0, orderType: '' })
const checkout = ref<CheckoutInfoResponse>({ methods: {}, global_min: 0, global_max: 0, plans: [], balance_disabled: false, balance_recharge_multiplier: 1, recharge_fee_rate: 0, help_text: '', help_image_url: '', stripe_publishable_key: '' })

const enabledMethods = computed(() => Object.keys(checkout.value.methods))
const validAmount = computed(() => amount.value ?? 0)
const balanceRechargeMultiplier = computed(() => checkout.value.balance_recharge_multiplier > 0 ? checkout.value.balance_recharge_multiplier : 1)
const creditedAmount = computed(() => Math.round(validAmount.value * balanceRechargeMultiplier.value * 100) / 100)
const globalMinAmount = computed(() => checkout.value.global_min)
const selectedLimit = computed(() => checkout.value.methods[selectedMethod.value])
const amountInputValue = computed(() => amount.value ?? '')
const planGridClass = computed(() => checkout.value.plans.length <= 2 ? 'grid grid-cols-1 gap-5 lg:grid-cols-2' : 'grid grid-cols-1 gap-5 lg:grid-cols-2 2xl:grid-cols-3')

const purchaseNoteItems = computed(() => {
  const source = checkout.value.help_text?.trim() || t('payment.purchaseNotesFallback')
  return source.split(/\r?\n+/).map((item) => item.replace(/^\d+\.\s*/, '').trim()).filter(Boolean)
})
const tabs = computed(() => {
  const result: { key: 'recharge' | 'subscription'; label: string }[] = []
  if (!checkout.value.balance_disabled) result.push({ key: 'recharge', label: t('payment.tabTopUp') })
  result.push({ key: 'subscription', label: t('payment.tabSubscribe') })
  return result
})

function amountFitsMethod(amt: number, methodType: string): boolean {
  if (amt <= 0) return true
  const ml = checkout.value.methods[methodType]
  if (!ml) return false
  if (ml.single_min > 0 && amt < ml.single_min) return false
  if (ml.single_max > 0 && amt > ml.single_max) return false
  return true
}

const methodOptions = computed<PaymentMethodOption[]>(() => enabledMethods.value.map((type) => {
  const ml = checkout.value.methods[type]
  return { type, fee_rate: ml?.fee_rate ?? 0, available: ml?.available !== false && amountFitsMethod(validAmount.value, type) }
}))
const subscriptionMethodOptions = computed<PaymentMethodOption[]>(() => enabledMethods.value.map((type) => {
  const ml = checkout.value.methods[type]
  return { type, fee_rate: ml?.fee_rate ?? 0, available: ml?.available !== false }
}))
const amountError = computed(() => {
  if (validAmount.value <= 0) return ''
  if (!enabledMethods.value.some((m) => amountFitsMethod(validAmount.value, m))) return t('payment.amountNoMethod')
  const ml = selectedLimit.value
  if (ml) {
    if (ml.single_min > 0 && validAmount.value < ml.single_min) return t('payment.amountTooLow', { min: ml.single_min })
    if (ml.single_max > 0 && validAmount.value > ml.single_max) return t('payment.amountTooHigh', { max: ml.single_max })
  }
  return ''
})
const canSubmit = computed(() => validAmount.value > 0 && amountFitsMethod(validAmount.value, selectedMethod.value) && selectedLimit.value?.available !== false)
const orderFilters = computed(() => [
  { value: 'all' as const, label: t('common.all') },
  { value: 'PENDING' as const, label: t('payment.status.pending') },
  { value: 'COMPLETED' as const, label: t('payment.status.completed') },
  { value: 'REFUNDED' as const, label: t('payment.status.refunded') },
])
const activeSubscriptionCount = computed(() => allSubscriptions.value.filter((item) => item.status === 'active').length)
const expiredSubscriptionCount = computed(() => allSubscriptions.value.filter((item) => item.status === 'expired').length)
const subscriptionFilters = computed(() => [
  { value: 'active' as const, label: `${t('userSubscriptions.status.active')} (${activeSubscriptionCount.value})` },
  { value: 'expired' as const, label: `${t('userSubscriptions.status.expired')} (${expiredSubscriptionCount.value})` },
  { value: 'all' as const, label: t('common.all') },
])
const filteredOrders = computed(() => {
  const keyword = orderSearch.value.trim().toLowerCase()
  return recentOrders.value.filter((order) => {
    const matchStatus = orderStatusFilter.value === 'all' || order.status === orderStatusFilter.value
    const matchSearch = !keyword || String(order.id).includes(keyword) || order.out_trade_no.toLowerCase().includes(keyword) || order.payment_type.toLowerCase().includes(keyword)
    return matchStatus && matchSearch
  })
})
const filteredSubscriptions = computed(() => allSubscriptions.value.filter((sub) => subscriptionStatusFilter.value === 'all' || sub.status === subscriptionStatusFilter.value))

const showRenewalModal = ref(false)
const renewGroupId = ref<number | null>(null)
const renewalPlans = computed(() => renewGroupId.value == null ? [] : checkout.value.plans.filter((plan) => plan.group_id === renewGroupId.value))

function getDaysRemaining(expiresAt: string): number {
  const diff = new Date(expiresAt).getTime() - Date.now()
  return Math.max(0, Math.ceil(diff / (1000 * 60 * 60 * 24)))
}
function formatDateTime(dateStr?: string): string {
  if (!dateStr) return '-'
  return new Date(dateStr).toLocaleString()
}
function orderCardTitle(order: PaymentOrder): string {
  return order.order_type === 'balance'
    ? t('payment.balanceOrderTitle', { amount: order.amount.toFixed(2), credited: order.amount.toFixed(2) })
    : t('payment.subscriptionOrderTitle', { amount: order.amount.toFixed(2) })
}
function onAmountInput(event: Event) {
  const value = Number((event.target as HTMLInputElement).value)
  amount.value = Number.isFinite(value) && value > 0 ? value : null
}
function resetPayment() {
  paymentPhase.value = 'select'
  paymentState.value = { orderId: 0, amount: 0, qrCode: '', expiresAt: '', paymentType: '', payUrl: '', clientSecret: '', payAmount: 0, orderType: '' }
}
function onPaymentDone() {
  const wasSubscription = paymentState.value.orderType === 'subscription'
  resetPayment()
  loadRecentOrders().catch(() => {})
  if (wasSubscription) loadSubscriptions().catch(() => {})
}
function onPaymentSuccess() {
  authStore.refreshUser()
  loadRecentOrders().catch(() => {})
  loadSubscriptions().catch(() => {})
}
function onStripeDone() {
  const wasSubscription = paymentState.value.orderType === 'subscription'
  resetPayment()
  loadRecentOrders().catch(() => {})
  if (wasSubscription) loadSubscriptions().catch(() => {})
}
function onStripeRedirect(orderId: number, payUrl: string) {
  paymentState.value = { ...paymentState.value, orderId, payUrl, qrCode: '' }
  paymentPhase.value = 'paying'
}
function selectPlan(plan: SubscriptionPlan) {
  startSubscriptionCheckout(plan).catch(() => {})
}
async function selectPlanFromModal(plan: SubscriptionPlan) {
  showRenewalModal.value = false
  renewGroupId.value = null
  await startSubscriptionCheckout(plan)
}
function closeRenewalModal() {
  showRenewalModal.value = false
  renewGroupId.value = null
}
function getAvailableMethodForAmount(orderAmount: number): string {
  if (
    selectedMethod.value
    && checkout.value.methods[selectedMethod.value]?.available !== false
    && amountFitsMethod(orderAmount, selectedMethod.value)
  ) {
    return selectedMethod.value
  }

  return enabledMethods.value.find((type) => {
    const method = checkout.value.methods[type]
    return method?.available !== false && amountFitsMethod(orderAmount, type)
  }) || ''
}
async function startSubscriptionCheckout(plan: SubscriptionPlan) {
  activeTab.value = 'subscription'
  errorMessage.value = ''

  const method = getAvailableMethodForAmount(plan.price)
  if (!method) {
    errorMessage.value = t('payment.amountNoMethod')
    appStore.showError(errorMessage.value)
    return
  }

  selectedMethod.value = method
  await createOrder(plan.price, 'subscription', plan.id)
}
function openRenewal(groupId: number, options?: { directPay?: boolean }) {
  const directPay = options?.directPay ?? true
  activeTab.value = 'subscription'
  const groupPlans = checkout.value.plans.filter((plan) => plan.group_id === groupId)
  if (groupPlans.length === 1) {
    if (directPay) {
      startSubscriptionCheckout(groupPlans[0]).catch(() => {})
    }
  } else if (groupPlans.length > 1) {
    renewGroupId.value = groupId
    showRenewalModal.value = true
  }
}
async function loadRecentOrders() {
  recentOrdersLoading.value = true
  try {
    const res = await paymentAPI.getMyOrders({ page: 1, page_size: 8 })
    recentOrders.value = res.data.items || []
  } catch {
    recentOrders.value = []
  } finally {
    recentOrdersLoading.value = false
  }
}
async function loadSubscriptions() {
  subscriptionsLoading.value = true
  try {
    allSubscriptions.value = await subscriptionsAPI.getMySubscriptions()
    subscriptionStore.fetchActiveSubscriptions(true).catch(() => {})
  } catch {
    allSubscriptions.value = []
  } finally {
    subscriptionsLoading.value = false
  }
}
async function handleSubmitRecharge() {
  if (!canSubmit.value || submitting.value) return
  await createOrder(validAmount.value, 'balance')
}
async function createOrder(orderAmount: number, orderType: OrderType, planId?: number) {
  submitting.value = true
  errorMessage.value = ''
  try {
    const result = await paymentStore.createOrder({ amount: orderAmount, payment_type: selectedMethod.value, order_type: orderType, plan_id: planId })
    loadRecentOrders().catch(() => {})
    const openWindow = (url: string) => {
      const win = window.open(url, 'paymentPopup', POPUP_WINDOW_FEATURES)
      if (!win || win.closed) window.location.href = url
    }
    if (result.client_secret) {
      paymentState.value = { orderId: result.order_id, amount: result.amount, qrCode: '', expiresAt: result.expires_at || '', paymentType: selectedMethod.value, payUrl: '', clientSecret: result.client_secret, payAmount: result.pay_amount, orderType }
      paymentPhase.value = 'stripe'
    } else if (isMobileDevice() && result.pay_url) {
      paymentState.value = { orderId: result.order_id, amount: result.amount, qrCode: '', expiresAt: result.expires_at || '', paymentType: selectedMethod.value, payUrl: result.pay_url, clientSecret: '', payAmount: 0, orderType }
      paymentPhase.value = 'paying'
      window.location.href = result.pay_url
      return
    } else if (result.qr_code) {
      paymentState.value = { orderId: result.order_id, amount: result.amount, qrCode: result.qr_code, expiresAt: result.expires_at || '', paymentType: selectedMethod.value, payUrl: '', clientSecret: '', payAmount: 0, orderType }
      paymentPhase.value = 'paying'
    } else if (result.pay_url) {
      openWindow(result.pay_url)
      paymentState.value = { orderId: result.order_id, amount: result.amount, qrCode: '', expiresAt: result.expires_at || '', paymentType: selectedMethod.value, payUrl: result.pay_url, clientSecret: '', payAmount: 0, orderType }
      paymentPhase.value = 'paying'
    } else {
      errorMessage.value = t('payment.result.failed')
      appStore.showError(errorMessage.value)
    }
  } catch (err: unknown) {
    const apiErr = err as Record<string, unknown>
    if (apiErr.reason === 'TOO_MANY_PENDING') {
      const metadata = apiErr.metadata as Record<string, unknown> | undefined
      errorMessage.value = t('payment.errors.tooManyPending', { max: metadata?.max || '' })
    } else if (apiErr.reason === 'CANCEL_RATE_LIMITED') {
      errorMessage.value = t('payment.errors.cancelRateLimited')
    } else {
      errorMessage.value = extractApiErrorMessage(err, t('payment.result.failed'))
    }
    appStore.showError(errorMessage.value)
  } finally {
    submitting.value = false
  }
}
watch(() => [validAmount.value, selectedMethod.value] as const, ([amt, method]) => {
  if (amt <= 0 || amountFitsMethod(amt, method)) return
  const available = enabledMethods.value.find((m) => amountFitsMethod(amt, m))
  if (available) selectedMethod.value = available
})
onMounted(async () => {
  try {
    const res = await paymentAPI.getCheckoutInfo()
    checkout.value = res.data
    if (enabledMethods.value.length) {
      const sorted = [...enabledMethods.value].sort((a, b) => {
        const ai = METHOD_ORDER.indexOf(a as typeof METHOD_ORDER[number])
        const bi = METHOD_ORDER.indexOf(b as typeof METHOD_ORDER[number])
        return (ai === -1 ? 999 : ai) - (bi === -1 ? 999 : bi)
      })
      selectedMethod.value = sorted[0]
    }
    if (!checkout.value.balance_disabled) {
      const preferred = quickAmounts.find((m) => amountFitsMethod(m, selectedMethod.value))
      amount.value = preferred ?? (globalMinAmount.value > 0 ? globalMinAmount.value : 10)
    } else {
      activeTab.value = 'subscription'
    }
    if (route.query.tab === 'subscription' && route.query.group) {
      activeTab.value = 'subscription'
      openRenewal(Number(route.query.group), { directPay: false })
    }
  } catch (err: unknown) {
    appStore.showError(extractApiErrorMessage(err, t('common.error')))
  } finally {
    loading.value = false
  }
  loadRecentOrders().catch(() => {})
  loadSubscriptions().catch(() => {})
})
</script>
