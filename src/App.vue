<script setup>
import { reactive, ref, onMounted } from 'vue'
import { chain, flatten, reduce, round, take, uniqueId } from 'lodash'
import { zhCN, dateZhCN } from 'naive-ui'
import dayjs from 'dayjs'
import isSameOrBefore from 'dayjs/plugin/isSameOrBefore'
import AdvanceRepaymentForm from '@/components/AdvanceRepaymentForm.vue'
import GithubIcon from '@/components/GithubIcon.vue'

dayjs.extend(isSameOrBefore)

const formRef = ref()

const monthlyPayments = ref([])

const loanForm = reactive({
  amount: 100,
  term: 360,
  rate: 5.45,
  // 还款方式
  repayType: '等额本息',
  beginMonth: dayjs('2020-01-01').valueOf()
})

const rules = {
  amount: [
    {
      required: true,
      message: '金额必须大于0',
      trigger: 'blur',
      type: 'number',
      min: 0
    }
  ],
  term: [
    {
      required: true,
      message: '期数必须大于0',
      trigger: 'blur',
      type: 'number',
      min: 1
    }
  ],
  rate: [
    {
      required: true,
      message: '利率范围在 0.01 ~ 100 %之间',
      trigger: 'blur',
      type: 'number',
      min: 0.01,
      max: 100
    }
  ]
}

/**
 * 计算每个月还款的利息和本金的数组 (等额本息)
 * @param {number} loanAmount - 贷款金额 (单位: 元)
 * @param {number} interestRate - 年利率 (百分比)
 * @param {number} months - 贷款总月数
 * @returns {Array} 每个月还款的利息和本金的对象数组 { interest: number, principal: number }
 */
function calculateMonthlyPayments(loanAmount, interestRate, months) {
  // 检查无效输入以防止错误
  if (loanAmount <= 0 || interestRate <= 0 || months <= 0) {
    console.warn('calculateMonthlyPayments: 无效输入', { loanAmount, interestRate, months })
    return []
  }
  var monthlyInterestRate = interestRate / 100 / 12

  // 避免除零或负指数问题
  var powerTerm = Math.pow(1 + monthlyInterestRate, -months)
  if (1 - powerTerm <= 1e-9) {
    // 如果利率非常低或月数非常大，或月利率为0，可能导致月供计算问题
    console.warn('由于利率/期数组合问题，无法计算月供。')
    return []
  }

  var monthlyPayment = (loanAmount * monthlyInterestRate) / (1 - powerTerm)

  var payments = []
  var remainingLoan = loanAmount

  for (var i = 0; i < months; i++) {
    var interest = remainingLoan * monthlyInterestRate
    // 确保本金不为负（如果月供小于利息，可能发生在舍入或边缘情况）
    var principal = Math.max(0, monthlyPayment - interest)
    // 确保剩余贷款不会因浮点问题显著低于零
    remainingLoan -= principal
    if (remainingLoan < 1e-2) {
      principal += remainingLoan // 调整最后一期本金以精确还清
      remainingLoan = 0
    }

    var payment = {
      interest: interest,
      principal: principal
    }
    payments.push(payment)
    if (remainingLoan <= 0) {
      break // 如果贷款提前还清则停止
    }
  }

  return payments
}

/**
 * 计算每个月还款的利息和本金的数组 (等额本金)
 * @param {number} loanAmount - 贷款金额 (单位: 元)
 * @param {number} interestRate - 年利率 (百分比)
 * @param {number} months - 贷款总月数
 * @returns {Array} 每个月还款的利息和本金的对象数组 { interest: number, principal: number }
 */
function calculateMonthlyPaymentsByPrincipal(loanAmount, interestRate, months) {
  // 检查无效输入以防止错误
  if (loanAmount <= 0 || interestRate <= 0 || months <= 0) {
    console.warn('calculateMonthlyPaymentsByPrincipal: 无效输入', { loanAmount, interestRate, months })
    return []
  }

  const monthlyInterestRate = interestRate / 100 / 12
  // 每月应还本金
  const monthlyPrincipal = loanAmount / months

  var payments = []
  var remainingLoan = loanAmount

  for (var i = 0; i < months; i++) {
    var interest = remainingLoan * monthlyInterestRate
    var principal = monthlyPrincipal

    // 处理最后一期可能的舍入误差
    if (i === months - 1) {
      principal = remainingLoan
    }

    remainingLoan -= principal
    if (remainingLoan < 1e-2) {
      remainingLoan = 0
    }

    payments.push({
      interest: interest,
      principal: principal
    })

    if (remainingLoan <= 0) {
      break
    }
  }

  return payments
}

// 辅助函数：计算剩余所需期数
function calculateRemainingTerm(loanAmount, interestRate, monthlyPayment) {
  // 基本验证
  if (loanAmount <= 0 || interestRate <= 0 || monthlyPayment <= 0) return 0

  const monthlyInterestRate = interestRate / 100 / 12
  // 检查月供是否覆盖利息
  const minPaymentRequired = loanAmount * monthlyInterestRate
  if (monthlyPayment <= minPaymentRequired + 1e-9) {
    // 月供不足以支付利息，期数实际上是无限的或贷款会增加
    console.warn('月供小于或等于利息。无法计算剩余期数。')
    return Infinity // 或作为错误/回退处理
  }

  // 公式：n = -log(1 - (P * r) / A) / log(1 + r)  <- 注释中的公式不准确，下面使用的是正确的
  // 正确公式：n = -log(1 - (loanAmount * monthlyInterestRate) / monthlyPayment) / log(1 + monthlyInterestRate)
  const logArg = 1 - (loanAmount * monthlyInterestRate) / monthlyPayment
  if (logArg <= 0) {
    // 这意味着月供不足或 loanAmount/rate/payment 组合无效
    console.warn('期数计算中对数函数的参数无效。')
    return Infinity // 或处理错误
  }

  const term = -Math.log(logArg) / Math.log(1 + monthlyInterestRate)
  // 为安全起见使用 ceil，确保贷款能够还清
  return Math.ceil(term)
}

// 辅助函数：确定"减少年限"时的参考月供
function getReduceTermReferencePayment(paymentsInPeriod, index, initialPayments, repaidData) {
  if (paymentsInPeriod.length > 0) {
    // 使用刚刚完成的期间的最后一笔实际支付作为参考
    const lastActualPayment = paymentsInPeriod[paymentsInPeriod.length - 1]
    return lastActualPayment.interest + lastActualPayment.principal
  } else if (index === 0 && initialPayments.length > 0) {
    // 如果是第一次提前还款，使用初始计算的月供
    return initialPayments[0].interest + initialPayments[0].principal
  } else {
    // 回退：如果当前期间为空，尝试从前一个还款段查找月供
    const prevSegment = repaidData.length > 1 ? repaidData[repaidData.length - 2] : null
    if (prevSegment && prevSegment.length > 0) {
      const lastPrevPayment = prevSegment[prevSegment.length - 1]
      return lastPrevPayment.interest + lastPrevPayment.principal
    }
  }
  // 最终回退：如果找不到之前的月供，尽可能使用初始月供
  return initialPayments.length > 0 ? initialPayments[0].interest + initialPayments[0].principal : -1
}

// 辅助函数：格式化最终数据以供表格显示
function formatMonthlyPaymentsForTable(repaidData, loanForm, advanceRepaymentFormDatas) {
  return chain(repaidData)
    .flatten()
    .map((item, index) => {
      // 在每一步基于扁平化数据精确重新计算剩余贷款
      const principalPaidToDate = flatten(repaidData)
        .slice(0, index + 1)
        .reduce((a, b) => a + b.principal, 0)
      const advancePaidToDate = advanceRepaymentFormDatas
        // 筛选日期早于或等于当前付款月份开始的提前还款记录
        .filter(form => form.date && dayjs(form.date).isBefore(dayjs(loanForm.beginMonth).add(index + 1, 'month')))
        .reduce((sum, form) => sum + (form.amount || 0) * 10000, 0)

      const currentRemainingLoan = loanForm.amount * 10000 - principalPaidToDate - advancePaidToDate

      return {
        month: dayjs(loanForm.beginMonth).add(index, 'month').format('YYYY-MM-01'),
        monthlyRepay: round(item.interest + item.principal, 2),
        principal: round(item.principal, 2),
        interest: round(item.interest, 2),
        remainingLoan: round(Math.max(0, currentRemainingLoan), 2)
      }
    })
    .value()
}

const columns = [
  {
    title: '月份',
    key: 'month'
  },
  {
    title: '月供',
    key: 'monthlyRepay'
  },
  {
    title: '本金',
    key: 'principal'
  },
  {
    title: '利息',
    key: 'interest'
  },
  {
    title: '剩余本金',
    key: 'remainingLoan'
  }
]

// 提前还款表单的数据
const advanceRepaymentFormDatas = reactive([
  {
    amount: 30,
    date: dayjs('2023-12-01').valueOf(),
    rate: 3.3,
    reduceType: 'reducePayment',
    key: uniqueId()
  }
])

// 提前还款表单组件的引用数组
const advanceRepaymentForms = ref([])

// 当提前还款日期改变时，对表单数据按日期排序
function onAdvanceRepaymentFormDateChange() {
  advanceRepaymentFormDatas.sort((a, b) => a.date - b.date)
}

// 新增一个空的提前还款表单数据项
function addAdvanceRepaymentForm() {
  advanceRepaymentFormDatas.push({
    amount: null,
    date: null,
    rate: null,
    reduceType: 'reducePayment',
    key: uniqueId()
  })
}

// 保存计算出的分段还款数据（二维数组）
const repaidData = reactive([])

// 辅助函数：根据还款方式计算新的还款计划
function calculateNewPaymentPlan(remainingLoan, rate, term, repayType, referencePayment = null) {
  if (repayType === '等额本息') {
    return calculateMonthlyPayments(remainingLoan, rate, term)
  } else {
    // 等额本金方式
    return calculateMonthlyPaymentsByPrincipal(remainingLoan, rate, term)
  }
}

// 主要计算函数：计算考虑提前还款后的月供详情
async function computedMonthlyPayments() {
  // 缓存数据到localStorage
  try {
    localStorage.setItem('loanForm', JSON.stringify(loanForm))
    localStorage.setItem('advanceRepaymentFormDatas', JSON.stringify(advanceRepaymentFormDatas))
  } catch (e) {
    console.warn('缓存数据失败', e)
  }

  // 首先验证所有表单
  try {
    await formRef.value?.validate()
    await Promise.all(advanceRepaymentForms.value.map((form) => form.validate()))
  } catch (validationError) {
    console.error('表单验证失败:', validationError)
    monthlyPayments.value = []
    return
  }

  let repaidPrincipal = 0 // 已偿还的总本金 (包括提前还款)
  repaidData.length = 0 // 清空之前的计算结果
  let effectiveTotalTerm = loanForm.term // 贷款的有效总期数，会因"减少年限"而改变

  // 计算初始的完整还款计划
  const initialPayments = calculateNewPaymentPlan(
    loanForm.amount * 10000,
    loanForm.rate,
    loanForm.term,
    loanForm.repayType
  )

  if (!initialPayments || initialPayments.length === 0) {
    console.error('初始贷款计算失败。')
    monthlyPayments.value = []
    return
  }
  repaidData.push([...initialPayments])

  // 按顺序处理每一笔提前还款
  for (let index = 0; index < advanceRepaymentFormDatas.length; index++) {
    const advanceForm = advanceRepaymentFormDatas[index]

    // --- 输入验证与日期处理 ---
    if (!advanceForm.date) {
      console.warn(`跳过第 ${index + 1} 笔提前还款，因为缺少日期。`)
      continue
    }

    const previousDate = index === 0 ? dayjs(loanForm.beginMonth) : dayjs(advanceRepaymentFormDatas[index - 1].date)
    if (!previousDate || dayjs(advanceForm.date).isSameOrBefore(previousDate)) {
      console.warn(`跳过第 ${index + 1} 笔提前还款，因为日期顺序无效或缺少前一日期。`)
      continue
    }

    const diffMonth = dayjs(advanceForm.date).diff(previousDate, 'month')
    if (diffMonth < 0) {
      console.warn(`跳过第 ${index + 1} 笔提前还款，因为月份差为负。`)
      continue
    }

    // --- 还款段处理 ---
    const lastSegment = repaidData[repaidData.length - 1]
    if (!lastSegment) {
      console.error('计算错误：缺少前一个还款段。')
      break
    }

    const paymentsInPeriod = take(lastSegment, diffMonth)
    const principalPaidInPeriod = chain(paymentsInPeriod).map('principal').sum().value()
    repaidPrincipal += principalPaidInPeriod
    repaidPrincipal += (advanceForm.amount || 0) * 10000
    repaidData[repaidData.length - 1] = paymentsInPeriod

    // --- 计算剩余状态 ---
    const remainingLoanAfterAdvance = Math.max(0, loanForm.amount * 10000 - repaidPrincipal)
    if (remainingLoanAfterAdvance < 1e-2) {
      effectiveTotalTerm = reduce(repaidData, (sum, segment) => sum + segment.length, 0)
      break
    }

    const totalMonthsPaid = reduce(repaidData, (sum, segment) => sum + segment.length, 0)
    const currentRate = advanceForm.rate || loanForm.rate // 使用新利率，如果没有设置则使用原利率

    // --- 根据提前还款类型计算新的还款计划 ---
    let newSegmentTerm = 0
    if (advanceForm.reduceType === 'reduceTerm') {
      // 减少年限：保持月供基本不变（等额本息）或保持每月还款本金不变（等额本金）
      if (loanForm.repayType === '等额本息') {
        // 等额本息：基于上一期月供计算新的期数
        const referencePayment = getReduceTermReferencePayment(paymentsInPeriod, index, initialPayments, repaidData)
        const minPaymentRequired = remainingLoanAfterAdvance * (currentRate / 100 / 12)
        
        if (referencePayment <= minPaymentRequired + 1e-9 || referencePayment <= 0) {
          newSegmentTerm = Math.max(0, effectiveTotalTerm - totalMonthsPaid)
        } else {
          newSegmentTerm = calculateRemainingTerm(remainingLoanAfterAdvance, currentRate, referencePayment)
        }
      } else {
        // 等额本金：保持每月还款本金不变
        const lastPayment = paymentsInPeriod[paymentsInPeriod.length - 1]
        if (lastPayment) {
          const monthlyPrincipal = lastPayment.principal
          newSegmentTerm = Math.ceil(remainingLoanAfterAdvance / monthlyPrincipal)
        } else {
          // 如果没有参考还款，则基于剩余本金和剩余期数计算
          newSegmentTerm = Math.ceil(remainingLoanAfterAdvance / (remainingLoanAfterAdvance / (effectiveTotalTerm - totalMonthsPaid)))
        }
      }
    } else {
      // 减少月供：保持剩余期数不变
      newSegmentTerm = Math.max(0, effectiveTotalTerm - totalMonthsPaid)
    }

    // 确保计算出的期数有效
    if (newSegmentTerm === Infinity || newSegmentTerm < 0) {
      console.warn(`第 ${index + 1} 笔提前还款：计算剩余期数失败，使用剩余期数。`)
      newSegmentTerm = Math.max(0, effectiveTotalTerm - totalMonthsPaid)
    }

    // 生成新的还款计划
    if (newSegmentTerm > 0 && remainingLoanAfterAdvance > 1e-2) {
      const newSegmentPayments = calculateNewPaymentPlan(
        remainingLoanAfterAdvance,
        currentRate,
        newSegmentTerm,
        loanForm.repayType
      )

      if (newSegmentPayments && newSegmentPayments.length > 0) {
        repaidData.push(newSegmentPayments)
        effectiveTotalTerm = totalMonthsPaid + newSegmentTerm
      } else {
        console.warn(
          `为第 ${index + 1} 次提前还款后的段计算出 0 笔付款，但期数=${newSegmentTerm} 且剩余本金=${remainingLoanAfterAdvance}`
        )
        effectiveTotalTerm = totalMonthsPaid
        break
      }
    } else {
      effectiveTotalTerm = totalMonthsPaid
      break
    }
  }

  // 最终格式化输出
  monthlyPayments.value = formatMonthlyPaymentsForTable(repaidData, loanForm, advanceRepaymentFormDatas)
}

onMounted(() => {
  try {
    const loanFormCache = localStorage.getItem('loanForm')
    const advanceRepaymentCache = localStorage.getItem('advanceRepaymentFormDatas')
    if (loanFormCache) {
      const parsed = JSON.parse(loanFormCache)
      Object.assign(loanForm, parsed)
      // 日期字段特殊处理
      if (parsed.beginMonth) {
        loanForm.beginMonth = parsed.beginMonth
      }
    }
    if (advanceRepaymentCache) {
      const arr = JSON.parse(advanceRepaymentCache)
      advanceRepaymentFormDatas.splice(0, advanceRepaymentFormDatas.length, ...arr.map(item => ({
        ...item,
        // 日期字段特殊处理
        date: item.date ? item.date : null
      })))
    }
  } catch (e) {
    console.warn('读取缓存失败', e)
  }
})
</script>

<template>
  <n-config-provider :locale="zhCN" :date-locale="dateZhCN">
    <n-flex id="container" style="height: 100vh; gap: 16px; padding: 16px;">
      <n-flex vertical style="flex: 1; height: 100%; overflow-y: auto;">
        <!-- 贷款基础信息表单 -->
        <div>
          <n-h3>
            <n-space align="center">
              <span>房贷计算器</span>
              <n-button quaternary circle tag="a" href="https://github.com/chenjiangui/loan-computed" target="_blank" style="margin-left: 8px;">
                <GithubIcon />
              </n-button>
            </n-space>
          </n-h3>
          <n-form
            ref="formRef"
            label-placement="left"
            label-width="auto"
            :model="loanForm"
            :rules="rules"
          >
            <n-form-item label="贷款金额" path="amount">
              <n-input-number
                v-model:value="loanForm.amount"
                placeholder="请输入贷款金额"
                :show-button="false"
                :precision="2"
                min="0"
              >
                <template #suffix>万元</template>
              </n-input-number>
            </n-form-item>
            <n-form-item label="贷款期数" path="term">
              <n-input-number
                v-model:value="loanForm.term"
                placeholder="请输入贷款期数"
                :show-button="false"
                max="360"
                min="1"
                :precision="0"
              >
                <template #suffix>期</template>
              </n-input-number>
            </n-form-item>
            <n-form-item label="贷款利率" path="rate">
              <n-input-number
                v-model:value="loanForm.rate"
                placeholder="请输入贷款利率"
                :show-button="false"
                :precision="2"
                min="0.01"
                max="100"
              >
                <template #suffix>%</template>
              </n-input-number>
            </n-form-item>
            <n-form-item label="还款方式">
              <n-radio-group v-model:value="loanForm.repayType">
                <n-radio value="等额本息">等额本息</n-radio>
                <n-radio value="等额本金">等额本金</n-radio>
              </n-radio-group>
            </n-form-item>
            <n-form-item label="开始还款日期">
              <n-date-picker v-model:value="loanForm.beginMonth" type="month"></n-date-picker>
            </n-form-item>
          </n-form>
        </div>

        <!-- 提前还款操作区域 -->
        <n-h3>
          <n-space justify="space-between">
            <div>
              <n-button type="primary" @click="addAdvanceRepaymentForm">新增提前还款</n-button>
              <n-text type="info"> (如果输入金额为0，则只调整利率) </n-text>
            </div>
            <n-button type="primary" @click="computedMonthlyPayments">开始计算</n-button>
          </n-space>
        </n-h3>

        <!-- 提前还款表单列表 -->
        <div style="flex: 1; overflow-y: auto;">
          <advance-repayment-form
            v-for="(form, index) in advanceRepaymentFormDatas"
            ref="advanceRepaymentForms"
            :key="form.key"
            :form-data="form"
            @delete="advanceRepaymentFormDatas.splice(index, 1)"
            @date-change="onAdvanceRepaymentFormDateChange"
          ></advance-repayment-form>
        </div>
      </n-flex>

      <!-- 计算结果表格 -->
      <div style="flex: 1">
        <n-h3>计算结果</n-h3>
        <n-data-table
          max-height="calc(100vh - 150px)" 
          :data="monthlyPayments"
          :columns="columns"
          bordered
          virtual-scroll 
        ></n-data-table>
      </div>
    </n-flex>
  </n-config-provider>
</template>
