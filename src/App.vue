<script setup>
import { reactive, ref } from 'vue'
import { chain, flatten, reduce, round, take, uniqueId } from 'lodash'
import { zhCN, dateZhCN } from 'naive-ui'
import dayjs from 'dayjs'
import AdvanceRepaymentForm from '@/components/AdvanceRepaymentForm.vue'

const formRef = ref()

const monthlyPayments = ref([])

const loanForm = reactive({
  amount: 139,
  term: 30,
  rate: 5.45,
  //还款方式
  repayType: '等额本息',
  beginMonth: dayjs('2020-01-01').valueOf()
})

const rules = {
  amount: [
    {
      required: true,
      message: '金额必须大于一万元',
      trigger: 'blur',
      type: 'number',
      min: 1
    }
  ],
  term: [
    {
      required: true,
      message: '年限范围在 1~30 年之间',
      trigger: 'blur',
      type: 'number',
      min: 1,
      max: 30
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
 * 计算每个月还款的利息和本金的数组
 * @param {number} loanAmount - 贷款金额
 * @param {number} interestRate - 利率（百分比）
 * @param {number} months - 贷款月份
 * @returns {Array} 每个月还款的利息和本金的数组
 */
function calculateMonthlyPayments(loanAmount, interestRate, months) {
  var monthlyInterestRate = interestRate / 100 / 12

  var monthlyPayment =
    (loanAmount * monthlyInterestRate) / (1 - Math.pow(1 + monthlyInterestRate, -months))

  var payments = []
  var remainingLoan = loanAmount

  for (var i = 0; i < months; i++) {
    var interest = remainingLoan * monthlyInterestRate
    var principal = monthlyPayment - interest

    var payment = {
      interest: interest,
      principal: principal
    }

    payments.push(payment)

    remainingLoan -= principal
  }

  return payments
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

const advanceRepaymentFormDatas = reactive([
  {
    amount: 30,
    date: dayjs('2023-12-01').valueOf(),
    rate: 4.2,
    key: uniqueId()
  },
  {
    amount: 10,
    date: dayjs('2024-12-01').valueOf(),
    rate: 4.2,
    key: uniqueId()
  },
  {
    amount: 10,
    date: dayjs('2025-12-01').valueOf(),
    rate: 4.2,
    key: uniqueId()
  }
])
const advanceRepaymentForms = ref([])

function addAdvanceRepaymentForm() {
  advanceRepaymentFormDatas.push({
    amount: null,
    date: null,
    rate: null,
    key: uniqueId()
  })
}

const repaidData = reactive([])

async function computedMonthlyPayments() {
  await formRef.value?.validate()
  await Promise.all(advanceRepaymentForms.value.map((form) => form.validate()))
  // 已经偿还的本金
  let repaidPrincipal = 0
  // 偿还数据的二维数组
  repaidData.length = 0
  repaidData.push(
    calculateMonthlyPayments(loanForm.amount * 10000, loanForm.rate, loanForm.term * 12)
  )

  for (let index = 0; index < advanceRepaymentFormDatas.length; index++) {
    // debugger
    const advanceForm = advanceRepaymentFormDatas[index]
    const diffMonth = dayjs(advanceForm.date).diff(
      index == 0 ? dayjs(loanForm.beginMonth) : advanceRepaymentFormDatas[index - 1].date,
      'month'
    )
    repaidPrincipal += advanceForm.amount * 10000
    repaidPrincipal += chain(repaidData)
      .last()
      .take(diffMonth)
      .map((item) => item.principal)
      .sum()
      .value()
    repaidData[repaidData.length - 1] = take(repaidData[repaidData.length - 1], diffMonth)

    const advanceRepaymentData = calculateMonthlyPayments(
      loanForm.amount * 10000 - repaidPrincipal,
      advanceForm.rate,
      loanForm.term * 12 - reduce(repaidData, (a, b) => a + b.length, 0)
    )
    repaidData.push(advanceRepaymentData)
  }

  monthlyPayments.value = chain(repaidData)
    .flatten()
    .map((item, index) => {
      return {
        month: dayjs(loanForm.beginMonth).add(index, 'month').format('YYYY-MM-01'),
        monthlyRepay: round(item.interest + item.principal, 2),
        principal: round(item.principal, 2),
        interest: round(item.interest, 2),
        remainingLoan: round(
          loanForm.amount * 10000 -
            flatten(repaidData)
              .slice(0, index + 1)
              .reduce((a, b) => a + b.principal, 0),
          2
        )
      }
    })
    .value()
}
</script>

<template>
  <n-config-provider :locale="zhCN" :date-locale="dateZhCN">
    <n-flex id="container" vertical style="height: 100vh; padding: 32px">
      <n-flex>
        <div style="flex: 1">
          <n-h3>房贷计算器</n-h3>
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
                :precision="0"
                min="1"
              >
                <template #suffix>万元</template>
              </n-input-number>
            </n-form-item>
            <n-form-item label="贷款年限" path="term">
              <n-input-number
                v-model:value="loanForm.term"
                placeholder="请输入贷款年限"
                :show-button="false"
                max="30"
                min="1"
                :precision="0"
              >
                <template #suffix>年</template>
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
                <!-- <n-radio value="等额本金">等额本金</n-radio> -->
              </n-radio-group>
            </n-form-item>
            <n-form-item label="开始还款日期">
              <n-date-picker v-model:value="loanForm.beginMonth" type="month"></n-date-picker>
            </n-form-item>
          </n-form>
        </div>

        <div style="flex: 1">
          <n-h3>
            <n-button type="primary" @click="addAdvanceRepaymentForm">提前还款</n-button>
          </n-h3>
          <advance-repayment-form
            v-for="(form, index) in advanceRepaymentFormDatas"
            ref="advanceRepaymentForms"
            :key="form.key"
            :form-data="form"
            @delete="advanceRepaymentFormDatas.splice(index, 1)"
          ></advance-repayment-form>
        </div>
      </n-flex>

      <n-space justify="center">
        <n-button type="primary" @click="computedMonthlyPayments">开始计算</n-button>
      </n-space>

      <n-h3>计算结果</n-h3>

      <n-scrollbar style="flex: 1; overflow-y: auto">
        <n-data-table :data="monthlyPayments" :columns="columns" bordered></n-data-table>
      </n-scrollbar>
    </n-flex>
  </n-config-provider>
</template>
