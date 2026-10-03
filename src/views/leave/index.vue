<template>
  <el-card shadow="never" class="leave-card">
    <template #header>
      <div class="card-header">
        <span>请假申请</span>
      </div>
    </template>

    <el-form :model="form" label-width="100px">
      <el-form-item label="请假类型">
        <el-select v-model="form.type" placeholder="请选择请假类型">
          <el-option label="事假" value="事假" />
          <el-option label="病假" value="病假" />
          <el-option label="年假" value="年假" />
        </el-select>
      </el-form-item>

      <el-form-item label="开始时间">
        <el-date-picker
          v-model="form.startTime"
          type="datetime"
          placeholder="选择开始时间"
        />
      </el-form-item>

      <el-form-item label="结束时间">
        <el-date-picker
          v-model="form.endTime"
          type="datetime"
          placeholder="选择结束时间"
        />
      </el-form-item>

      <el-form-item label="请假原因">
        <el-input
          v-model="form.reason"
          type="textarea"
          rows="4"
          placeholder="请填写请假原因"
        />
      </el-form-item>

      <el-form-item>
        <el-button type="primary" @click="onSubmit">提交申请</el-button>
        <el-button @click="resetForm">重置</el-button>
      </el-form-item>
    </el-form>
  </el-card>
</template>

<script setup>
import { ref } from 'vue'
import { ElMessage } from 'element-plus'

const form = ref({
  type: '',
  startTime: '',
  endTime: '',
  reason: ''
})

const onSubmit = () => {
  console.log('请假表单：', form.value)
  ElMessage.success('提交成功！')
}

const resetForm = () => {
  form.value = {
    type: '',
    startTime: '',
    endTime: '',
    reason: ''
  }
  ElMessage.info('已重置')
}
</script>

<style scoped>
.leave-card {
  height: 100%;
}
.card-header {
  font-size: 18px;
  font-weight: bold;
}
</style>
