<script setup>
import { onActivated, ref, watch } from 'vue'
import { storeToRefs } from 'pinia'
import { ElMessage, ElMessageBox } from 'element-plus'
import { listRuns, bulkDeleteRuns } from '@/api/register'
import { fmtTime } from '@/api/request'
import { useRuntimeStore } from '@/stores/runtime'
import StatusDot from '@/components/StatusDot.vue'

const { dataVersion } = storeToRefs(useRuntimeStore())
const rows = ref([])
const selected = ref([])
const loading = ref(false)
const deleting = ref(false)

const STATUS_TYPE = { done: 'primary', failed: 'danger', running: 'warning' }

async function load() {
  loading.value = true
  try { const { items } = await listRuns(200); rows.value = items; selected.value = [] }
  catch (e) { ElMessage.error(e.message) }
  finally { loading.value = false }
}

async function confirm(msg) {
  try {
    await ElMessageBox.confirm(msg, '确认', {
      type: 'warning', confirmButtonText: '确定', cancelButtonText: '取消',
      customClass: 'confirm-multiline',
    })
    return true
  } catch { return false }
}

async function deleteSelected() {
  const ids = selected.value.map((r) => r.run_id)
  if (!ids.length) return
  if (!(await confirm(`确定删除选中的 ${ids.length} 条运行记录？（含日志文件，不可恢复）`))) return
  deleting.value = true
  try { const r = await bulkDeleteRuns({ run_ids: ids }); ElMessage.success(`已删除 ${r.deleted} 条`); load() }
  catch (e) { ElMessage.error(e.message) }
  finally { deleting.value = false }
}

async function deleteAll() {
  if (!rows.value.length) { ElMessage.info('暂无记录'); return }
  if (!(await confirm(`将清空全部 ${rows.value.length} 条运行记录（含日志文件），不可恢复，确定？`))) return
  if (!(await confirm('再次确认：真的要清空全部运行记录吗？'))) return
  deleting.value = true
  try { const r = await bulkDeleteRuns({ all: true }); ElMessage.success(`已清空 ${r.deleted} 条`); load() }
  catch (e) { ElMessage.error(e.message) }
  finally { deleting.value = false }
}

watch(dataVersion, () => load())
onActivated(() => load())
</script>

<template>
  <div class="page">
    <el-card shadow="never">
      <template #header>
        <div style="display: flex; align-items: center; justify-content: space-between; gap: 12px; flex-wrap: wrap">
          <span class="section-title" style="margin: 0">运行记录</span>
          <el-space wrap>
            <el-button size="small" @click="load"><el-icon><Refresh /></el-icon>刷新</el-button>
            <el-button size="small" type="danger" plain :loading="deleting" :disabled="!selected.length" @click="deleteSelected">删除选中 ({{ selected.length }})</el-button>
            <el-button size="small" type="danger" :loading="deleting" :disabled="!rows.length" @click="deleteAll">清空全部</el-button>
            <span class="hint" v-if="rows.length">共 {{ rows.length }} 条 · 选中 {{ selected.length }}</span>
          </el-space>
        </div>
      </template>
      <el-skeleton v-if="loading && !rows.length" :rows="6" animated style="padding: 8px 0" />
      <el-table v-else v-loading="loading" :data="rows" size="small" stripe @selection-change="(v) => (selected = v)">
        <el-table-column type="selection" width="44" />
        <el-table-column prop="run_id" label="run_id" width="180">
          <template #default="{ row }"><span class="mono">{{ row.run_id }}</span></template>
        </el-table-column>
        <el-table-column prop="email" label="邮箱" min-width="200" show-overflow-tooltip />
        <el-table-column label="状态" width="100">
          <template #default="{ row }">
            <StatusDot :type="STATUS_TYPE[row.status] || 'info'" :text="row.status" />
          </template>
        </el-table-column>
        <el-table-column label="开始时间" width="170">
          <template #default="{ row }">{{ fmtTime(row.started_at) }}</template>
        </el-table-column>
        <el-table-column prop="error" label="错误" min-width="200" show-overflow-tooltip />
        <template #empty>
          <el-empty description="暂无运行记录" :image-size="70" />
        </template>
      </el-table>
    </el-card>
  </div>
</template>
