<template>
  <div class="todo-page">
    <div class="page-heading">
      <div>
        <h1 class="page-title">待办事项</h1>
        <p class="page-desc">安排好每一项工作，让计划稳步推进。</p>
      </div>
      <el-button type="primary" @click="openCreateDialog">
        <el-icon><Plus /></el-icon>
        新建待办
      </el-button>
    </div>

    <el-row :gutter="16" class="stats-row">
      <el-col v-for="item in stats" :key="item.label" :xs="12" :sm="6">
        <el-card shadow="never" class="stats-card">
          <div class="stats-card__label">{{ item.label }}</div>
          <div class="stats-card__value" :class="`stats-card__value--${item.tone}`">
            {{ item.value }}
          </div>
        </el-card>
      </el-col>
    </el-row>

    <el-card shadow="never" class="todo-card">
      <div class="toolbar">
        <el-input
          v-model="keyword"
          class="toolbar__search"
          clearable
          placeholder="搜索待办名称或描述"
        >
          <template #prefix><el-icon><Search /></el-icon></template>
        </el-input>
        <el-select v-model="priorityFilter" class="toolbar__priority" placeholder="全部优先级">
          <el-option label="全部优先级" value="全部" />
          <el-option label="高优先级" value="高" />
          <el-option label="中优先级" value="中" />
          <el-option label="低优先级" value="低" />
        </el-select>
      </div>

      <el-tabs v-model="statusFilter" class="status-tabs">
        <el-tab-pane label="全部" name="全部" />
        <el-tab-pane :label="`未开始 ${statusCounts.todo}`" name="未开始" />
        <el-tab-pane :label="`进行中 ${statusCounts.doing}`" name="进行中" />
        <el-tab-pane :label="`已完成 ${statusCounts.done}`" name="已完成" />
      </el-tabs>

      <el-table
        v-if="filteredTasks.length"
        :data="filteredTasks"
        row-key="id"
        class="todo-table"
      >
        <el-table-column label="待办内容" min-width="260">
          <template #default="{ row }">
            <div class="task-content">
              <span class="task-content__title" :class="{ 'is-done': row.status === '已完成' }">
                {{ row.title }}
              </span>
              <span v-if="row.description" class="task-content__description">
                {{ row.description }}
              </span>
            </div>
          </template>
        </el-table-column>
        <el-table-column label="截止日期" width="150">
          <template #default="{ row }">
            <span :class="{ 'due-date--overdue': isOverdue(row) }">
              {{ row.dueDate || '未设置' }}
            </span>
            <el-tag v-if="isOverdue(row)" type="danger" size="small" class="overdue-tag">
              已逾期
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column label="优先级" width="110">
          <template #default="{ row }">
            <el-tag :type="priorityType(row.priority)" effect="light" round>
              {{ row.priority }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column label="状态" width="110">
          <template #default="{ row }">
            <el-tag :type="statusType(row.status)" effect="light" round>
              {{ row.status }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column label="操作" width="150" fixed="right">
          <template #default="{ row }">
            <el-button
              v-if="row.status !== '已完成'"
              link
              type="success"
              title="标记完成"
              @click="completeTask(row)"
            >
              完成
            </el-button>
            <el-button link type="primary" title="编辑待办" @click="openEditDialog(row)">
              编辑
            </el-button>
            <el-button link type="danger" title="删除待办" @click="removeTask(row)">
              删除
            </el-button>
          </template>
        </el-table-column>
      </el-table>

      <el-empty v-else :description="emptyDescription" :image-size="100" />
    </el-card>

    <el-dialog
      v-model="dialogVisible"
      :title="editingId === null ? '新建待办' : '编辑待办'"
      width="520px"
      destroy-on-close
    >
      <el-form ref="formRef" :model="form" :rules="formRules" label-width="84px">
        <el-form-item label="待办名称" prop="title">
          <el-input v-model="form.title" maxlength="40" show-word-limit placeholder="例如：准备例会材料" />
        </el-form-item>
        <el-form-item label="详细描述" prop="description">
          <el-input
            v-model="form.description"
            type="textarea"
            :rows="3"
            maxlength="200"
            show-word-limit
            placeholder="补充待办事项的具体内容（选填）"
          />
        </el-form-item>
        <el-form-item label="截止日期" prop="dueDate">
          <el-date-picker
            v-model="form.dueDate"
            type="date"
            value-format="YYYY-MM-DD"
            format="YYYY年MM月DD日"
            placeholder="选择截止日期（选填）"
            class="form-field"
          />
        </el-form-item>
        <el-form-item label="优先级" prop="priority">
          <el-radio-group v-model="form.priority">
            <el-radio-button value="高">高</el-radio-button>
            <el-radio-button value="中">中</el-radio-button>
            <el-radio-button value="低">低</el-radio-button>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="当前状态" prop="status">
          <el-select v-model="form.status" class="form-field">
            <el-option label="未开始" value="未开始" />
            <el-option label="进行中" value="进行中" />
            <el-option label="已完成" value="已完成" />
          </el-select>
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="dialogVisible = false">取消</el-button>
        <el-button type="primary" :loading="saving" @click="submitForm">保存待办</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { computed, reactive, ref } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'

const STORAGE_KEY = 'student-work-todos'

const initialTasks = [
  {
    id: 1,
    title: '准备学生会例会材料',
    description: '整理本周各部门工作进度和需要讨论的问题。',
    dueDate: '2026-10-12',
    priority: '高',
    status: '进行中',
  },
  {
    id: 2,
    title: '汇总校园文化活动报名信息',
    description: '核对报名名单，并将最终人数同步给活动负责人。',
    dueDate: '2026-10-14',
    priority: '中',
    status: '未开始',
  },
  {
    id: 3,
    title: '完成新学期部门工作计划',
    description: '补充时间安排与阶段性目标，提交给指导老师。',
    dueDate: '2026-10-10',
    priority: '高',
    status: '未开始',
  },
  {
    id: 4,
    title: '更新志愿者值班表',
    description: '',
    dueDate: '2026-10-08',
    priority: '低',
    status: '已完成',
  },
  {
    id: 5,
    title: '整理上周活动反馈',
    description: '归纳同学意见，为后续活动优化提供参考。',
    dueDate: '',
    priority: '中',
    status: '已完成',
  },
]

function loadTasks() {
  try {
    const savedTasks = localStorage.getItem(STORAGE_KEY)
    if (savedTasks === null) return initialTasks.map((task) => ({ ...task }))

    const parsedTasks = JSON.parse(savedTasks)
    if (!Array.isArray(parsedTasks)) throw new Error('待办数据格式不正确')
    return parsedTasks
  } catch (error) {
    ElMessage.warning(`本地待办数据读取失败，已载入示例事项：${error.message}`)
    return initialTasks.map((task) => ({ ...task }))
  }
}

const tasks = ref(loadTasks())
const keyword = ref('')
const statusFilter = ref('全部')
const priorityFilter = ref('全部')
const dialogVisible = ref(false)
const saving = ref(false)
const editingId = ref(null)
const formRef = ref()
const form = reactive(createEmptyForm())

const formRules = {
  title: [{ required: true, message: '请输入待办名称', trigger: 'blur' }],
}

function createEmptyForm() {
  return {
    title: '',
    description: '',
    dueDate: '',
    priority: '中',
    status: '未开始',
  }
}

const statusCounts = computed(() => ({
  todo: tasks.value.filter((task) => task.status === '未开始').length,
  doing: tasks.value.filter((task) => task.status === '进行中').length,
  done: tasks.value.filter((task) => task.status === '已完成').length,
}))

const stats = computed(() => [
  { label: '全部待办', value: tasks.value.length, tone: 'primary' },
  { label: '未开始', value: statusCounts.value.todo, tone: 'warning' },
  { label: '进行中', value: statusCounts.value.doing, tone: 'info' },
  { label: '已完成', value: statusCounts.value.done, tone: 'success' },
])

const filteredTasks = computed(() => {
  const searchText = keyword.value.trim().toLocaleLowerCase()
  return tasks.value.filter((task) => {
    const matchesKeyword =
      !searchText ||
      task.title.toLocaleLowerCase().includes(searchText) ||
      task.description.toLocaleLowerCase().includes(searchText)
    const matchesStatus = statusFilter.value === '全部' || task.status === statusFilter.value
    const matchesPriority =
      priorityFilter.value === '全部' || task.priority === priorityFilter.value
    return matchesKeyword && matchesStatus && matchesPriority
  })
})

const emptyDescription = computed(() =>
  tasks.value.length ? '没有符合条件的待办事项' : '还没有待办事项，点击右上角创建第一条吧',
)

function persistTasks() {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(tasks.value))
    return true
  } catch (error) {
    ElMessage.error(`待办保存失败：${error.message}`)
    return false
  }
}

function openCreateDialog() {
  editingId.value = null
  Object.assign(form, createEmptyForm())
  dialogVisible.value = true
}

function openEditDialog(task) {
  editingId.value = task.id
  Object.assign(form, task)
  dialogVisible.value = true
}

function submitForm() {
  if (!formRef.value || saving.value) return

  formRef.value.validate((valid) => {
    if (!valid) return

    saving.value = true
    const taskData = { ...form, title: form.title.trim(), description: form.description.trim() }
    let createdTaskId
    let previousTask
    if (editingId.value === null) {
      createdTaskId = Date.now()
      tasks.value.unshift({ id: createdTaskId, ...taskData })
    } else {
      const index = tasks.value.findIndex((task) => task.id === editingId.value)
      if (index === -1) {
        ElMessage.error('找不到要编辑的待办事项，请刷新后重试')
        saving.value = false
        return
      }
      previousTask = { ...tasks.value[index] }
      tasks.value[index] = { ...tasks.value[index], ...taskData }
    }

    if (persistTasks()) {
      ElMessage.success(editingId.value === null ? '待办已创建' : '待办已更新')
      dialogVisible.value = false
    } else if (editingId.value === null) {
      const index = tasks.value.findIndex((task) => task.id === createdTaskId)
      if (index !== -1) tasks.value.splice(index, 1)
    } else {
      const index = tasks.value.findIndex((task) => task.id === editingId.value)
      if (index !== -1) tasks.value[index] = previousTask
    }
    saving.value = false
  })
}

function completeTask(task) {
  const previousStatus = task.status
  task.status = '已完成'
  if (persistTasks()) {
    ElMessage.success('已标记为完成')
  } else {
    task.status = previousStatus
  }
}

async function removeTask(task) {
  try {
    await ElMessageBox.confirm(`确定删除「${task.title}」吗？删除后无法恢复。`, '删除待办', {
      confirmButtonText: '删除',
      cancelButtonText: '取消',
      type: 'warning',
    })
  } catch (error) {
    if (error === 'cancel' || error === 'close') return
    ElMessage.error(`删除确认失败：${error.message ?? error}`)
    return
  }

  const index = tasks.value.findIndex((item) => item.id === task.id)
  if (index === -1) return
  const [removedTask] = tasks.value.splice(index, 1)
  if (persistTasks()) {
    ElMessage.success('待办已删除')
  } else {
    tasks.value.splice(index, 0, removedTask)
  }
}

function isOverdue(task) {
  if (!task.dueDate || task.status === '已完成') return false
  const dueDate = new Date(`${task.dueDate}T23:59:59`)
  return dueDate < new Date()
}

function priorityType(priority) {
  return { 高: 'danger', 中: 'warning', 低: 'info' }[priority] ?? 'info'
}

function statusType(status) {
  return { 未开始: 'info', 进行中: 'primary', 已完成: 'success' }[status] ?? 'info'
}
</script>

<style scoped>
.todo-page {
  display: flex;
  flex-direction: column;
  gap: var(--st-gap-base);
}

.page-heading {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--st-gap-base);
}

.stats-row {
  margin-bottom: calc(var(--st-gap-base) * -1);
}

.stats-card {
  margin-bottom: var(--st-gap-base);
  border: none;
}

.stats-card__label {
  color: var(--st-text-secondary);
  font-size: 13px;
}

.stats-card__value {
  margin-top: 8px;
  font-size: 26px;
  font-weight: 600;
  line-height: 1;
}

.stats-card__value--primary {
  color: var(--st-primary);
}

.stats-card__value--warning {
  color: var(--st-warning);
}

.stats-card__value--info {
  color: var(--st-info);
}

.stats-card__value--success {
  color: var(--st-success);
}

.todo-card {
  border: none;
}

.toolbar {
  display: flex;
  gap: 12px;
  margin-bottom: 4px;
}

.toolbar__search {
  max-width: 360px;
}

.toolbar__priority {
  width: 150px;
}

.status-tabs :deep(.el-tabs__header) {
  margin-bottom: 8px;
}

.task-content {
  display: flex;
  flex-direction: column;
  gap: 5px;
  padding: 4px 0;
}

.task-content__title {
  font-weight: 500;
  color: var(--st-text-primary);
}

.task-content__title.is-done {
  color: var(--st-text-secondary);
  text-decoration: line-through;
}

.task-content__description {
  overflow: hidden;
  color: var(--st-text-secondary);
  font-size: 12px;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.due-date--overdue {
  color: var(--st-danger);
}

.overdue-tag {
  margin-left: 6px;
}

.form-field {
  width: 100%;
}

@media (max-width: 600px) {
  .page-heading {
    align-items: flex-start;
  }

  .page-heading :deep(.el-button) {
    flex-shrink: 0;
  }

  .toolbar {
    flex-direction: column;
  }

  .toolbar__search,
  .toolbar__priority {
    width: 100%;
    max-width: none;
  }
}
</style>
