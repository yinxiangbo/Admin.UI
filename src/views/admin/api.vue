<template>
  <section style="padding:10px;">
    <!--查询-->
    <el-form class="ad-form-query" :inline="true" :model="filters" @submit.native.prevent>
      <el-form-item>
        <el-input v-model="filters.label" placeholder="接口名或地址" @keyup.enter.native="onGetList">
          <template #prefix>
            <i class="el-input__icon el-icon-search" />
          </template>
        </el-input>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" @click="onGetList">查询</el-button>
      </el-form-item>
      <el-form-item v-if="checkPermission(['api:admin:api:add'])">
        <el-button type="primary" @click="onAdd">新增</el-button>
      </el-form-item>
      <el-form-item v-if="checkPermission(['api:admin:api:sync'])">
        <my-confirm-button
          :icon="'el-icon-refresh'"
          :placement="'bottom-end'"
          :loading="syncLoading"
          style="margin:0px;"
          @click="onSync"
        >
          <template #content>
            <p>确定要同步Api吗？</p>
          </template>
          同步Api
        </my-confirm-button>
      </el-form-item>
      <!-- <el-form-item>
        <el-button type="primary" icon="el-icon-s-operation" @click="onGenerate">生成前端Api</el-button>
      </el-form-item> -->
      <el-form-item v-if="checkPermission(['api:admin:api:batchsoftdelete'])">
        <my-confirm-button
          :disabled="sels.length === 0"
          :type="'delete'"
          :placement="'bottom-end'"
          :loading="deleteLoading"
          style="margin-left: 0px;"
          @click="onBatchDelete"
        >
          <template #content>
            <p>确定要批量删除吗？</p>
          </template>
          批量删除
        </my-confirm-button>
      </el-form-item>
    </el-form>

    <!--列表-->
    <el-table
      ref="multipleTable"
      v-loading="listLoading"
      row-key="id"
      :data="apiTree"
      :default-expand-all="false"
      :tree-props="{ children: 'children', hasChildren: 'hasChildren' }"
      highlight-current-row
      style="width: 100%;"
      @select-all="onSelectAll"
      @select="onSelect"
    >
      <el-table-column type="selection" width="50" />
      <el-table-column prop="label" label="接口名" width="180" />
      <el-table-column prop="path" label="接口地址" width />
      <el-table-column prop="description" label="接口描述" width />
      <!-- <el-table-column prop="createTime" label="创建时间" :formatter="formatCreatedTime" width="120" >
            </el-table-column>
            <el-table-column prop="createUserName" label="创建者" width="100" >
      </el-table-column>-->
      <el-table-column prop="enabled" label="状态" width="100">
        <template #default="{row}">
          <el-tag
            :type="row.enabled ? 'success' : 'info'"
            disable-transitions
          >{{ row.enabled ? '正常' : '禁用' }}</el-tag>
        </template>
      </el-table-column>
      <el-table-column v-if="checkPermission(['api:admin:api:update','api:admin:api:softdelete'])" label="操作" width="180">
        <template #default="{ $index, row }">
          <el-button v-if="checkPermission(['api:admin:api:update'])" @click="onEdit($index, row)">编辑</el-button>
          <my-confirm-button v-if="checkPermission(['api:admin:api:softdelete'])" type="delete" :loading="row._loading" @click="onDelete($index, row)" />
        </template>
      </el-table-column>
    </el-table>

    <!--新增窗口-->
    <my-window
      v-if="checkPermission(['api:admin:api:add'])"
      title="新增"
      :visible.sync="addFormVisible"
      @close="onCloseAddForm"
    >
      <el-form ref="addForm" :model="addForm" label-width="80px" :rules="addFormRules">
        <el-form-item prop="parentIds" label="上级接口">
          <el-cascader
            :key="addFormKey"
            v-model="addForm.parentIds"
            placeholder="请选择，支持搜索功能"
            :options="modules"
            :props="{ checkStrictly: true, value: 'id' }"
            filterable
            style="width:100%;"
          />
        </el-form-item>
        <el-form-item label="接口名" prop="label">
          <el-input v-model="addForm.label" auto-complete="off" />
        </el-form-item>
        <el-form-item label="接口地址" prop="path">
          <el-input v-model="addForm.path" auto-complete="off" />
        </el-form-item>
        <el-form-item label="接口方法" prop="httpMethods">
          <el-radio-group v-model="addForm.httpMethods">
            <el-radio-button label="get" />
            <el-radio-button label="put" />
            <el-radio-button label="post" />
            <el-radio-button label="patch" />
            <el-radio-button label="delete" />
          </el-radio-group>
        </el-form-item>
        <el-form-item label="启用" prop="enabled">
          <el-switch v-model="addForm.enabled" />
        </el-form-item>
        <el-form-item label="说明" prop="description">
          <el-input v-model="addForm.description" type="textarea" rows="2" auto-complete="off" />
        </el-form-item>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button @click.native="addFormVisible = false">取消</el-button>
          <my-confirm-button type="submit" :validate="addFormValidate" :loading="addLoading" @click="onAddSubmit" />
        </div>
      </template>
    </my-window>

    <!--编辑窗口-->
    <my-window
      v-if="checkPermission(['api:admin:api:update'])"
      title="编辑"
      :visible.sync="editFormVisible"
      @close="onCloseEditForm"
    >
      <el-form ref="editForm" :model="editForm" :rules="editFormRules" label-width="80px">
        <el-form-item prop="parentIds" label="上级接口">
          <el-cascader
            :key="editFormKey"
            v-model="editForm.parentIds"
            placeholder="请选择，支持搜索功能"
            :options="modules"
            :props="{ checkStrictly: true, value: 'id' }"
            filterable
            style="width:100%;"
          />
        </el-form-item>
        <el-form-item label="接口名" prop="label">
          <el-input v-model="editForm.label" auto-complete="off" />
        </el-form-item>
        <el-form-item label="接口地址" prop="path">
          <el-input v-model="editForm.path" auto-complete="off" />
        </el-form-item>
        <el-form-item label="接口方法" prop="httpMethods">
          <el-radio-group v-model="editForm.httpMethods">
            <el-radio-button label="get" />
            <el-radio-button label="put" />
            <el-radio-button label="post" />
            <el-radio-button label="patch" />
            <el-radio-button label="delete" />
          </el-radio-group>
        </el-form-item>
        <el-form-item label="启用" prop="enabled">
          <el-switch v-model="editForm.enabled" />
        </el-form-item>
        <el-form-item label="说明" prop="description">
          <el-input v-model="editForm.description" type="textarea" rows="2" auto-complete="off" />
        </el-form-item>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button @click.native="editFormVisible = false">取消</el-button>
          <my-confirm-button type="submit" :validate="editFormValidate" :loading="editLoading" @click="onEditSubmit" />
        </div>
      </template>
    </my-window>
  </section>
</template>

<script>
import { formatTime, treeToList, listToTree, getTreeParents } from '@/utils'
import { removeApi, editApi, addApi, getV2SwaggerJson, syncApi, getApiList, batchRemoveApi, getApi } from '@/api/admin/api'
import MyWindow from '@/components/my-window'
import MyConfirmButton from '@/components/my-confirm-button'

export default {
  name: 'Api',
  components: { MyWindow, MyConfirmButton },
  data() {
    return {
      filters: {
        label: ''
      },
      apiTree: [],
      listLoading: false,
      sels: [], // 列表选中列

      addDialogFormVisible: false,
      editFormVisible: false, // 编辑界面是否显示
      editLoading: false,
      editFormRules: {
        parentIds: [{ required: true, message: '请选择上级接口', trigger: 'change' }],
        path: [{ required: true, message: '请输入接口地址', trigger: 'blur' }],
        label: [{ required: true, message: '请输入接口名', trigger: 'blur' }]
      },
      // 编辑界面数据
      editForm: {
        id: 0,
        parentIds: [],
        path: '',
        label: '',
        httpMethods: '',
        enabled: false,
        description: ''
      },
      editFormKey: 1,

      addFormVisible: false, // 新增界面是否显示
      addLoading: false,
      addFormRules: {
        parentIds: [{ required: true, message: '请选择上级接口', trigger: 'change' }],
        path: [{ required: true, message: '请输入接口地址', trigger: 'blur' }],
        label: [{ required: true, message: '请输入接口名', trigger: 'blur' }]
      },
      // 新增界面数据
      addForm: {
        parentIds: [],
        path: '',
        label: '',
        httpMethods: '',
        enabled: true,
        description: ''
      },
      addFormKey: 1,
      modules: [],
      syncLoading: false,
      deleteLoading: false
    }
  },
  mounted() {
    this.onGetList()
  },
  methods: {
    formatCreatedTime: function(row, column, time) {
      return formatTime(time, 'YYYY-MM-DD HH:mm')
    },
    // 获取列表
    async onGetList() {
      const para = {
        key: this.filters.label
      }
      this.listLoading = true
      const res = await getApiList(para)
      this.listLoading = false

      if (!res?.success) {
        return
      }

      const list = _.cloneDeep(res.data)
      const parentModules = list.filter(l => l.parentId === 0)
      this.modules = listToTree(_.cloneDeep(parentModules), {
        id: 0,
        parentId: 0,
        label: '顶级'
      })

      list.forEach(l => {
        l._loading = false
      })
      const tree = listToTree(list)
      this.sels = []
      this.apiTree = tree
    },
    // 显示编辑界面
    async onEdit(index, row) {
      const loading = this.$loading()
      const res = await getApi({ id: row.id })
      loading.close()
      if (res && res.success) {
        const parents = getTreeParents(this.apiTree, row.id)
        const parentIds = parents.map(p => p.id)
        parentIds.unshift(0)

        const data = res.data
        data.parentIds = parentIds
        this.editForm = data
        this.editFormVisible = true
        ++this.editFormKey
      }
    },
    onCloseEditForm() {
      this.$refs.editForm.resetFields()
      ++this.editFormKey
    },
    // 显示新增界面
    onAdd() {
      this.addFormVisible = true
    },
    onCloseAddForm() {
      this.$refs.addForm.resetFields()
      ++this.addFormKey
    },
    // 编辑
    editFormValidate: function() {
      let isValid = false
      this.$refs.editForm.validate(valid => {
        isValid = valid
      })
      return isValid
    },
    async onEditSubmit() {
      this.editLoading = true
      const para = _.cloneDeep(this.editForm)
      para.parentId = para.parentIds.pop()
      if (para.id === para.parentId) {
        this.$message({
          message: '上级接口不能是自己！',
          type: 'error'
        })
        this.editLoading = false
        return
      }

      const res = await editApi(para)
      this.editLoading = false
      if (!res?.success) {
        return
      }
      this.$message({
        message: this.$t('admin.updateOk'),
        type: 'success'
      })
      this.$refs['editForm'].resetFields()
      this.editFormVisible = false
      this.onGetList()
    },
    // 新增
    addFormValidate: function() {
      let isValid = false
      this.$refs.addForm.validate(valid => {
        isValid = valid
      })
      return isValid
    },
    async onAddSubmit() {
      this.addLoading = true
      const para = _.cloneDeep(this.addForm)
      para.parentId = para.parentIds.pop()

      const res = await addApi(para)
      this.addLoading = false

      if (!res?.success) {
        return
      }
      this.$message({
        message: this.$t('admin.addOk'),
        type: 'success'
      })
      this.$refs['addForm'].resetFields()
      this.addFormVisible = false
      this.onGetList()
    },
    // 删除
    async onDelete(index, row) {
      row._loading = true
      const para = { id: row.id }
      const res = await removeApi(para)

      row._loading = false

      if (!res?.success) {
        return
      }
      this.$message({
        message: this.$t('admin.deleteOk'),
        type: 'success'
      })
      this.onGetList()
    },
    // 批量删除
    async onBatchDelete() {
      const para = { ids: [] }
      para.ids = this.sels.map(s => {
        return s.id
      })

      this.deleteLoading = true
      const res = await batchRemoveApi(para.ids)
      this.deleteLoading = false

      if (!res?.success) {
        return
      }
      this.$message({
        message: this.$t('admin.batchDeleteOk'),
        type: 'success'
      })

      this.onGetList()
    },
    // 同步api
    async onSync() {
      this.syncLoading = true
      const res = await getV2SwaggerJson()

      if (!res) {
        this.syncLoading = false
        return
      }

      const tags = res.tags
      const paths = res.paths

      const apis = []
      // tags
      if (tags && tags.length > 0) {
        tags.forEach(t => {
          apis[apis.length] = {
            label: t.description,
            path: t.name
          }
        })
      }
      // paths
      if (paths) {
        for (const [key, value] of Object.entries(paths)) {
          const keys = Object.keys(value)
          const values = Object.values(value)
          const v = values && values.length > 0 ? values[0] : {}
          const parentPath = v.tags && v.tags.length > 0 ? v.tags[0] : ''
          apis[apis.length] = {
            label: v.summary,
            path: key,
            parentPath,
            httpMethods: keys.join(',')
          }
        }
      }

      const syncRes = await syncApi({ apis })
      this.syncLoading = false

      if (!syncRes?.success) {
        return
      }
      this.$message({
        message: this.$t('api.sync'),
        type: 'success'
      })
      this.onGetList()
    },
    // 生成前端api
    onGenerate() {
      // let bb = this.sels
    },
    onSelectAll: function(selection) {
      const selections = treeToList(selection)
      const rows = treeToList(this.apiTree)
      const checked = selections.length === rows.length
      rows.forEach(row => {
        this.$refs.multipleTable.toggleRowSelection(row, checked)
      })

      this.sels = this.$refs.multipleTable.selection
    },
    onSelect: function(selection, row) {
      const checked = selection.some(s => s.id === row.id)
      if (row.children && row.children.length > 0) {
        const rows = treeToList(row.children)
        rows.forEach(row => {
          this.$refs.multipleTable.toggleRowSelection(row, checked)
        })
      }

      this.sels = this.$refs.multipleTable.selection
    }
  }
}
</script>
