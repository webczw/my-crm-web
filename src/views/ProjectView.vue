<script lang="ts" setup>
import { reactive, ref, onMounted } from 'vue'
import axios from 'axios'
import { ElLoading,ElMessage, ElMessageBox, ElTable } from 'element-plus'
import { View } from '@element-plus/icons-vue'


const tableData = reactive({projectList:[]})

const dataSize = ref('1')

// 表格引用，用于多选功能
const multipleTableRef = ref<InstanceType<typeof ElTable>>()

// 选中的项目
const multipleSelection = ref([])

// 弹窗相关
const dialogFormVisible = ref(false)
const dialogTitle = ref('')
const formType = ref('') // 'add' 或 'edit'

// 查看弹窗相关
const viewDialogVisible = ref(false)
const viewData = reactive({
  projectCode: '',
  projectName: '',
  planStartDate: '',
  planEndDate: '',
  projectAddress: ''
})

// 表单引用
const projectFormRef = ref()

// 表单数据
const projectForm = reactive({
  projectCode: '',
  projectName: '',
  planStartDate: '',
  planEndDate: '',
  projectAddress: ''
})

// 表单验证规则
const rules = reactive({
  projectCode: [
    { required: true, message: '请输入项目编码', trigger: 'blur' },
    { min: 1, max: 50, message: '项目编码长度应在1-50之间', trigger: 'blur' }
  ],
  projectName: [
    { required: true, message: '请输入项目名称', trigger: 'blur' },
    { min: 1, max: 100, message: '项目名称长度应在1-100之间', trigger: 'blur' }
  ],
  planStartDate: [
    { required: true, message: '请选择计划开始日期', trigger: 'change' }
  ],
  planEndDate: [
    { required: true, message: '请选择计划结束日期', trigger: 'change' }
  ]
})

// 当前编辑的项目索引
const currentEditIndex = ref(-1)

const handleChange = (value: number) => {
  console.log(value)
}

// 搜索函数
function search(){

  const loading = ElLoading.service({
    lock: true,
    text: '加载中...',
    background: 'rgba(0, 0, 0, 0.2)',
  })

  
    axios({
      method: 'post',
      url: '/ui-api/my-crm/project/queryList',
      headers: {
      'Content-Type': 'application/json'
      },
      data: {
        dataSize: dataSize.value
      }
    }).then(res => {
        if(res.data.data){
          tableData.projectList = res.data.data
        }
        loading.close()
    }, err => {
        loading.close()
        ElMessage({
          showClose: true,
          message: err.message,
          type: 'error',
        })
    })
}

// 选择行变化
const handleSelectionChange = (val: any) => {
  multipleSelection.value = val
}

// 打开新增项目弹窗
const openAddDialog = () => {
  formType.value = 'add'
  dialogTitle.value = '新增项目'
  // 重置表单
  Object.assign(projectForm, {
    projectCode: '',
    projectName: '',
    planStartDate: '',
    planEndDate: '',
    projectAddress: ''
  })
  dialogFormVisible.value = true
}

// 打开编辑项目弹窗
const openEditDialog = (row: any, index: number) => {
  formType.value = 'edit'
  dialogTitle.value = '修改项目'
  currentEditIndex.value = index
  // 填充表单数据
  Object.assign(projectForm, row)
  dialogFormVisible.value = true
}

// 打开查看项目弹窗
const openViewDialog = (row: any) => {
  // 填充查看数据
  Object.assign(viewData, row)
  viewDialogVisible.value = true
}

// 删除项目
const deleteProject = async (row: any, index: number) => {
  try {
    await ElMessageBox.confirm(
      '确定要删除这个项目吗？',
      '警告',
      {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning',
      }
    )
    
    // 这里调用删除API
    const response = await axios({
      method: 'post',
      url: '/ui-api/my-crm/project/delete',
      headers: {
        'Content-Type': 'application/json'
      },
      data: {
        projectCode: row.projectCode // 假设用项目编码作为唯一标识
      }
    })
    
    if(response.data.code === 200) { // 假设返回200表示成功
      tableData.projectList.splice(index, 1)
      ElMessage({
        type: 'success',
        message: '删除成功',
      })
    } else {
      ElMessage({
        type: 'error',
        message: response.data.message || '删除失败',
      })
    }
  } catch (error) {
    // 用户取消删除
  }
}

// 批量删除项目
const batchDeleteProjects = async () => {
  if (multipleSelection.value.length === 0) {
    ElMessage({
      type: 'warning',
      message: '请先选择要删除的项目',
    })
    return
  }
  
  try {
    await ElMessageBox.confirm(
      `确定要删除这${multipleSelection.value.length}个项目吗？`,
      '警告',
      {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning',
      }
    )
    
    // 收集要删除的项目编码
    const projectCodes = multipleSelection.value.map((item: any) => item.projectCode)
    
    // 调用批量删除API
    const response = await axios({
      method: 'post',
      url: '/ui-api/my-crm/project/batchDelete',
      headers: {
        'Content-Type': 'application/json'
      },
      data: {
        projectCodes: projectCodes
      }
    })
    
    if(response.data.code === 200) { // 假设返回200表示成功
      // 从列表中移除已删除的项目
      multipleSelection.value.forEach((item: any) => {
        const index = tableData.projectList.findIndex((p: any) => p.projectCode === item.projectCode)
        if (index > -1) {
          tableData.projectList.splice(index, 1)
        }
      })
      ElMessage({
        type: 'success',
        message: '批量删除成功',
      })
    } else {
      ElMessage({
        type: 'error',
        message: response.data.message || '批量删除失败',
      })
    }
  } catch (error) {
    // 用户取消删除
  }
}

// 提交表单
const submitForm = async () => {
  // 验证表单
  const valid = await projectFormRef.value.validate().catch(() => false)
  if (!valid) {
    return
  }
  
  try {
    let response
    if (formType.value === 'add') {
      // 新增项目API
      response = await axios({
        method: 'post',
        url: '/ui-api/my-crm/project/add',
        headers: {
          'Content-Type': 'application/json'
        },
        data: projectForm
      })
    } else {
      // 修改项目API
      response = await axios({
        method: 'post',
        url: '/ui-api/my-crm/project/update',
        headers: {
          'Content-Type': 'application/json'
        },
        data: projectForm
      })
    }
    
    if(response.data.code === 200) { // 假设返回200表示成功
      if (formType.value === 'add') {
        // 添加到列表开头
        tableData.projectList.unshift(response.data.data || projectForm)
        ElMessage({
          type: 'success',
          message: '新增成功',
        })
      } else {
        // 更新列表中的数据
        Object.assign(tableData.projectList[currentEditIndex.value], projectForm)
        ElMessage({
          type: 'success',
          message: '修改成功',
        })
      }
      dialogFormVisible.value = false
    } else {
      ElMessage({
        type: 'error',
        message: response.data.message || (formType.value === 'add' ? '新增失败' : '修改失败'),
      })
    }
  } catch (error) {
    ElMessage({
      type: 'error',
      message: formType.value === 'add' ? '新增失败' : '修改失败',
    })
  }
}

onMounted(() => {
  search()
})
</script>

<template>
  <el-row :gutter="10" style="margin-bottom: 10px;">
    <el-col :span="4">
      <el-input v-model="dataSize" placeholder="请输入数据量"/>
    </el-col>
    <el-col :span="20">
      <el-button type="primary" @click="search">搜索</el-button>
      <el-button type="primary" @click="openAddDialog">新增项目</el-button>
      <el-button 
        type="danger" 
        :disabled="multipleSelection.length === 0" 
        @click="batchDeleteProjects"
      >
        批量删除
      </el-button>
    </el-col>
  </el-row>
  <el-table 
    ref="multipleTableRef"
    :data="tableData.projectList" 
    style="width: 100%" 
    max-height="500" 
    empty-text="无数据" 
    border
    @selection-change="handleSelectionChange"
  >
    <el-table-column type="selection" width="55" />
    <el-table-column prop="projectCode" label="项目编码" width="300" />
    <el-table-column prop="projectName" label="项目名称" width="220" />
    <el-table-column prop="planStartDate" label="计划开始日期" width="180" />
    <el-table-column prop="planEndDate" label="计划结束日期" width="180" />
    <el-table-column prop="projectAddress" label="项目地址" />
    <el-table-column label="操作" width="250">
      <template #default="{ row, $index }">
        <el-button size="small" :icon="View" @click="openViewDialog(row)" title="查看"></el-button>
        <el-button size="small" @click="openEditDialog(row, $index)">修改</el-button>
        <el-button size="small" type="danger" @click="deleteProject(row, $index)">删除</el-button>
      </template>
    </el-table-column>
  </el-table>
  
  <!-- 新增/修改项目弹窗 -->
  <el-dialog v-model="dialogFormVisible" :title="dialogTitle" width="600px">
    <el-form :model="projectForm" :rules="rules" ref="projectFormRef" label-width="120px">
      <el-form-item label="项目编码" prop="projectCode">
        <el-input v-model="projectForm.projectCode" :disabled="formType === 'edit'" />
      </el-form-item>
      <el-form-item label="项目名称" prop="projectName">
        <el-input v-model="projectForm.projectName" />
      </el-form-item>
      <el-form-item label="计划开始日期" prop="planStartDate">
        <el-date-picker
          v-model="projectForm.planStartDate"
          type="date"
          placeholder="选择日期"
          format="YYYY-MM-DD"
          value-format="YYYY-MM-DD"
          style="width: 100%"
        />
      </el-form-item>
      <el-form-item label="计划结束日期" prop="planEndDate">
        <el-date-picker
          v-model="projectForm.planEndDate"
          type="date"
          placeholder="选择日期"
          format="YYYY-MM-DD"
          value-format="YYYY-MM-DD"
          style="width: 100%"
        />
      </el-form-item>
      <el-form-item label="项目地址">
        <el-input v-model="projectForm.projectAddress" />
      </el-form-item>
    </el-form>
    <template #footer>
      <span class="dialog-footer">
        <el-button @click="dialogFormVisible = false">取消</el-button>
        <el-button type="primary" @click="submitForm">确定</el-button>
      </span>
    </template>
  </el-dialog>

  <!-- 查看项目弹窗 -->
  <el-dialog v-model="viewDialogVisible" title="查看项目" width="600px">
    <el-form :model="viewData" label-width="120px">
      <el-form-item label="项目编码">
        <el-input v-model="viewData.projectCode" disabled />
      </el-form-item>
      <el-form-item label="项目名称">
        <el-input v-model="viewData.projectName" disabled />
      </el-form-item>
      <el-form-item label="计划开始日期">
        <el-input v-model="viewData.planStartDate" disabled />
      </el-form-item>
      <el-form-item label="计划结束日期">
        <el-input v-model="viewData.planEndDate" disabled />
      </el-form-item>
      <el-form-item label="项目地址">
        <el-input v-model="viewData.projectAddress" disabled />
      </el-form-item>
    </el-form>
    <template #footer>
      <span class="dialog-footer">
        <el-button type="primary" @click="viewDialogVisible = false">关闭</el-button>
      </span>
    </template>
  </el-dialog>
</template>
