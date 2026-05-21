<script setup>
import { ref, computed, watch } from 'vue'

const isDark = ref(false)

const toggleTheme = () => {
  isDark.value = !isDark.value
}

const monthlyKWh = ref(133380)
const superPeakRate = ref(11.46)
const peakRate = ref(7.11)
const unitCapacityKWh = ref(261)
const efficiency = ref(88)
const priceDiff = ref(0.8)
const priceDiff2 = ref(0.5)
const m2Months = ref(0)
const operatingDays = ref(330)
const degradationRate = ref(2)
const revenueShare = ref(80)

const equipmentRatePerWh = ref(0.8)
const installCostWan = ref(10)
const designCostWan = ref(5)
const opexRatePerWh = ref(0.01)
const loanRatio = ref(0)
const loanRate = ref(3)
const loanTerm = ref(10)
const discountRate = ref(8)

const calcParams = computed(() => ({
  monthlyKWh: monthlyKWh.value,
  superPeakRate: superPeakRate.value / 100,
  peakRate: peakRate.value / 100,
  unitCapacityKWh: unitCapacityKWh.value,
  efficiency: efficiency.value / 100,
  priceDiff: priceDiff.value,
  priceDiff2: priceDiff2.value,
  m2Months: m2Months.value,
  operatingDays: operatingDays.value,
  degradationRate: degradationRate.value / 100,
  revenueShare: revenueShare.value / 100,
  equipmentRatePerWh: equipmentRatePerWh.value,
  installCostWan: installCostWan.value,
  designCostWan: designCostWan.value,
  opexRatePerWh: opexRatePerWh.value,
  loanRatio: loanRatio.value / 100,
  loanRate: loanRate.value / 100,
  loanTerm: loanTerm.value,
  discountRate: discountRate.value / 100
}))

const calcResults = computed(() => {
  const p = calcParams.value
  
  const peakRatio = p.superPeakRate + p.peakRate
  const dailyPeakKWh = (p.monthlyKWh * peakRatio) / 30
  const dailyArbitrage = Math.min(dailyPeakKWh, p.unitCapacityKWh * p.efficiency)
  const requiredUnits = Math.ceil(dailyPeakKWh / (p.unitCapacityKWh * p.efficiency))
  const unitCount = Math.max(1, requiredUnits)
  
  const m1Months = 12 - p.m2Months
  const annualArbitrageM1 = dailyArbitrage * p.priceDiff * p.operatingDays * m1Months / 12
  const annualArbitrageM2 = dailyArbitrage * (p.priceDiff + p.priceDiff2) * p.operatingDays * p.m2Months / 12
  const grossBefore = (annualArbitrageM1 + annualArbitrageM2) / 10000
  
  const equipmentCost = unitCount * p.unitCapacityKWh * 1000 * p.equipmentRatePerWh / 10000
  const opexAnnual = unitCount * p.unitCapacityKWh * 1000 * p.opexRatePerWh / 10000
  const capex = equipmentCost + p.installCostWan + p.designCostWan
  
  const year1NetA = (grossBefore * p.revenueShare - opexAnnual)
  const dailyGross = grossBefore * 10000 / p.operatingDays
  const dailyGrossAfterA = year1NetA * 10000 / p.operatingDays
  const annualGrossB = grossBefore * (1 - p.revenueShare)
  const dailyGrossB = annualGrossB * 10000 / p.operatingDays
  
  const arrRate = year1NetA / capex * 100
  
  let cumulative = 0
  let paybackYear = 0
  for (let i = 1; i <= 20; i++) {
    const degradation = Math.pow(1 - p.degradationRate, i - 1)
    const yearNet = (grossBefore * degradation * p.revenueShare - opexAnnual)
    cumulative += yearNet
    if (cumulative >= capex && paybackYear === 0) {
      paybackYear = i
    }
  }
  const payback = paybackYear > 0 ? paybackYear : '-'

  let tenYearCum = 0
  for (let i = 1; i <= 10; i++) {
    const degradation = Math.pow(1 - p.degradationRate, i - 1)
    tenYearCum += (grossBefore * degradation * p.revenueShare - opexAnnual)
  }

    //银行借款比例   
  const loanAmount = capex * p.loanRatio
  const loanPayment = loanAmount > 0 ? (loanAmount * p.loanRate * Math.pow(1 + p.loanRate, p.loanTerm)) / (Math.pow(1 + p.loanRate, p.loanTerm) - 1) : 0
  
  let npv = -capex * (1 - p.loanRatio)
  for (let i = 1; i <= 10; i++) {
    const degradation = Math.pow(1 - p.degradationRate, i - 1)
    const fcf = (grossBefore * degradation * p.revenueShare - opexAnnual - loanPayment)
    npv += fcf / Math.pow(1 + p.discountRate, i)
  }
  
  const irr = calculateIRR(capex * (1 - p.loanRatio), grossBefore, p.revenueShare, opexAnnual, loanPayment, p.degradationRate, p.loanTerm)
  
  const roi10 = (tenYearCum / capex) * 100
  
  const coverageRate = (dailyArbitrage / dailyPeakKWh * 100)
  
  return {
    grossBefore: grossBefore.toFixed(2),
    dailyGross: dailyGross.toFixed(2),
    dailyGrossAfterA: dailyGrossAfterA.toFixed(2),
    year1NetA: year1NetA.toFixed(2),
    annualGrossB: annualGrossB.toFixed(2),
    dailyGrossB: dailyGrossB.toFixed(2),
    arrRate: arrRate.toFixed(1),
    unitCount: unitCount,
    dailyArbitrage: dailyArbitrage.toFixed(1),
    capex: capex.toFixed(2),
    equipmentCost: equipmentCost.toFixed(2),
    opexAnnual: opexAnnual.toFixed(2),
    payback: payback,
    tenYearCum: tenYearCum.toFixed(2),
    loanPayment: loanPayment.toFixed(2),
    npv: npv.toFixed(2),
    irr: irr.toFixed(2),
    roi10: roi10.toFixed(1),
    coverageRate: coverageRate.toFixed(1),
    m1Months: m1Months,
    m2Months: p.m2Months,
    year1GrossA: (grossBefore * p.revenueShare).toFixed(2),
    degradation10: ((1 - p.degradationRate) ** 9 * 100).toFixed(1)
  }
})

function calculateIRR(initialInvestment, grossBefore, share, opex, loanPayment, degradation, term) {
  let irr = 0.1
  let npv = -initialInvestment
  for (let i = 1; i <= term; i++) {
    const deg = Math.pow(1 - degradation, i - 1)
    const fcf = (grossBefore * deg * share - opex - loanPayment)
    npv += fcf / Math.pow(1 + irr, i)
  }
  
  if (npv > 0) {
    let high = 0.5, low = 0
    for (let i = 0; i < 50; i++) {
      irr = (low + high) / 2
      npv = -initialInvestment
      for (let j = 1; j <= term; j++) {
        const deg = Math.pow(1 - degradation, j - 1)
        const fcf = (grossBefore * deg * share - opex - loanPayment)
        npv += fcf / Math.pow(1 + irr, j)
      }
      if (npv > 0) low = irr
      else high = irr
    }
  }
  
  return irr * 100
}

const equipmentPercent = computed(() => {
  const capex = parseFloat(calcResults.value.capex)
  const equipment = parseFloat(calcResults.value.equipmentCost)
  return ((equipment / capex) * 100).toFixed(0)
})

const installPercent = computed(() => {
  const capex = parseFloat(calcResults.value.capex)
  return ((installCostWan.value / capex) * 100).toFixed(0)
})

const designPercent = computed(() => {
  const capex = parseFloat(calcResults.value.capex)
  return ((designCostWan.value / capex) * 100).toFixed(0)
})

const wfGrossAPercent = computed(() => {
  return (revenueShare.value).toFixed(0)
})

const wfNetAPercent = computed(() => {
  const gross = parseFloat(calcResults.value.grossBefore)
  const net = parseFloat(calcResults.value.year1NetA)
  return ((net / gross) * 100).toFixed(2)
})

const expandedSections = ref({
  sec1: true,
  sec2: true,
  sec3: true,
  sec4: true,
  sec5: true,
  sec6: true
})

const toggleSection = (sec) => {
  expandedSections.value[sec] = !expandedSections.value[sec]
}

const sensitivityData = computed(() => {
  const p = calcParams.value
  const shareValues = [90, 85, 80, 75, 70]
  const priceDiffValues = [0.5, 0.6, 0.7, 0.8, 0.9, 1.0, 1.1]
  
  return shareValues.map(share => ({
    share,
    rows: priceDiffValues.map(diff => {
      const dailyArbitrage = Math.min((p.monthlyKWh * (p.superPeakRate + p.peakRate)) / 30, p.unitCapacityKWh * p.efficiency)
      const annualArbitrage = dailyArbitrage * diff * p.operatingDays / 10000
      const value = (annualArbitrage * share / 100).toFixed(1)
      const isCurrent = share === p.revenueShare * 100 && diff === p.priceDiff
      return { diff, value, isCurrent }
    })
  }))
})

const getHeatmapColor = (value) => {
  const val = parseFloat(value)
  if (val < 8) return 'rgba(217,89,23,0.88)'
  if (val < 10) return 'rgba(239,125,20,0.88)'
  if (val < 12) return 'rgba(249,158,11,0.88)'
  if (val < 14) return 'rgba(217,179,38,0.88)'
  if (val < 16) return 'rgba(163,184,38,0.88)'
  if (val < 18) return 'rgba(107,185,69,0.88)'
  return 'rgba(16,185,129,0.88)'
}

const getHeatmapTextColor = (value) => {
  return parseFloat(value) < 11 ? '#fff' : '#0F1724'
}

const cashflowData = computed(() => {
  const p = calcParams.value
  const grossBefore = parseFloat(calcResults.value.grossBefore)
  const opex = parseFloat(calcResults.value.opexAnnual)
  const data = []
  let cumulative = -parseFloat(calcResults.value.capex)
  
  for (let i = 1; i <= 10; i++) {
    const degradation = Math.pow(1 - p.degradationRate, i - 1)
    const yearNet = grossBefore * degradation * p.revenueShare - opex
    cumulative += yearNet
    data.push({ year: i, net: yearNet.toFixed(2), cumulative: cumulative.toFixed(2) })
  }
  return data
})

watch(m2Months, (val) => {
  const inputs = document.querySelectorAll('.inp-field')
  const priceDiff2Input = document.querySelector('#inp_priceDiff2')
  const unitPriceDiff2 = document.querySelector('#unit_priceDiff2')
  const hintPriceDiff2 = document.querySelector('#hint_priceDiff2')
  
  if (val > 0) {
    priceDiff2Input?.removeAttribute('disabled')
    unitPriceDiff2?.classList.remove('disabled')
    hintPriceDiff2?.style.setProperty('color', 'var(--text3)')
  } else {
    priceDiff2Input?.setAttribute('disabled', 'disabled')
    unitPriceDiff2?.classList.add('disabled')
    hintPriceDiff2?.style.setProperty('color', 'var(--text3)')
  }
})
</script>

<template>
  <div :class="{ light: !isDark }">
    <div class="topbar">
      <div class="topbar-inner">
        <div class="topbar-icon">
          <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="#fff" stroke-width="2.5">
            <path d="M13 2L3 14h9l-1 8 10-12h-9l1-8z"></path>
          </svg>
        </div>
        <div>
          <div class="topbar-title">工商业储能投资测算器</div>
        </div>
        <button class="theme-btn" @click="toggleTheme">
          <svg v-if="!isDark" width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <circle cx="12" cy="12" r="5"></circle>
            <line x1="12" y1="1" x2="12" y2="3"></line>
            <line x1="12" y1="21" x2="12" y2="23"></line>
            <line x1="4.22" y1="4.22" x2="5.64" y2="5.64"></line>
            <line x1="18.36" y1="18.36" x2="19.78" y2="19.78"></line>
            <line x1="1" y1="12" x2="3" y2="12"></line>
            <line x1="21" y1="12" x2="23" y2="12"></line>
            <line x1="4.22" y1="19.78" x2="5.64" y2="18.36"></line>
            <line x1="18.36" y1="5.64" x2="19.78" y2="4.22"></line>
          </svg>
          <svg v-else width="15" height="15" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"></path>
          </svg>
        </button>
      </div>
    </div>

    <div class="scroll-body">
      <div class="summary-card">
        <div class="summary-role-hd">
          <div class="role-bar" style="background:linear-gradient(to bottom,var(--orange),var(--cyan));"></div>
          <div>
            <div class="role-title">项目总收益</div>
          </div>
        </div>
        <div class="summary-total">
          <div class="sh-row" style="margin-top:0;">
            <div class="sh-item">
              <div class="sh-item-l">年度总收益</div>
              <div class="sh-item-v-lg" style="color:var(--orange);font-size:19px;">{{ calcResults.grossBefore }} 万元/年</div>
            </div>
            <div class="sh-item" style="text-align:right;">
              <div class="sh-item-l">日均总收益</div>
              <div class="sh-item-v" style="color:var(--orange);">{{ calcResults.dailyGross }} 元/天</div>
            </div>
          </div>
        </div>

        <div class="summary-role-divider"></div>

        <div class="summary-role-hd">
          <div class="role-bar" style="background:linear-gradient(to bottom,var(--blue),var(--cyan));"></div>
          <div style="display:flex;align-items:baseline;gap:7px;">
            <div class="role-title">投资运营方净收益</div>
            <span class="role-share" style="color:var(--blue);">分成 {{ revenueShare }}%</span>
          </div>
        </div>
        <div class="summary-hero-b">
          <div class="sh-row" style="margin-top:0;">
            <div class="sh-item">
              <div class="sh-item-l">年度净收益</div>
              <div class="sh-val-inline">
                <div class="sh-item-v-lg" style="color:var(--blue);">{{ calcResults.year1NetA }} 万元/年</div>
                <div class="ary-badge" style="border-color: rgba(16, 185, 129, 0.35); background: rgba(16, 185, 129, 0.1);">
                  <span class="ary-badge-lbl" style="color: rgba(16, 185, 129, 0.7);">年化回报率</span>
                  <span style="color: var(--green);">{{ calcResults.arrRate }}%</span>
                </div>
              </div>
            </div>
            <div class="sh-item" style="text-align:right;">
              <div class="sh-item-l">日均净收益</div>
              <div class="sh-item-v" style="color:var(--blue);">{{ calcResults.dailyGrossAfterA }} 元/天</div>
              <div class="sh-item-l" style="margin-top:2px;">首年值，按 {{ degradationRate }}%/年衰减</div>
            </div>
          </div>
        </div>

        <div class="summary-grid">
          <div class="sg-cell sg-cell-blue">
            <div class="sg-name">推荐配置台数</div>
            <div class="sg-val" style="color: var(--blue);">{{ calcResults.unitCount }} 台</div>
            <div class="sg-sub">装机 {{ (calcResults.unitCount * unitCapacityKWh).toFixed(0) }} kWh · 日套利 {{ calcResults.dailyArbitrage }} kWh</div>
          </div>
          <div class="sg-cell sg-cell-cyan">
            <div class="sg-name">总投资 CAPEX</div>
            <div class="sg-val" style="color:var(--cyan);">{{ calcResults.capex }} 万元</div>
            <div class="sg-sub">设备 + 安装 + 设计</div>
          </div>
          <div class="sg-cell sg-cell-purple">
            <div class="sg-name">静态回收期</div>
            <div class="sg-val" style="color:var(--purple);">{{ calcResults.payback }} 年</div>
            <div class="sg-sub">全额投资口径</div>
          </div>
          <div class="sg-cell sg-cell-green">
            <div class="sg-name">10年累计净收益</div>
            <div class="sg-val" style="color:var(--green);">{{ calcResults.tenYearCum }} 万元</div>
          </div>
        </div>

        <div class="summary-role-divider"></div>

        <div class="summary-role-hd">
          <div class="role-bar" style="background:linear-gradient(to bottom,var(--green),var(--cyan));"></div>
          <div style="display:flex;align-items:baseline;gap:7px;">
            <div class="role-title">用能方净收益</div>
            <span class="role-share" style="color:var(--green);">分成 {{ (100 - revenueShare) }}%</span>
          </div>
        </div>
        <div class="summary-hero-b">
          <div class="sh-row" style="margin-top:0;">
            <div class="sh-item">
              <div class="sh-item-l">年度收益</div>
              <div class="sh-item-v-lg" style="color:var(--green);">{{ calcResults.annualGrossB }} 万元/年</div>
            </div>
            <div class="sh-item" style="text-align:right;">
              <div class="sh-item-l">日均收益</div>
              <div class="sh-item-v" style="color:var(--green);">{{ calcResults.dailyGrossB }} 元/天</div>
            </div>
          </div>
        </div>
      </div>

      <div class="desktop-cols">
        <div class="col-left">
          <div class="card">
            <div class="card-hd" @click="toggleSection('sec1')">
              <div class="card-hd-l">
                <div class="card-acc"></div>
                <span class="card-title">用电与套利参数</span>
              </div>
              <svg :class="['card-chev', { closed: !expandedSections.sec1 }]" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <polyline points="6 9 12 15 18 9"></polyline>
              </svg>
            </div>
            <div id="sec1" class="card-body" :style="{ maxHeight: expandedSections.sec1 ? '10000px' : '0px' }">
              <div class="input-sec">
                <div class="input-group">
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">月用电量</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">企业月度总用电量，可从电费账单中获取。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="monthlyKWh">
                      <div class="inp-unit">kWh</div>
                    </div>
                  </div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">尖峰用电占比</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">尖峰时段用电量占月总用电量的比例。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="superPeakRate" step="0.01">
                      <div class="inp-unit">%</div>
                    </div>
                  </div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">峰用电占比</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">峰时段用电量占月总用电量的比例，与尖峰合计决定可套利电量上限。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="peakRate" step="0.01">
                      <div class="inp-unit">%</div>
                    </div>
                  </div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">单台储能额定容量</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">单套储能系统的额定可用放电容量，由设备型号决定。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="unitCapacityKWh">
                      <div class="inp-unit">kWh</div>
                    </div>
                  </div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">充放电效率</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">一次完整充放电循环的综合效率，含PCS变流器损耗。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="efficiency" step="0.1">
                      <div class="inp-unit">%</div>
                    </div>
                    <div class="inp-hint">通常在 85%～92% 之间</div>
                  </div>

                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">充放电策略月份分配</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">拖动滑块分配全年12个月中两充两放的月数，剩余月份为一充一放。默认全年一充一放。</div>
                      </div>
                    </div>
                    <div class="ss-wrap">
                      <div class="ss-labels">
                        <span :class="['ss-lbl', { 'active-1c1d': m2Months === 0 }]">一充一放</span>
                        <span :class="['ss-lbl', { 'active-2c2d': m2Months > 0 }]">两充两放</span>
                      </div>
                      <div class="ss-track-outer">
                        <input type="range" class="ss-range" min="0" max="12" step="1" v-model.number="m2Months" style="background: var(--bar-track);">
                        <div class="ss-ticks">
                          <div v-for="i in 13" :key="i-1" class="ss-tick">
                            <div class="ss-tick-line"></div>
                            <div class="ss-tick-num">{{ i-1 }}</div>
                          </div>
                        </div>
                      </div>
                      <div class="ss-info" style="color: var(--cyan);">一充一放 {{ calcResults.m1Months }} 个月 · 两充两放 {{ calcResults.m2Months }} 个月</div>
                    </div>
                  </div>

                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">第一次充放电价差</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">当地峰时与谷时电价之差，是套利收益最核心的驱动因子。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="priceDiff" step="0.01">
                      <div class="inp-unit">元/kWh</div>
                    </div>
                    <div class="inp-hint">各地差异较大，通常 0.5～1.2 元/kWh</div>
                  </div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">第二次充放电价差</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">两充两放时第二次充放的峰谷价差，通常发生在次峰/平段，价差低于第一次。有两充两放月份时可编辑。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input id="inp_priceDiff2" class="inp-field" type="number" v-model.number="priceDiff2" step="0.01" :disabled="m2Months === 0">
                      <div id="unit_priceDiff2" :class="['inp-unit', { disabled: m2Months === 0 }]">元/kWh</div>
                    </div>
                    <div id="hint_priceDiff2" class="inp-hint">需先设置两充两放月份后方可编辑</div>
                  </div>

                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">年有效运行天数</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">扣除维保、节假日等因素后的年实际运行天数。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="operatingDays">
                      <div class="inp-unit">天</div>
                    </div>
                    <div class="inp-hint">建议 300～340 天</div>
                  </div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">年容量衰减率</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">电池每年容量自然衰减比例，影响逐年收益递减。磷酸铁锂通常1.5%~3%/年。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="degradationRate" step="0.1">
                      <div class="inp-unit">%/年</div>
                    </div>
                    <div class="inp-hint">磷酸铁锂通常 1.5%～3%/年</div>
                  </div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">投资方分成比例</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">投资方在项目套利总收益中的分成比例，剩余部分归业主。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="revenueShare" step="0.5">
                      <div class="inp-unit">%</div>
                    </div>
                    <div class="inp-hint">通常在 70%～90% 之间</div>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <div class="card">
            <div class="card-hd" @click="toggleSection('sec2')">
              <div class="card-hd-l">
                <div class="card-acc" style="background:linear-gradient(to bottom,var(--purple),var(--cyan));"></div>
                <span class="card-title">成本与融资参数</span>
              </div>
              <svg :class="['card-chev', { closed: !expandedSections.sec2 }]" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <polyline points="6 9 12 15 18 9"></polyline>
              </svg>
            </div>
            <div id="sec2" class="card-body" :style="{ maxHeight: expandedSections.sec2 ? '10000px' : '0px' }">
              <div class="input-sec">
                <div class="input-group">
                  <div class="inp-sub-label">设备与建设成本</div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">储能及配套设备单价</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">含电池包、PCS、BMS及配套电气柜等主设备，不含安装费用。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="equipmentRatePerWh" step="0.01">
                      <div class="inp-unit">元/Wh</div>
                    </div>
                    <div class="inp-hint">磷酸铁锂系统市场报价约 0.6～0.9 元/Wh</div>
                    <div class="inp-calc">设备费 ≈ {{ calcResults.equipmentCost }} 万元</div>
                  </div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">安装建设总费用</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">含现场土建、储能柜安装、电缆敷设、接地防雷、调试验收的全部费用。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="installCostWan" step="0.5">
                      <div class="inp-unit">万元</div>
                    </div>
                    <div class="inp-warn">⚠ 最低建议不低于 8 万元</div>
                  </div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">设计总费用</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">含电气方案设计、施工图出图、并网接入方案报批等全过程设计服务。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="designCostWan" step="0.5">
                      <div class="inp-unit">万元</div>
                    </div>
                  </div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">年度运维单价</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">含每年定期检修、电池容量测试、PCS维护及日常巡检人力成本。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="opexRatePerWh" step="0.001">
                      <div class="inp-unit">元/Wh·年</div>
                    </div>
                    <div class="inp-hint">约 0.008～0.015 元/Wh·年</div>
                    <div class="inp-calc">年运维费 ≈ {{ calcResults.opexAnnual }} 万元/年</div>
                  </div>
                  <div class="inp-div"></div>
                  <div class="inp-sub-label">财务融资参数</div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">银行借款比例</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">总投资(CAPEX)中通过银行贷款融资的比例，其余部分为自有资金。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="loanRatio" step="5">
                      <div class="inp-unit">%</div>
                    </div>
                    <div class="inp-hint">自有资金 = CAPEX × (1 - 借款比例)</div>
                  </div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">贷款年利率</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">银行借款的年化利率，按等额本息还款计算年度还款额。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="loanRate" step="0.1">
                      <div class="inp-unit">%</div>
                    </div>
                    <div class="inp-hint">当前市场参考利率约 3.5%～5.5%</div>
                  </div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">贷款期限</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">银行借款的还款年限，采用等额本息方式，每年还款额固定。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="loanTerm" step="1">
                      <div class="inp-unit">年</div>
                    </div>
                  </div>
                  <div class="inp-item">
                    <div class="inp-lbl-row">
                      <span class="inp-lbl">折现率（NPV）</span>
                      <div class="info-icon" tabindex="0">ℹ
                        <div class="info-tip">用于计算净现值(NPV)的折现率，代表资金的机会成本或最低期望回报率。</div>
                      </div>
                    </div>
                    <div class="inp-row">
                      <input class="inp-field" type="number" v-model.number="discountRate" step="0.5">
                      <div class="inp-unit">%</div>
                    </div>
                    <div class="inp-hint">通常取 6%～10%</div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div class="col-right">
          <div class="card">
            <div class="card-hd" @click="toggleSection('sec3')">
              <div class="card-hd-l">
                <div class="card-acc" style="background:linear-gradient(to bottom,var(--purple),var(--blue));"></div>
                <span class="card-title">投资构成与收益结构</span>
              </div>
              <svg :class="['card-chev', { closed: !expandedSections.sec3 }]" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <polyline points="6 9 12 15 18 9"></polyline>
              </svg>
            </div>
            <div id="sec3" class="card-body" :style="{ maxHeight: expandedSections.sec3 ? '10000px' : '0px' }">
              <div class="inv-sec">
                <div class="inv-top">
                  <span class="inv-title">初始投资构成</span>
                  <span class="capex-lbl">总投资 {{ calcResults.capex }} 万元</span>
                </div>
                <div class="inv-bar-row">
                  <div class="inv-bar-lbl">设备费</div>
                  <div class="inv-bar-track">
                    <div class="inv-bar-fill" style="background: var(--blue);" :style="{ width: equipmentPercent + '%' }">
                      <span class="inv-bar-val">{{ calcResults.equipmentCost }}万</span>
                    </div>
                  </div>
                  <div class="inv-bar-info">{{ calcResults.equipmentCost }} 万 · {{ equipmentPercent }}%</div>
                </div>
                <div class="inv-bar-row">
                  <div class="inv-bar-lbl">安装费</div>
                  <div class="inv-bar-track">
                    <div class="inv-bar-fill" style="background: var(--cyan);" :style="{ width: installPercent + '%' }">
                      <span class="inv-bar-val">{{ installCostWan }}万</span>
                    </div>
                  </div>
                  <div class="inv-bar-info">{{ installCostWan }} 万 · {{ installPercent }}%</div>
                </div>
                <div class="inv-bar-row">
                  <div class="inv-bar-lbl">设计费</div>
                  <div class="inv-bar-track">
                    <div class="inv-bar-fill" style="background: var(--purple);" :style="{ width: designPercent + '%' }">
                      <span class="inv-bar-val">{{ designCostWan }}万</span>
                    </div>
                  </div>
                  <div class="inv-bar-info">{{ designCostWan }} 万 · {{ designPercent }}%</div>
                </div>
              </div>

              <div class="wf-sec">
                <div class="wf-title">年度收益流向（首年）</div>
                <div class="wf-row">
                  <div class="wf-lbl">套利总收益</div>
                  <div class="wf-track">
                    <div class="wf-fill" style="background:linear-gradient(90deg,#D97706,var(--orange));width:100%;">
                      <span class="wfv">{{ calcResults.grossBefore }}万/年</span>
                    </div>
                  </div>
                </div>
                <div class="wf-arrow">↓ 用能方分成 <span style="color:var(--green);">{{ (100 - revenueShare) }}%</span> → <span style="color:var(--green);">{{ calcResults.annualGrossB }}万/年</span></div>
                <div class="wf-row">
                  <div class="wf-lbl">投资方毛收益</div>
                  <div class="wf-track">
                    <div class="wf-fill" style="background: linear-gradient(90deg,var(--blue),var(--cyan));" :style="{ width: wfGrossAPercent + '%' }">
                      <span class="wfv">{{ calcResults.year1GrossA }}万/年</span>
                    </div>
                  </div>
                </div>
                <div class="wf-arrow">↓ 运维成本 → <span style="color:var(--red);">{{ calcResults.opexAnnual }}万/年</span></div>
                <div class="wf-row">
                  <div class="wf-lbl">投资方净收益</div>
                  <div class="wf-track">
                    <div class="wf-fill" style="background: linear-gradient(90deg,#065F46,var(--green));" :style="{ width: wfNetAPercent + '%' }">
                      <span class="wfv">{{ calcResults.year1NetA }}万/年</span>
                    </div>
                  </div>
                </div>
                <div v-if="loanRatio > 0" style="display:block;">
                  <div class="wf-arrow">↓ 年还款额 → <span style="color:var(--orange);">{{ calcResults.loanPayment }}万/年</span></div>
                  <div class="wf-row">
                    <div class="wf-lbl">自由现金流</div>
                    <div class="wf-track">
                      <div class="wf-fill" style="background:linear-gradient(90deg,#1e3a5f,var(--blue));" :style="{ width: ((parseFloat(calcResults.year1NetA) - parseFloat(calcResults.loanPayment)) / parseFloat(calcResults.grossBefore) * 100).toFixed(2) + '%' }">
                        <span class="wfv">{{ (parseFloat(calcResults.year1NetA) - parseFloat(calcResults.loanPayment)).toFixed(2) }}万/年</span>
                      </div>
                    </div>
                  </div>
                </div>
                <div class="bar-notes">
                  <div class="bar-note">策略：<span>一充一放 {{ calcResults.m1Months }}月{{ calcResults.m2Months > 0 ? ' · 两充两放 ' + calcResults.m2Months + '月' : '' }}</span></div>
                  <div class="bar-note">套利覆盖率：<span>{{ calcResults.coverageRate }}</span>%</div>
                  <div class="bar-note">容量衰减：<span>{{ degradationRate }}</span>%/年，第10年约首年的 <span>{{ calcResults.degradation10 }}</span>%</div>
                  <div class="bar-note" style="width:100%;color:var(--text3);">以上为首年数据，收益逐年按衰减率递减</div>
                </div>
              </div>
            </div>
          </div>

          <div class="card">
            <div class="card-hd" @click="toggleSection('sec4')">
              <div class="card-hd-l">
                <div class="card-acc" style="background:linear-gradient(to bottom,var(--green),var(--blue));"></div>
                <span class="card-title">财务指标分析</span>
              </div>
              <svg :class="['card-chev', { closed: !expandedSections.sec4 }]" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <polyline points="6 9 12 15 18 9"></polyline>
              </svg>
            </div>
            <div id="sec4" class="card-body" :style="{ maxHeight: expandedSections.sec4 ? '10000px' : '0px' }">
              <div class="fin-grid">
                <div class="fin-card o">
                  <div class="fin-c-lbl">投资回报率 ROI（10年）</div>
                  <div class="fin-c-val" style="color:var(--orange);">{{ calcResults.roi10 }}%</div>
                  <div class="fin-c-sub">10年净收益合计 ÷ CAPEX</div>
                </div>
                <div class="fin-card b">
                  <div class="fin-c-lbl">内部收益率 IRR（10年）</div>
                  <div class="fin-c-val" style="color:var(--blue);">{{ calcResults.irr }}%</div>
                  <div class="fin-c-sub">使NPV=0的折现率，自有资金口径</div>
                </div>
                <div class="fin-card p">
                  <div class="fin-c-lbl">净现值 NPV（10年）</div>
                  <div class="fin-c-val" style="color:var(--purple);">{{ calcResults.npv }} 万元</div>
                  <div class="fin-c-sub">折现率 {{ discountRate }}%，自有资金口径</div>
                </div>
                <div class="fin-card r">
                  <div class="fin-c-lbl">年均还款额</div>
                  <div class="fin-c-val" style="color:var(--red);">{{ calcResults.loanPayment }} 万/年</div>
                  <div class="fin-c-sub">等额本息，还款期 {{ loanTerm }} 年</div>
                </div>
              </div>
              <div class="table-wrap">
                <table class="fin-table">
                  <thead>
                    <tr>
                      <th>指标</th>
                      <th style="text-align:right;">无杠杆</th>
                      <th style="text-align:right;">有杠杆</th>
                      <th style="text-align:right;">说明</th>
                    </tr>
                  </thead>
                  <tbody>
                    <tr>
                      <td>自有资金投入</td>
                      <td class="val">{{ calcResults.capex }}万</td>
                      <td class="val" v-if="loanRatio > 0">{{ (parseFloat(calcResults.capex) * (1 - loanRatio / 100)).toFixed(2) }}万</td>
                      <td class="val" v-else><span style="color:var(--text3);">—</span></td>
                      <td style="color:var(--text3);font-size:10px;">项目初始资本支出，有杠杆时为股权投入部分</td>
                    </tr>
                    <tr>
                      <td>年度自由现金流</td>
                      <td class="val pos">{{ calcResults.year1NetA }}万/年</td>
                      <td class="val" v-if="loanRatio > 0">{{ (parseFloat(calcResults.year1NetA) - parseFloat(calcResults.loanPayment)).toFixed(2) }}万/年</td>
                      <td class="val" v-else><span style="color:var(--text3);">—</span></td>
                      <td style="color:var(--text3);font-size:10px;">扣除运维及债务偿还后的年度可支配现金</td>
                    </tr>
                    <tr>
                      <td>静态回收期</td>
                      <td class="val warn">{{ calcResults.payback }}年</td>
                      <td class="val warn" v-if="loanRatio > 0">{{ calcResults.payback }}年</td>
                      <td class="val warn" v-else>—</td>
                      <td style="color:var(--text3);font-size:10px;">全额投资口径，基于年度净现金流的简单回收年限</td>
                    </tr>
                    <tr>
                      <td>自有资金回收期</td>
                      <td class="val" style="color:var(--text3);">—</td>
                      <td class="val warn" v-if="loanRatio > 0">{{ calcResults.payback }}年</td>
                      <td class="val warn" v-else><span style="color:var(--text3);">—</span></td>
                      <td style="color:var(--text3);font-size:10px;">自有资金口径，含债务偿还成本的实际资金回收年限</td>
                    </tr>
                    <tr>
                      <td>10年累计净收益</td>
                      <td class="val pos">{{ calcResults.tenYearCum }}万</td>
                      <td class="val" v-if="loanRatio > 0">{{ (parseFloat(calcResults.tenYearCum) - parseFloat(calcResults.loanPayment) * loanTerm).toFixed(2) }}万</td>
                      <td class="val" v-else><span style="color:var(--text3);">—</span></td>
                      <td style="color:var(--text3);font-size:10px;">10年净现金流累计，有杠杆列扣除期末未偿本金</td>
                    </tr>
                    <tr>
                      <td>贷款利息合计</td>
                      <td class="val" style="color:var(--text3);">—</td>
                      <td class="val neg" v-if="loanRatio > 0">{{ ((parseFloat(calcResults.loanPayment) * loanTerm) - (parseFloat(calcResults.capex) * loanRatio / 100)).toFixed(2) }}万</td>
                      <td class="val neg" v-else><span style="color:var(--text3);">—</span></td>
                      <td style="color:var(--text3);font-size:10px;">等额本息还款模式下的融资总利息支出</td>
                    </tr>
                  </tbody>
                </table>
              </div>
              <div class="dscr-sec">
                <div class="dscr-hd">
                  <span class="dscr-title">DSCR 逐年债务覆盖率</span>
                  <span class="dscr-sub">每年收益覆盖还款的倍数，低于 1.0 表示当年现金流不足以偿债</span>
                </div>
                <div v-if="loanRatio === 0" class="dscr-no-loan" style="display: flex;">
                  <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" opacity=".4">
                    <rect x="2" y="7" width="20" height="14" rx="2"></rect>
                    <path d="M16 7V5a2 2 0 00-4 0v2"></path>
                  </svg>
                  <span>未设置贷款，DSCR 不适用</span>
                </div>
              </div>
            </div>
          </div>

          <div class="card">
            <div class="card-hd" @click="toggleSection('sec5')">
              <div class="card-hd-l">
                <div class="card-acc" style="background:linear-gradient(to bottom,var(--orange),var(--red));"></div>
                <span class="card-title">敏感性分析</span>
              </div>
              <svg :class="['card-chev', { closed: !expandedSections.sec5 }]" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <polyline points="6 9 12 15 18 9"></polyline>
              </svg>
            </div>
            <div id="sec5" class="card-body" :style="{ maxHeight: expandedSections.sec5 ? '10000px' : '0px' }">
              <div class="hm-sec">
                <div class="hm-title">年度净收益（分成后，万元）</div>
                <div class="hm-sub">调整第一次充放电价差与分成比例，观察净收益变化｜高亮边框为当前参数位置</div>
                <div class="hm-leg-row">
                  <span class="hm-leg-lbl">低</span>
                  <div class="hm-grad-bar"></div>
                  <span class="hm-leg-lbl">高</span>
                </div>
                <div class="hm-scroll">
                  <table class="hm-table">
                    <thead>
                      <tr>
                        <th style="text-align:right;padding-right:6px;font-size:9px;">分成\价差①</th>
                        <th v-for="pd in [0.5, 0.6, 0.7, 0.8, 0.9, 1.0, 1.1]" :key="pd">{{ pd }}<br><span style="font-size:9px;opacity:.6;">元/kWh</span></th>
                      </tr>
                    </thead>
                    <tbody>
                      <tr v-for="row in sensitivityData" :key="row.share">
                        <td style="text-align:right;padding-right:6px;background:transparent;color:var(--text3);font-size:10px;border:none;">{{ row.share }}%</td>
                        <td v-for="cell in row.rows" :key="cell.diff" :class="{ cur: cell.isCurrent }" :style="{ backgroundColor: getHeatmapColor(cell.value), color: getHeatmapTextColor(cell.value) }">
                          {{ cell.value }}
                          <span v-if="cell.isCurrent" class="cur-dot"></span>
                        </td>
                      </tr>
                    </tbody>
                  </table>
                </div>
              </div>
            </div>
          </div>

          <div class="card">
            <div class="card-hd" @click="toggleSection('sec6')">
              <div class="card-hd-l">
                <div class="card-acc" style="background:linear-gradient(to bottom,var(--cyan),var(--blue));"></div>
                <span class="card-title">投资回收模型</span>
              </div>
              <svg :class="['card-chev', { closed: !expandedSections.sec6 }]" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <polyline points="6 9 12 15 18 9"></polyline>
              </svg>
            </div>
            <div id="sec6" class="card-body" :style="{ maxHeight: expandedSections.sec6 ? '10000px' : '0px' }">
              <div class="cf-sec">
                <div class="cf-top">
                  <div>
                    <div class="cf-sub">年度净收益（分成后）累计 · 10年投资回收轨迹</div>
                  </div>
                </div>
                <div class="cf-canvas-wrap" style="height: 200px;">
                  <svg width="100%" height="100%" viewBox="0 0 600 180" preserveAspectRatio="none">
                    <defs>
                      <linearGradient id="areaGradient" x1="0%" y1="0%" x2="0%" y2="100%">
                        <stop offset="0%" style="stop-color:rgba(239,68,68,0.3)"/>
                        <stop offset="50%" style="stop-color:rgba(239,68,68,0.1)"/>
                        <stop offset="50%" style="stop-color:rgba(59,130,246,0.1)"/>
                        <stop offset="100%" style="stop-color:rgba(59,130,246,0.3)"/>
                      </linearGradient>
                    </defs>
                    <g transform="translate(50, 10)">
                      <line x1="0" y1="160" x2="550" y2="160" stroke="var(--border)" stroke-width="1"/>
                      <line x1="0" y1="0" x2="0" y2="160" stroke="var(--border)" stroke-width="1"/>
                      
                      <g v-for="i in 11" :key="'grid-h-'+i">
                        <line x1="0" :y1="(i-1) * 16" x2="550" :y2="(i-1) * 16" stroke="var(--border2)" stroke-width="0.5" stroke-dasharray="2,2"/>
                      </g>
                      
                      <g v-for="(item, idx) in cashflowData" :key="'bar-'+idx">
                        <rect :x="idx * 50 + 5" :y="160 - (parseFloat(item.cumulative) / 100 * 160)" width="40" :height="Math.abs(parseFloat(item.cumulative) / 100 * 160)" :fill="parseFloat(item.cumulative) >= 0 ? 'rgba(59,130,246,0.45)' : 'rgba(239,68,68,0.45)'"/>
                        <text :x="idx * 50 + 25" y="175" text-anchor="middle" font-size="10" fill="var(--text3)" font-family="Exo 2">{{ item.year }}</text>
                      </g>
                      
                      <line x1="0" y1="80" x2="550" y2="80" stroke="var(--cyan)" stroke-width="1" stroke-dasharray="4,2" opacity="0.7"/>
                      <text x="560" y="83" font-size="9" fill="var(--text3)">0</text>
                    </g>
                  </svg>
                </div>
                <div class="cf-legend">
                  <div class="cf-leg">
                    <div class="cf-leg-box" style="background:rgba(239,68,68,0.45);"></div>
                    未回收
                  </div>
                  <div class="cf-leg">
                    <div class="cf-leg-box" style="background:rgba(59,130,246,0.45);"></div>
                    盈利
                  </div>
                  <div class="cf-pb-row">
                    <div style="width:16px;height:0;border-top:2px dashed rgba(150,190,230,0.7);"></div>
                    回收期约 <strong style="color:var(--cyan);margin-left:3px;">{{ calcResults.payback }}</strong> 年
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="footer">
        <div class="footer-main">© 2026 Tian · 工商业储能投资测算器</div>
        <div class="footer-cta">关注公众号获取更多免费工具</div>
        <div class="footer-contact-row">
          <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8">
            <path d="M21 11.5a8.38 8.38 0 0 1-.9 3.8 8.5 8.5 0 0 1-7.6 4.7 8.38 8.38 0 0 1-3.8-.9L3 21l1.9-5.7a8.38 8.38 0 0 1-.9-3.8 8.5 8.5 0 0 1 4.7-7.6 8.38 8.38 0 0 1 3.8-.9h.5a8.48 8.48 0 0 1 8 8v.5z"></path>
          </svg>
          <span>硅碳Alpha</span>
          <span class="footer-contact-id-sep">|</span>
          <span>ID: xSiC_Alpha</span>
          <div class="footer-div"></div>
          <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8">
            <path d="M4 4h16c1.1 0 2 .9 2 2v12c0 1.1-.9 2-2 2H4c-1.1 0-2-.9-2-2V6c0-1.1.9-2 2-2z"></path>
            <polyline points="22,6 12,13 2,6"></polyline>
          </svg>
          <a href="mailto:zy.tian@outlook.com">zy.tian@outlook.com</a>
        </div>
        <div class="footer-disclaimer">本工具仅供参考，测算结果不构成投资建议，请结合实际情况审慎决策</div>
        <div class="footer-date">Released 2026.04.01</div>
      </div>
    </div>
  </div>
</template>

<style>
:root {
  --bg:#0B1120;--bg2:#0E1628;--card:#14213A;--card2:#182845;
  --border:#253D62;--border2:#1E3050;
  --blue:#3B82F6;--cyan:#06B6D4;--purple:#8B5CF6;
  --green:#10B981;--red:#EF4444;--orange:#F59E0B;
  --text:#EDF2FF;--text2:#ABBDD6;--text3:#7898BC;
  --input-bg:rgba(10,17,38,0.85);--unit-bg:rgba(35,55,88,0.75);
  --tip-bg:#111E35;--bar-track:rgba(25,38,65,0.8);
  --fin-bg:rgba(14,22,42,0.9);--fin-border:rgba(59,130,246,0.22);
  --header-h:56px;
  --content-max:1240px;
  --col-left-w:420px;
}
body.light {
  --bg:#EEF3FF;--bg2:#E2EAF8;--card:#FFFFFF;--card2:#F4F8FF;
  --border:#C4D2EC;--border2:#D0DCF2;
  --text:#0F1828;--text2:#2E456E;--text3:#6278A0;
  --input-bg:rgba(240,246,255,0.9);--unit-bg:rgba(195,215,245,0.6);
  --tip-bg:#FFFFFF;--bar-track:rgba(180,200,235,0.35);
  --fin-bg:rgba(230,240,255,0.7);--fin-border:rgba(59,130,246,0.22);
}
*{box-sizing:border-box;margin:0;padding:0;-webkit-tap-highlight-color:transparent;}
html,body{height:100%;overflow-x:hidden;}
body{font-family:'Noto Sans SC',sans-serif;background:var(--bg);color:var(--text);font-size:14px;transition:background .3s,color .3s;}
::-webkit-scrollbar{width:4px;height:4px;}
::-webkit-scrollbar-track{background:var(--bg2);}
::-webkit-scrollbar-thumb{background:var(--border);border-radius:2px;}

.topbar{position:fixed;top:0;left:0;right:0;height:var(--header-h);z-index:100;
  background:rgba(11,17,32,0.97);border-bottom:1px solid var(--border);
  backdrop-filter:blur(12px);}
body.light .topbar{background:rgba(238,243,255,0.96);}
.topbar-inner{max-width:var(--content-max);margin:0 auto;height:100%;
  display:flex;align-items:center;padding:0 14px;gap:10px;}
.topbar-icon{width:34px;height:34px;background:linear-gradient(135deg,#1D3B7A,var(--cyan));
  border-radius:8px;display:flex;align-items:center;justify-content:center;flex-shrink:0;}
.topbar-title{font-family:'Exo 2',sans-serif;font-size:15px;font-weight:800;
  background:linear-gradient(90deg,var(--blue),var(--cyan));
  -webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text;flex:1;}
.theme-btn{width:32px;height:32px;border-radius:50%;border:1px solid var(--border);
  background:var(--card2);color:var(--text2);cursor:pointer;
  display:flex;align-items:center;justify-content:center;flex-shrink:0;transition:all .2s;}

.scroll-body{
  max-width:var(--content-max);
  margin:0 auto;
  padding:calc(var(--header-h) + 12px) 12px 24px;
  display:flex;flex-direction:column;gap:12px;
}

.desktop-cols{display:flex;flex-direction:column;gap:12px;width:100%;}
.col-left{display:flex;flex-direction:column;gap:12px;}
.col-right{display:flex;flex-direction:column;gap:12px;}

.card{background:var(--card);border:1px solid var(--border);border-radius:14px;overflow:hidden;
  transition:background .3s,border-color .3s;}
.card-hd{padding:12px 14px;display:flex;align-items:center;justify-content:space-between;
  border-bottom:1px solid var(--border2);cursor:pointer;user-select:none;}
.card-hd-l{display:flex;align-items:center;gap:8px;}
.card-acc{width:3px;height:16px;border-radius:2px;background:linear-gradient(to bottom,var(--blue),var(--cyan));}
.card-title{font-family:'Exo 2',sans-serif;font-size:13px;font-weight:700;color:var(--text);}
.card-chev{transition:transform .28s;color:var(--text3);flex-shrink:0;}
.card-chev.closed{transform:rotate(-90deg);}
.card-body{overflow:hidden;transition:max-height .4s ease;}

.summary-card{background:var(--card);border:1px solid var(--border);border-radius:14px;overflow:hidden;}
.summary-role-hd{display:flex;align-items:center;gap:8px;padding:8px 14px 0;}
.role-bar{width:4px;height:24px;border-radius:2px;flex-shrink:0;}
.role-title{font-family:'Exo 2',sans-serif;font-size:13px;font-weight:800;color:var(--text);letter-spacing:0.3px;}
.role-sub{font-size:10px;color:var(--text3);margin-top:1px;}
.role-share{font-size:11px;font-weight:400;opacity:0.65;font-family:'Exo 2',sans-serif;}
.summary-role-divider{height:1px;background:var(--border);margin:8px 0 0;}
.summary-total{padding:5px 14px 8px;}
.summary-hero-b{padding:5px 14px 10px;}
.sh-item-v-lg{font-family:'Exo 2',sans-serif;font-size:16px;font-weight:800;line-height:1.2;}
.sh-item-mid{display:flex;align-items:center;justify-content:center;flex:1;}
.sh-b-note{font-size:10px;color:var(--text3);margin-top:5px;padding:3px 9px;
  background:rgba(100,116,139,0.06);border:1px solid var(--border2);border-radius:6px;}
.sh-row{display:flex;align-items:center;justify-content:space-between;margin-top:5px;gap:8px;}
.sh-item{display:flex;flex-direction:column;gap:1px;}
.sh-item-l{font-size:10px;color:var(--text3);}
.sh-item-v{font-family:'Exo 2',sans-serif;font-size:12px;font-weight:700;color:var(--text2);}
.sh-val-inline{display:flex;align-items:center;gap:7px;flex-wrap:wrap;}
.ary-badge{display:inline-flex;align-items:center;gap:4px;background:rgba(59,130,246,0.1);
  border:1px solid rgba(59,130,246,0.3);border-radius:20px;padding:2px 8px;
  font-family:'Exo 2',sans-serif;font-size:12px;font-weight:700;color:var(--blue);
  white-space:nowrap;align-self:center;}
.ary-badge-lbl{font-size:9px;color:var(--text3);font-family:'Noto Sans SC',sans-serif;font-weight:400;}
.summary-grid{display:grid;grid-template-columns:1fr 1fr;gap:0;}
.sg-cell{padding:8px 12px;border-right:1px solid var(--border2);border-bottom:1px solid var(--border2);position:relative;}
.sg-cell:nth-child(2n){border-right:none;}
.sg-cell:nth-last-child(-n+2){border-bottom:none;}
.sg-cell-blue{background:rgba(59,130,246,0.05);}
.sg-cell-cyan{background:rgba(6,182,212,0.05);}
.sg-cell-purple{background:rgba(139,92,246,0.05);}
.sg-cell-green{background:rgba(16,185,129,0.05);}
.sg-name{font-size:10px;color:var(--text3);margin-bottom:1px;}
.sg-val{font-family:'Exo 2',sans-serif;font-size:15px;font-weight:800;color:var(--text);line-height:1.2;margin-bottom:2px;}
.sg-sub{font-size:9px;color:var(--text3);line-height:1.5;white-space:normal;}

.input-sec{padding:14px;}
.input-group{display:flex;flex-direction:column;gap:11px;}
.inp-item{display:flex;flex-direction:column;gap:5px;}
.inp-lbl-row{display:flex;align-items:center;gap:6px;}
.inp-lbl{font-size:12px;font-weight:500;color:var(--text2);}
.inp-row{display:flex;align-items:stretch;gap:0;}
.inp-field{flex:1;background:var(--input-bg);border:1px solid var(--border);
  border-right:none;border-radius:8px 0 0 8px;padding:11px 12px;
  font-size:15px;font-family:'Exo 2',sans-serif;color:var(--text);
  outline:none;transition:border-color .2s,opacity .2s;-webkit-appearance:none;min-height:44px;}
.inp-field:focus{border-color:var(--blue);}
.inp-field:disabled{opacity:0.38;cursor:not-allowed;pointer-events:none;}
.inp-unit{background:var(--unit-bg);border:1px solid var(--border);
  border-radius:0 8px 8px 0;padding:0 10px;font-size:11px;color:var(--text3);
  display:flex;align-items:center;white-space:nowrap;}
.inp-unit.disabled{opacity:0.38;}
.inp-hint{font-size:10px;color:var(--text3);padding-left:2px;}
.inp-calc{font-size:11px;color:var(--cyan);font-family:'Exo 2',sans-serif;font-weight:600;padding-left:2px;}
.inp-warn{font-size:10px;color:var(--orange);padding-left:2px;}
.inp-div{height:1px;background:var(--border2);margin:4px 0;}
.inp-sub-label{font-size:10px;font-weight:600;color:var(--text3);letter-spacing:1.2px;
  text-transform:uppercase;margin-top:4px;padding-left:2px;}
.info-icon{width:15px;height:15px;border-radius:50%;background:rgba(100,116,139,0.1);
  border:1px solid var(--border);color:var(--text3);font-size:9px;
  display:flex;align-items:center;justify-content:center;font-weight:700;flex-shrink:0;
  position:relative;cursor:help;}
.info-icon .info-tip{display:none;position:absolute;left:20px;top:-4px;width:200px;
  background:var(--tip-bg);border:1px solid var(--border);border-radius:8px;
  padding:8px 10px;font-size:11px;color:var(--text2);line-height:1.6;z-index:300;
  box-shadow:0 4px 20px rgba(0,0,0,0.3);white-space:normal;}
.info-icon:active .info-tip,.info-icon:focus .info-tip{display:block;}

.ss-wrap{display:flex;flex-direction:column;gap:6px;margin-top:2px;}
.ss-labels{display:flex;justify-content:space-between;align-items:center;}
.ss-lbl{font-size:10px;color:var(--text3);}
.ss-lbl.active-1c1d{color:var(--cyan);font-weight:600;}
.ss-lbl.active-2c2d{color:var(--blue);font-weight:600;}
.ss-track-outer{position:relative;padding:10px 0 4px;}
.ss-range{
  width:100%;-webkit-appearance:none;appearance:none;
  height:6px;border-radius:3px;outline:none;cursor:pointer;
  background:var(--bar-track);
  transition:background .15s;
}
.ss-range::-webkit-slider-thumb{
  -webkit-appearance:none;width:22px;height:22px;border-radius:50%;
  background:linear-gradient(135deg,var(--blue),var(--cyan));
  border:2px solid var(--card);
  box-shadow:0 0 8px rgba(59,130,246,0.55);
  cursor:grab;transition:box-shadow .2s;
}
.ss-range::-webkit-slider-thumb:active{cursor:grabbing;box-shadow:0 0 14px rgba(59,130,246,0.8);}
.ss-range::-moz-range-thumb{
  width:22px;height:22px;border-radius:50%;
  background:linear-gradient(135deg,var(--blue),var(--cyan));
  border:2px solid var(--card);cursor:grab;
}
.ss-ticks{display:flex;justify-content:space-between;padding:0 1px;margin-top:3px;}
.ss-tick{display:flex;flex-direction:column;align-items:center;gap:2px;}
.ss-tick-line{width:1px;height:4px;background:var(--border);border-radius:1px;}
.ss-tick-num{font-size:8px;color:var(--text3);font-family:'Exo 2',sans-serif;}
.ss-info{font-size:11px;color:var(--cyan);font-family:'Exo 2',sans-serif;
  font-weight:600;text-align:center;padding:4px 8px;
  background:rgba(6,182,212,0.07);border:1px solid rgba(6,182,212,0.18);
  border-radius:6px;line-height:1.5;}

.inv-sec{padding:12px 14px 14px;}
.inv-top{display:flex;align-items:center;justify-content:space-between;margin-bottom:10px;}
.inv-title{font-size:11px;color:var(--text3);}
.capex-lbl{font-family:'Exo 2',sans-serif;font-size:14px;font-weight:700;color:var(--cyan);}
.inv-bar-row{display:flex;align-items:center;gap:8px;margin:5px 0;}
.inv-bar-lbl{font-size:10px;color:var(--text3);width:44px;flex-shrink:0;text-align:right;}
.inv-bar-track{flex:1;height:16px;background:var(--bar-track);border-radius:4px;overflow:hidden;position:relative;}
.inv-bar-fill{height:100%;border-radius:4px;transition:width .5s;display:flex;align-items:center;justify-content:flex-end;padding-right:6px;}
.inv-bar-val{font-size:9px;font-family:'Exo 2',sans-serif;font-weight:700;color:rgba(255,255,255,0.9);white-space:nowrap;}
.inv-bar-info{font-size:10px;color:var(--text3);width:90px;flex-shrink:0;text-align:right;font-family:'Exo 2',sans-serif;}

.wf-sec{padding:12px 14px 14px;border-top:1px solid var(--border2);}
.wf-title{font-size:11px;color:var(--text3);margin-bottom:10px;}
.wf-row{display:flex;align-items:center;gap:8px;margin:4px 0;}
.wf-lbl{font-size:10px;color:var(--text3);width:72px;flex-shrink:0;text-align:right;line-height:1.4;}
.wf-track{flex:1;height:18px;background:var(--bar-track);border-radius:4px;overflow:hidden;position:relative;}
.wf-fill{height:100%;border-radius:4px;transition:width .5s;display:flex;align-items:center;justify-content:flex-end;padding-right:6px;overflow:hidden;}
.wfv{font-size:9px;font-family:'Exo 2',sans-serif;font-weight:700;color:rgba(255,255,255,0.92);white-space:nowrap;}
.wf-deduct{font-size:9px;color:var(--text3);margin-left:2px;white-space:nowrap;}
.wf-arrow{font-size:10px;color:var(--text3);text-align:center;margin:1px 0;padding-left:80px;}
.bar-notes{display:flex;flex-wrap:wrap;gap:6px;margin-top:10px;}
.bar-note{font-size:10px;color:var(--text3);background:rgba(255,255,255,0.03);
  border:1px solid var(--border2);border-radius:6px;padding:3px 8px;}
body.light .bar-note{background:rgba(0,0,0,0.03);}
.bar-note span{color:var(--text2);font-family:'Exo 2',sans-serif;font-weight:600;}

.fin-grid{display:grid;grid-template-columns:1fr 1fr;gap:10px;padding:14px;}
.fin-card{background:var(--fin-bg);border:1px solid var(--fin-border);border-radius:10px;padding:13px 14px;}
.fin-card.g{border-color:rgba(16,185,129,0.35);background:rgba(16,185,129,0.06);}
.fin-card.b{border-color:rgba(59,130,246,0.35);background:rgba(59,130,246,0.06);}
.fin-card.p{border-color:rgba(139,92,246,0.35);background:rgba(139,92,246,0.06);}
.fin-card.o{border-color:rgba(245,158,11,0.35);background:rgba(245,158,11,0.06);}
.fin-card.r{border-color:rgba(239,68,68,0.3);background:rgba(239,68,68,0.05);}
.fin-c-lbl{font-size:10px;color:var(--text3);margin-bottom:5px;}
.fin-c-val{font-family:'Exo 2',sans-serif;font-size:20px;font-weight:800;line-height:1;margin-bottom:3px;}
.fin-c-sub{font-size:10px;color:var(--text3);line-height:1.4;}

.table-wrap{overflow-x:auto;padding:0 14px 14px;}
.fin-table{width:100%;border-collapse:collapse;font-size:11px;min-width:460px;}
.fin-table th{text-align:left;padding:8px 10px;
  background:rgba(42,58,85,0.25);color:var(--text3);font-weight:500;font-size:10px;
  border-bottom:1px solid var(--border2);white-space:nowrap;}
body.light .fin-table th{background:rgba(180,200,235,0.25);}
.fin-table td{padding:9px 10px;border-bottom:1px solid var(--border2);color:var(--text2);}
.fin-table tr:last-child td{border-bottom:none;}
.fin-table td:first-child{color:var(--text3);font-size:10px;white-space:nowrap;}
.fin-table td.val{font-family:'Exo 2',sans-serif;font-weight:600;color:var(--text);text-align:right;white-space:nowrap;}
.fin-table td.val.pos{color:var(--green);}
.fin-table td.val.neg{color:var(--red);}
.fin-table td.val.warn{color:var(--orange);}

.dscr-sec{padding:12px 14px 14px;border-top:1px solid var(--border2);}
.dscr-hd{display:flex;align-items:baseline;gap:8px;margin-bottom:10px;}
.dscr-title{font-family:'Exo 2',sans-serif;font-size:12px;font-weight:700;color:var(--text);}
.dscr-sub{font-size:10px;color:var(--text3);}
.dscr-no-loan{height:80px;display:flex;flex-direction:column;align-items:center;justify-content:center;
  gap:6px;color:var(--text3);font-size:11px;
  background:rgba(255,255,255,0.02);border:1px dashed var(--border2);border-radius:8px;}
body.light .dscr-no-loan{background:rgba(0,0,0,0.02);}

.hm-sec{padding:14px;}
.hm-title{font-family:'Exo 2',sans-serif;font-size:13px;font-weight:700;color:var(--text);margin-bottom:3px;}
.hm-sub{font-size:10px;color:var(--text3);margin-bottom:10px;line-height:1.5;}
.hm-leg-row{display:flex;align-items:center;gap:8px;margin-bottom:10px;}
.hm-leg-lbl{font-size:10px;color:var(--text3);}
.hm-grad-bar{height:8px;flex:1;max-width:160px;border-radius:4px;
  background:linear-gradient(90deg,#7F1D1D,#EF4444,#F59E0B,#84CC16,#10B981);
  border:1px solid var(--border);}
.hm-scroll{overflow-x:auto;-webkit-overflow-scrolling:touch;}
.hm-table{border-collapse:separate;border-spacing:3px;white-space:nowrap;}
.hm-table th{font-size:10px;color:var(--text3);padding:4px 3px;text-align:center;
  font-weight:500;font-family:'Exo 2',sans-serif;}
.hm-table td{padding:5px 3px;text-align:center;font-size:11px;font-family:'Exo 2',sans-serif;
  font-weight:600;cursor:default;border-radius:4px;min-width:42px;}
.hm-table td.cur{outline:2px solid rgba(255,255,255,0.9);outline-offset:-1px;position:relative;z-index:3;}
body.light .hm-table td.cur{outline-color:#1E40AF;}
.cur-dot{display:inline-block;width:5px;height:5px;background:var(--blue);
  border-radius:50%;vertical-align:super;margin-left:1px;}

.cf-sec{padding:14px;}
.cf-top{display:flex;align-items:flex-start;justify-content:space-between;margin-bottom:12px;}
.cf-title{font-family:'Exo 2',sans-serif;font-size:13px;font-weight:700;color:var(--text);}
.cf-sub{font-size:10px;color:var(--text3);margin-top:3px;}
.cf-canvas-wrap{position:relative;height:220px;}
.cf-legend{display:flex;flex-wrap:wrap;gap:10px;margin-top:10px;align-items:center;}
.cf-leg{display:flex;align-items:center;gap:5px;font-size:10px;color:var(--text3);}
.cf-leg-box{width:10px;height:10px;border-radius:2px;}
.cf-pb-row{margin-left:auto;display:flex;align-items:center;gap:4px;font-size:10px;color:var(--text3);}

.footer{margin-top:8px;padding:20px 16px 32px;border-top:1px solid var(--border2);
  display:flex;flex-direction:column;align-items:center;gap:8px;text-align:center;}
.footer-main{font-family:'Exo 2',sans-serif;font-size:12px;font-weight:600;color:var(--text3);}
.footer-contact-row{display:flex;align-items:center;gap:6px;font-size:11px;color:var(--text3);opacity:.75;flex-wrap:wrap;justify-content:center;}
.footer-contact-row svg{flex-shrink:0;opacity:.55;}
.footer-contact-row a{color:var(--text3);text-decoration:none;transition:color .2s;}
.footer-contact-row a:hover{color:var(--cyan);}
.footer-contact-id-sep{color:var(--text3);opacity:.5;margin:0 1px;}
.footer-div{width:1px;height:12px;background:var(--border2);opacity:.5;margin:0 4px;}
.footer-cta{font-family:'Exo 2',sans-serif;font-size:12px;font-weight:600;color:var(--cyan);opacity:.75;}
.footer-disclaimer{font-size:10px;color:var(--text3);opacity:.6;line-height:1.6;
  max-width:280px;padding-top:8px;border-top:1px solid var(--border2);margin-top:2px;}
.footer-date{font-family:'Exo 2',sans-serif;font-size:10px;color:var(--text3);opacity:.5;}

/* CSS 媒体查询（Media Query），根据不同的设备或屏幕尺寸应用不同的样式 */
@media (min-width: 640px) {
  :root { --header-h: 60px; }
  .topbar-inner { padding: 0 24px; gap: 14px; }
  .topbar-title { font-size: 16px; }
  .scroll-body { padding: calc(var(--header-h) + 16px) 20px 32px; gap: 14px; }
  .summary-grid { grid-template-columns: 1fr 1fr 1fr 1fr; }
  .sg-cell { border-bottom: none; }
  .sg-cell:nth-child(2n) { border-right: 1px solid var(--border2); }
  .sg-cell:nth-child(4n), .sg-cell:last-child { border-right: none; }
  .fin-grid { grid-template-columns: 1fr 1fr 1fr 1fr; }
  .input-sec { padding: 16px; }
  .inp-field { font-size: 16px; }
}

@media (min-width: 900px) {
  :root { --header-h: 64px; }
  .topbar-inner { padding: 0 32px; }
  .topbar-title { font-size: 17px; }
  .scroll-body { padding: calc(var(--header-h) + 20px) 28px 40px; gap: 16px; }
  .desktop-cols { flex-direction: row; align-items: flex-start; gap: 16px; }
  .col-left { width: var(--col-left-w); flex-shrink: 0; gap: 16px; position: sticky; top: calc(var(--header-h) + 12px); align-self: flex-start; }
  .col-right { flex: 1; min-width: 0; gap: 16px; }
  .summary-card { width: 100%; }
  .card-title { font-size: 14px; }
  .info-icon .info-tip { left: 22px; width: 220px; }
}

@media (min-width: 1200px) {
  :root { --col-left-w: 460px; }
  .scroll-body { padding-left: 36px; padding-right: 36px; }
  .topbar-inner { padding: 0 36px; }
}
</style>