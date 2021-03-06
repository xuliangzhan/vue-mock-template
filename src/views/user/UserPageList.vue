<template>
  <div class="home">

    <vxe-toolbar>
      <template v-slot:buttons>
        <vxe-button @click="insertEvent()">新增</vxe-button>
        <vxe-button @click="pendingRemoveEvent()">标记/取消删除</vxe-button>
        <vxe-button @click="deleteListEvent()">删除选中</vxe-button>
        <vxe-button @click="saveEvent()">保存</vxe-button>
        <vxe-button @click="$refs.xTable.exportCsv()">导出数据</vxe-button>
      </template>
    </vxe-toolbar>

    <vxe-table
      ref="xTable"
      class="vxe-table-element user-table"
      stripe
      border
      height="444"
      highlight-hover-row
      :loading="loading"
      :data="list"
      :row-class-name="tableRowClassName"
      :edit-config="{key: 'id', trigger: 'click', mode: 'row'}"
      :edit-rules="validRules">
      <vxe-table-column
        type="seq"
        width="55">
      </vxe-table-column>
      <vxe-table-column
        field="id"
        title="ID"
        width="80">
      </vxe-table-column>
      <vxe-table-column
        field="name"
        title="姓名"
        show-overflow
        :edit-render="{name: 'ElInput', attrs: {placeholder: '请输入姓名！'}}">
      </vxe-table-column>
      <vxe-table-column
        field="age"
        title="年龄"
        align="center"
        :edit-render="{name: 'ElInputNumber', attrs: {placeholder: '请输入年龄！'}}">
      </vxe-table-column>
      <vxe-table-column
        field="email"
        title="邮箱"
        :edit-render="{name: 'ElInput', attrs: {placeholder: '请输入邮箱！'}}">
      </vxe-table-column>
      <vxe-table-column
        field="createTime"
        title="创建日期"
        :formatter="formatColumnDate">
      </vxe-table-column>
      <vxe-table-column
        field="updateTime"
        title="最后更新时间"
        :formatter="formatColumnDate">
      </vxe-table-column>
      <vxe-table-column
        field="describe"
        title="备注"
        show-overflow
        :edit-render="{name: 'ElInput'}">
      </vxe-table-column>
    </vxe-table>

    <vxe-pager
      :loading="loading"
      :current-page="pageVO.currentPage"
      :page-sizes="[5, 10, 15, 20, 50, 100, 200, 500, 1000, 2000, 5000]"
      :page-size="pageVO.pageSize"
      :total="pageVO.totalResult"
      @size-change="handleSizeChange"
      @current-change="handleCurrentChange">
    </vxe-pager>
  </div>
</template>

<script>
import XEUtils from 'xe-utils'
import XEAjax from 'xe-ajax'
import { MessageBox, Message } from 'element-ui'

export default {
  data () {
    return {
      loading: false,
      list: [],
      pendingRemoveList: [],
      pageVO: {
        currentPage: 1,
        pageSize: 10,
        totalResult: 0
      },
      validRules: {
        name: [
          { required: true, message: '请写姓名！', trigger: 'change' },
          { max: 30, message: '最多30个字符！', trigger: 'change' }
        ],
        age: [
          { required: true, message: '请写年龄！', trigger: 'change' },
          { type: 'number', min: 18, max: 35, message: '年龄必须在18-35之间！', trigger: 'change' }
        ],
        email: [
          { pattern: /^[A-Za-z\d]+([-_.][A-Za-z\d]+)*@([A-Za-z\d]+[-.])+[A-Za-z\d]{2,4}$/, message: '邮箱格式错误！', trigger: 'change' }
        ],
        describe: [
          { max: 200, message: '最多100个字符！', trigger: 'change' }
        ]
      }
    }
  },
  created () {
    this.findList()
  },
  methods: {
    handleSizeChange (pageSize) {
      this.pageVO.pageSize = pageSize
      this.findList()
    },
    handleCurrentChange (currentPage) {
      this.pageVO.currentPage = currentPage
      this.findList()
    },
    findList () {
      this.loading = true
      this.pendingRemoveList = []
      XEAjax.getJSON(`/api/user/page/list/${this.pageVO.pageSize}/${this.pageVO.currentPage}`).then(({ page, result }) => {
        this.pageVO.totalResult = page.totalResult
        this.list = result
        this.loading = false
      }).catch(e => {
        this.loading = false
      })
    },
    formatColumnDate ({ row, column, cellValue }) {
      return XEUtils.toDateString(cellValue)
    },
    tableRowClassName ({ row, rowIndex }) {
      if (this.pendingRemoveList.some(item => item === row)) {
        return 'delete-row'
      }
      return ''
    },
    insertEvent () {
      this.$refs.xTable.insert({ age: '28' }).then(({ row }) => {
        this.$refs.xTable.validate(row).catch(e => e)
      })
    },
    pendingRemoveEvent () {
      let selection = this.$refs.xTable.getSelectRecords()
      if (selection.length) {
        let plus = []
        let minus = []
        selection.forEach(data => {
          if (this.pendingRemoveList.some(item => data === item)) {
            minus.push(data)
          } else {
            plus.push(data)
          }
        })
        if (minus.length) {
          this.pendingRemoveList = this.pendingRemoveList.filter(item => minus.some(data => data !== item)).concat(plus)
        } else if (plus) {
          this.pendingRemoveList = this.pendingRemoveList.concat(plus)
        }
        this.$refs.xTable.clearSelection()
      } else {
        Message({
          type: 'info',
          message: '请至少选择一条数据！'
        })
      }
    },
    deleteListEvent () {
      let selection = this.$refs.xTable.getSelectRecords()
      if (selection.length) {
        MessageBox.confirm('此操作将永久删除该文件, 是否继续?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.loading = true
          XEAjax.postJSON('/api/user/save', { removeRecords: selection }).then(data => {
            Message({
              type: 'success',
              message: '删除成功!'
            })
            this.findList()
          }).catch(e => {
            this.loading = false
          })
        }).catch(() => {
          Message({
            type: 'info',
            message: '已取消删除'
          })
        })
      } else {
        Message({
          type: 'info',
          message: '请至少选择一条数据！'
        })
      }
    },
    saveEvent () {
      this.$refs.xTable.validate(valid => {
        if (valid) {
          let removeRecords = this.pendingRemoveList
          let { insertRecords, updateRecords } = this.$refs.xTable.getRecordset()
          if (insertRecords.length || removeRecords.length || updateRecords.length) {
            this.loading = true
            XEAjax.postJSON('/api/user/save', { insertRecords, removeRecords, updateRecords }).then(data => {
              Message({
                type: 'success',
                message: '保存成功!'
              })
              this.findList()
            }).catch(e => {
              this.loading = false
            })
          } else {
            Message({
              type: 'info',
              message: '数据未改动！'
            })
          }
        }
      })
    }
  }
}
</script>

<style>
.user-table .delete-row {
  color: #f56c6c;
  text-decoration: line-through;
}
</style>
