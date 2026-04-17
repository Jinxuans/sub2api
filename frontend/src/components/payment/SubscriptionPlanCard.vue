<template>
  <div
    :class="[
      'group relative flex self-start flex-col overflow-hidden rounded-[22px] border border-slate-200/90 bg-white/95 shadow-[0_10px_24px_rgba(15,23,42,0.04)] transition-all',
      'hover:-translate-y-0.5 hover:shadow-[0_16px_30px_rgba(15,23,42,0.06)] dark:border-dark-700 dark:bg-dark-800',
    ]"
  >
    <div class="p-4">
      <div class="flex items-start justify-between gap-4">
        <div class="min-w-0 flex-1">
          <div class="flex flex-wrap items-center gap-2">
            <h3 class="text-[15px] font-semibold leading-5 tracking-tight text-slate-900 break-words dark:text-white" :title="plan.name">{{ plan.name }}</h3>
            <span :class="['shrink-0 rounded-full px-2 py-0.5 text-[10px] font-medium', badgeLightClass]">
              {{ pLabel }}
            </span>
          </div>
        </div>
        <div class="shrink-0 text-right">
          <div class="flex items-end justify-end gap-1">
            <span class="pb-1 text-[11px] text-slate-400 dark:text-slate-500">¥</span>
            <span class="text-[34px] leading-none font-semibold tracking-tight text-slate-900 dark:text-white">{{ plan.price }}</span>
          </div>
          <div class="mt-1 flex items-center justify-end gap-2">
            <span class="text-[11px] text-slate-400 dark:text-slate-500">/ {{ validitySuffix }}</span>
            <span v-if="discountText" class="rounded bg-orange-50 px-1.5 py-0.5 text-[10px] font-semibold text-orange-600 dark:bg-orange-500/10 dark:text-orange-300">{{ discountText }}</span>
          </div>
          <div v-if="plan.original_price" class="mt-0.5 flex items-center justify-end gap-1.5">
            <span class="text-[11px] text-slate-400 line-through dark:text-slate-500">¥{{ plan.original_price }}</span>
          </div>
        </div>
      </div>

      <p v-if="plan.description" class="mt-2 text-xs leading-5 text-slate-500 break-words dark:text-slate-400">
        {{ plan.description }}
      </p>

      <div v-if="statItems.length > 0" class="mt-3 rounded-[18px] border border-slate-200/80 bg-slate-50/75 p-3 dark:border-dark-700 dark:bg-dark-900">
        <div class="grid gap-2 sm:grid-cols-2">
          <div v-for="item in statItems" :key="item.label" class="rounded-xl bg-white/80 px-3 py-2 dark:bg-dark-800/80">
            <p class="text-[11px] leading-4 text-slate-400 dark:text-slate-500">{{ item.label }}</p>
            <p class="mt-1 text-sm font-medium text-slate-700 dark:text-slate-200">{{ item.value }}</p>
          </div>
        </div>
      </div>

      <div v-if="plan.features.length > 0" class="mt-3 space-y-1.5">
        <div v-for="feature in plan.features" :key="feature" class="flex items-start gap-1.5">
          <svg class="mt-0.5 h-3.5 w-3.5 flex-shrink-0 text-orange-500 dark:text-orange-300" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2.25">
            <path stroke-linecap="round" stroke-linejoin="round" d="M4.5 12.75l6 6 9-13.5" />
          </svg>
          <span class="text-xs leading-5 text-slate-600 dark:text-slate-300">{{ feature }}</span>
        </div>
      </div>

      <button
        type="button"
        class="mt-4 w-full rounded-2xl bg-orange-500 py-2.5 text-sm font-semibold text-white transition-all hover:bg-orange-600 active:scale-[0.98] dark:bg-orange-500 dark:hover:bg-orange-400"
        @click="emit('select', plan)"
      >
        {{ isRenewal ? t('payment.renewNow') : t('payment.subscribeNow') }}
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { useI18n } from 'vue-i18n'
import type { SubscriptionPlan } from '@/types/payment'
import type { UserSubscription } from '@/types'
import {
  platformBadgeLightClass,
  platformLabel,
} from '@/utils/platformColors'

const props = defineProps<{ plan: SubscriptionPlan; activeSubscriptions?: UserSubscription[] }>()
const emit = defineEmits<{ select: [plan: SubscriptionPlan] }>()
const { t } = useI18n()

const platform = computed(() => props.plan.group_platform || '')
const isRenewal = computed(() =>
  props.activeSubscriptions?.some(s => s.group_id === props.plan.group_id && s.status === 'active') ?? false
)

const badgeLightClass = computed(() => platformBadgeLightClass(platform.value))
const pLabel = computed(() => platformLabel(platform.value))

const discountText = computed(() => {
  if (!props.plan.original_price || props.plan.original_price <= 0) return ''
  const pct = Math.round((1 - props.plan.price / props.plan.original_price) * 100)
  return pct > 0 ? `-${pct}%` : ''
})

const rateDisplay = computed(() => {
  const rate = props.plan.rate_multiplier ?? 1
  return `×${Number(rate.toPrecision(10))}`
})

const statItems = computed(() => {
  const items = [
    { label: t('payment.planCard.rate'), value: rateDisplay.value },
    { label: t('payment.planDuration'), value: validitySuffix.value },
    { label: t('payment.planCard.dailyLimit'), value: props.plan.daily_limit_usd },
    { label: t('payment.planCard.weeklyLimit'), value: props.plan.weekly_limit_usd },
    { label: t('payment.planCard.monthlyLimit'), value: props.plan.monthly_limit_usd },
  ]

  return items
    .filter((item) => typeof item.value === 'string' || (typeof item.value === 'number' && item.value > 0))
    .map((item) => ({
      label: item.label,
      value: typeof item.value === 'number' ? `$${item.value}` : item.value,
    }))
})

const validitySuffix = computed(() => {
  const u = props.plan.validity_unit || 'day'
  if (u === 'month') return t('payment.perMonth')
  if (u === 'year') return t('payment.perYear')
  return `${props.plan.validity_days}${t('payment.days')}`
})
</script>
