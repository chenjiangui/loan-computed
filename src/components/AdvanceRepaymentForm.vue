<script setup>
import { ref } from 'vue'

defineProps({
  formData: {
    type: Object,
    required: true
  }
})

const emit = defineEmits(['delete'])

const formRef = ref()

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
  date: [
    {
      required: true,
      type: 'number',
      message: '请选择提前还款日期',
      trigger: 'blur'
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

async function validate() {
  return await formRef.value.validate()
}

defineExpose({
  validate
})
</script>

<template>
  <n-form ref="formRef" :model="formData" :rules="rules" inline>
    <n-form-item label="提前还款金额" path="amount">
      <n-input-number
        v-model:value="formData.amount"
        placeholder="请输入提前还款金额"
        :show-button="false"
        :precision="0"
        min="1"
      >
        <template #suffix>万元</template>
      </n-input-number>
    </n-form-item>
    <n-form-item label="提前还款日期" path="date">
      <n-date-picker v-model:value="formData.date" type="month"></n-date-picker>
    </n-form-item>
    <n-form-item label="新利率" path="rate">
      <n-input-number
        v-model:value="formData.rate"
        placeholder="请输入贷款利率"
        :show-button="false"
        :precision="2"
        min="0.01"
        max="100"
      >
        <template #suffix>%</template>
      </n-input-number>
    </n-form-item>
    <n-form-item>
      <n-button type="error" @click="emit('delete')">删除</n-button>
    </n-form-item>
  </n-form>
</template>
