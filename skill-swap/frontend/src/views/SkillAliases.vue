<template>
  <div class="skill-aliases">
    <div class="card">
      <div class="page-header">
        <div>
          <h1 class="page-title">🏷️ 技能别名管理</h1>
          <p class="page-desc">维护技能标准名与别名，统一聚合统计、搜索与匹配，避免 Vue / vue.js / Vue3 被拆分成多个技能</p>
        </div>
        <el-button type="primary" size="large" @click="openCreateDialog">
          <el-icon><Plus /></el-icon>新增标准名
        </el-button>
      </div>

      <div class="filter-bar">
        <el-input
          v-model="filters.keyword"
          placeholder="搜索标准名或别名"
          style="width: 260px"
          clearable
          :prefix-icon="Search"
          @input="loadAliases"
        />
        <el-select
          v-model="filters.category"
          placeholder="按类别筛选"
          clearable
          style="width: 180px"
          @change="loadAliases"
        >
          <el-option
            v-for="cat in categories"
            :key="cat.id"
            :label="`${cat.icon} ${cat.name}`"
            :value="cat.id"
          />
        </el-select>
        <el-button @click="loadAliases">
          <el-icon><Refresh /></el-icon>刷新
        </el-button>
      </div>

      <div v-loading="loading" class="aliases-table-wrap">
        <el-table :data="aliases" stripe style="width: 100%">
          <el-table-column type="index" label="#" width="60" align="center" />
          <el-table-column label="标准名" min-width="140">
            <template #default="{ row }">
              <span class="std-name">{{ row.standardName }}</span>
            </template>
          </el-table-column>
          <el-table-column label="类别" width="130">
            <template #default="{ row }">
              <el-tag :type="getCategoryTagType(row.category)" effect="light">
                {{ getCategoryLabel(row.category) }}
              </el-tag>
            </template>
          </el-table-column>
          <el-table-column label="别名列表" min-width="300">
            <template #default="{ row }">
              <div class="alias-tags">
                <el-tag
                  v-for="al in row.aliases.slice(0, 8)"
                  :key="al"
                  size="small"
                  type="info"
                  effect="plain"
                  class="alias-tag"
                >
                  {{ al }}
                </el-tag>
                <el-tooltip
                  v-if="row.aliases.length > 8"
                  :content="row.aliases.slice(8).join('、')"
                  placement="top"
                >
                  <el-tag size="small" type="info" effect="plain" class="alias-tag more">
                    +{{ row.aliases.length - 8 }} 更多
                  </el-tag>
                </el-tooltip>
                <span v-if="!row.aliases || row.aliases.length === 0" class="empty-text">
                  暂无别名
                </span>
              </div>
            </template>
          </el-table-column>
          <el-table-column label="别名数" width="90" align="center">
            <template #default="{ row }">
              <el-badge :value="row.aliases?.length || 0" class="item" :max="99" />
            </template>
          </el-table-column>
          <el-table-column label="操作" width="160" align="center" fixed="right">
            <template #default="{ row }">
              <el-button link type="primary" size="small" @click="openEditDialog(row)">编辑</el-button>
              <el-popconfirm
                title="确定删除该标准名及其所有别名吗？"
                confirm-button-text="删除"
                cancel-button-text="取消"
                @confirm="deleteAlias(row)"
              >
                <template #reference>
                  <el-button link type="danger" size="small">删除</el-button>
                </template>
              </el-popconfirm>
            </template>
          </el-table-column>
        </el-table>

        <el-empty v-if="!loading && aliases.length === 0" description="暂无别名数据" />
      </div>
    </div>

    <el-dialog
      v-model="dialogVisible"
      :title="isEdit ? '编辑技能别名' : '新增技能标准名'"
      width="620px"
      :close-on-click-modal="false"
    >
      <el-form :model="form" :rules="rules" ref="formRef" label-width="90px">
        <el-form-item label="标准名" prop="standardName">
          <el-input v-model="form.standardName" placeholder="例如：Vue、Python、英语" size="large" />
        </el-form-item>
        <el-form-item label="所属类别" prop="category">
          <el-select v-model="form.category" placeholder="选择类别" size="large" style="width: 100%">
            <el-option
              v-for="cat in categories"
              :key="cat.id"
              :label="`${cat.icon} ${cat.name}`"
              :value="cat.id"
            />
          </el-select>
        </el-form-item>
        <el-form-item label="别名列表" prop="aliases">
          <div class="alias-input-wrap">
            <div class="alias-input-row">
              <el-input
                v-model="newAlias"
                placeholder="输入别名后按回车添加，例如：vue.js、Vue3、VUE"
                size="large"
                style="flex: 1"
                @keyup.enter="addAlias"
              />
              <el-button type="primary" size="large" @click="addAlias">添加</el-button>
            </div>
            <div class="alias-tags-edit" v-if="form.aliases.length">
              <el-tag
                v-for="(al, idx) in form.aliases"
                :key="idx"
                closable
                type="primary"
                effect="light"
                @close="removeAlias(idx)"
              >
                {{ al }}
              </el-tag>
            </div>
            <div class="alias-tip">
              <el-icon><InfoFilled /></el-icon>
              <span>别名用于匹配用户各种写法，匹配时不区分大小写、空格和点号</span>
            </div>
          </div>
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="dialogVisible = false">取消</el-button>
        <el-button type="primary" :loading="submitting" @click="submitForm">确认提交</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted } from 'vue'
import { skillAPI } from '../api'
import { ElMessage, ElMessageBox } from 'element-plus'
import {
  Plus, Search, Refresh, InfoFilled
} from '@element-plus/icons-vue'

const loading = ref(false)
const submitting = ref(false)
const dialogVisible = ref(false)
const isEdit = ref(false)
const editId = ref(null)
const aliases = ref([])
const categories = ref([])
const formRef = ref(null)
const newAlias = ref('')

const filters = reactive({
  keyword: '',
  category: ''
})

const form = reactive({
  standardName: '',
  category: '',
  aliases: []
})

const rules = {
  standardName: [{ required: true, message: '请输入标准名', trigger: 'blur' }],
  category: [{ required: true, message: '请选择类别', trigger: 'change' }],
  aliases: [
    {
      type: 'array',
      required: true,
      min: 1,
      message: '请至少添加一个别名',
      trigger: 'change'
    }
  ]
}

onMounted(async () => {
  await Promise.all([loadCategories(), loadAliases()])
})

async function loadCategories() {
  const res = await skillAPI.getCategories()
  categories.value = res.data
}

async function loadAliases() {
  loading.value = true
  try {
    const params = {}
    if (filters.keyword.trim()) params.keyword = filters.keyword.trim()
    if (filters.category) params.category = filters.category
    const res = await skillAPI.getAliases(params)
    aliases.value = res.data
  } catch (e) {
    ElMessage.error('加载别名列表失败')
  } finally {
    loading.value = false
  }
}

function openCreateDialog() {
  isEdit.value = false
  editId.value = null
  Object.assign(form, {
    standardName: '',
    category: '',
    aliases: []
  })
  newAlias.value = ''
  dialogVisible.value = true
}

function openEditDialog(row) {
  isEdit.value = true
  editId.value = row.id
  Object.assign(form, {
    standardName: row.standardName,
    category: row.category,
    aliases: [...row.aliases]
  })
  newAlias.value = ''
  dialogVisible.value = true
}

function addAlias() {
  const val = newAlias.value.trim()
  if (!val) return
  if (form.aliases.includes(val)) {
    ElMessage.warning('该别名已存在')
    return
  }
  form.aliases.push(val)
  newAlias.value = ''
}

function removeAlias(idx) {
  form.aliases.splice(idx, 1)
}

async function submitForm() {
  try {
    await formRef.value.validate()
    submitting.value = true
    if (isEdit.value) {
      await skillAPI.updateAlias(editId.value, { ...form })
      ElMessage.success('更新成功')
    } else {
      await skillAPI.createAlias({ ...form })
      ElMessage.success('创建成功')
    }
    dialogVisible.value = false
    await loadAliases()
  } catch (e) {
    if (e?.message) ElMessage.error(e.message)
  } finally {
    submitting.value = false
  }
}

async function deleteAlias(row) {
  try {
    await skillAPI.deleteAlias(row.id)
    ElMessage.success('删除成功')
    await loadAliases()
  } catch (e) {
    ElMessage.error('删除失败')
  }
}

function getCategoryLabel(id) {
  const cat = categories.value.find(c => c.id === id)
  return cat ? `${cat.icon} ${cat.name}` : id
}

const categoryTagColors = {
  programming: '',
  language: 'success',
  music: 'warning',
  design: 'danger',
  cooking: '',
  fitness: 'success',
  business: 'warning',
  photo: 'info',
  writing: '',
  other: 'info'
}

function getCategoryTagType(id) {
  return categoryTagColors[id] || ''
}
</script>

<style scoped>
.skill-aliases {
  display: flex;
  flex-direction: column;
  gap: 24px;
}

.page-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 24px;
}

.page-title {
  font-size: 24px;
  font-weight: 700;
  color: #333;
  margin: 0 0 8px;
}

.page-desc {
  color: #888;
  font-size: 14px;
  margin: 0;
}

.filter-bar {
  display: flex;
  gap: 12px;
  margin-bottom: 20px;
  flex-wrap: wrap;
}

.aliases-table-wrap {
  min-height: 200px;
}

.std-name {
  font-weight: 600;
  font-size: 15px;
  color: #333;
}

.alias-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.alias-tag {
  margin-right: 0 !important;
}

.alias-tag.more {
  cursor: pointer;
}

.empty-text {
  color: #bbb;
  font-size: 13px;
}

.alias-input-wrap {
  width: 100%;
}

.alias-input-row {
  display: flex;
  gap: 8px;
  margin-bottom: 12px;
}

.alias-tags-edit {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 10px;
}

.alias-tip {
  display: flex;
  align-items: flex-start;
  gap: 6px;
  font-size: 12px;
  color: #909399;
  padding: 8px 12px;
  background: #fafafa;
  border-radius: 6px;
}

.alias-tip .el-icon {
  margin-top: 2px;
  flex-shrink: 0;
}
</style>
