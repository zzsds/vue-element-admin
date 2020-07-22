<template>
  <div class="app-container">
    <div class="content-top">
      <div class="content-topR">
        <el-button v-waves :loading="downloadLoading" size="small" class="filter-item" type="primary" icon="el-icon-download" @click="downloadOpen('user')">导出</el-button>
        <el-button v-waves type="primary" icon="el-icon-refresh" size="small" @click="getList" />
      </div>

      <div class="content-topL">
        <el-input v-model="searchMobile" size="small" class="search" placeholder="输入用户手机" clearable />
        <!-- <el-select v-model="searchSelect" placeholder="请选择">
          <el-option
            v-for="item in options"
            :key="item.value"
            :label="item.label"
            :value="item.value"
          />
        </el-select> -->
        <el-button v-waves type="primary" icon="el-icon-search" size="small" @click="handlerSearch">搜索</el-button>
        <el-button v-waves type="warning" icon="el-icon-refresh" size="small" @click="resetList">重置</el-button>
      </div>
    </div>

    <!-- 用户列表 -->
    <el-table
      v-loading="listLoading"
      :data="list"
      border
      fit
      highlight-current-row
      style="width: 100%;"
      stripe
    >
      <!-- <el-table-column type="expand">
        <template slot-scope="props">
          <el-form label-position="left" inline class="demo-table-expand">
            <el-form-item v-if="props.row.voucherText" label="申诉原因">
              <span>{{ props.row.voucherText || '' }}</span>
            </el-form-item>
            <el-form-item v-if="props.row.stateRemarks" label="审核备注">
              <span>{{ props.row.stateRemarks || '' }}</span>
            </el-form-item>
          </el-form>
        </template>
      </el-table-column> -->
      <el-table-column align="center" label="用户ID" prop="id" sortable />
      <el-table-column align="center" label="手机号" prop="mobile" />
      <el-table-column align="center" label="上级ID" prop="parentId" />
      <el-table-column align="center" label="上级手机" prop="parentMobile" />
      <el-table-column align="center" label="创建时间" prop="createdAt" />
      <el-table-column align="center" label="操作" prop="bank">
        <template slot-scope="scope">
          <el-button size="small" type="warning" @click="changeUser(scope.row)">修改</el-button>
        </template>
      </el-table-column>
    </el-table>
    <pagination :total="total" :page.sync="listQuery.page" :layout="'total, prev, pager, next, jumper'" @pagination="getList" />

    <el-dialog title="修改用户" class="review" :visible.sync="isDialog">
      <el-form label-position="left" inline class="demo-table-expand">
        <el-form-item label="手机号">
          <el-input v-model="handlerForm.mobile" placeholder="请填写手机号" />
        </el-form-item>
      </el-form>
      <el-form label-position="left" inline class="demo-table-expand">
        <el-form-item label="姓名">
          <el-input v-model="handlerForm.name" placeholder="请填写姓名" />
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button size="small" :loading="changeLoading" type="primary" @click="handleChange">提交</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
import dayjs from 'dayjs'
import waves from '@/directive/waves'
import { ranks, userList, changeUser } from '@/api/user'
import { getRankName } from '@/utils'
import Pagination from '@/components/Pagination'
export default {
  name: 'User',
  directives: { waves },
  components: { Pagination },
  data() {
    return {
      submitLoading: false,
      downloadLoading: false,
      listLoading: false,
      isDialog: false,
      changeLoading: false,

      listQuery: {
        page: 1,
        size: 10,
        sort: 'id desc'
      },
      total: 0,
      list: [],
      searchMobile: '',

      itemId: '',
      handlerForm: {
        mobile: '',
        name: ''
      },

      searchSelect: 8,
      options: [
        {
          value: 8,
          label: '全部'
        },
        {
          value: 0,
          label: '会员（v0）'
        },
        {
          value: 1,
          label: '体验达人（v1）'
        },
        {
          value: 2,
          label: '分享达人（v2）'
        },
        {
          value: 3,
          label: '资深达人（v3）'
        },
        {
          value: 4,
          label: '初级掌柜（v4）'
        },
        {
          value: 5,
          label: '高级掌柜（v5）'
        },
        {
          value: 6,
          label: '初级经销商（v6）'
        },
        {
          value: 7,
          label: '高级经销商（v7）'
        }
      ]
    }
  },
  computed: {
  },
  watch: {
  },
  created() {
    this.getList()
  },
  methods: {
    changeUser(item) {
      this.itemId = item.id
      this.handlerForm.mobile = item.mobile || ''
      this.handlerForm.name = item.name || ''
      this.isDialog = true
    },
    handleChange() {
      this.changeLoading = true
      changeUser(this.handlerForm, { id: this.itemId }).then(() => {
        this.changeLoading = false
        this.isDialog = false
        this.$message.success('修改成功')
        this.getList()
      }).catch(() => {
        this.changeLoading = false
      })
    },
    resetList() {
      this.searchMobile = ''
      this.getList()
    },
    handlerSearch() {
      this.listQuery.page = 1
      this.getList()
    },
    // 导出按钮
    downloadOpen() {
      this.$prompt('请输入需要导出的数量', 'Excel导出', {
        confirmButtonText: '确定',
        cancelButtonText: '取消'
      }).then(({ value }) => {
        this.$message.info('正在导出')
        this.download(value)
      }).catch(() => {
        this.$message.message('取消导出')
      })
    },
    // 导出用户列表
    async download(size) {
      this.downloadLoading = true
      if (!size || size < 10) size = 10

      try {
        const res = await userList({ page: 0, size })
        const tmp = res.data.map(x => x.parentId || false)
        tmp.forEach((item, i) => {
          if (item === false) {
            tmp.splice(i, 1)
          }
        })
        const conditions = await userList({ size: tmp.length, sort: 'id desc', condition: `id in (${tmp.toString()})` })
        const rank = await ranks()
        const data = res.data.map(x => {
          let rankName
          if (!x.rankId) {
            rankName = `${getRankName('v0')}（v0）`
          } else {
            const rankRes = rank.data && rank.data.filter(f => f.id === x.rankId)[0]
            rankName = `${getRankName(rankRes.name)}（${rankRes.name}）`
          }

          const index = conditions.data && conditions.data.findIndex(o => o.id === x.parentId)
          return {
            ...x,
            parentMobile: index > -1 ? conditions.data[index].mobile : '',
            rankName: rankName,
            createdAt: dayjs(x.createdAt).format('YYYY-MM-DD HH:mm:ss')
          }
        })
        this.handleDownload(data, 'User-data')
      } catch (error) {
        console.log(error)
        this.downloadLoading = false
        this.$message.error('导出失败')
      }
    },
    async getList() {
      this.listLoading = true
      const query = JSON.parse(JSON.stringify(this.listQuery))
      query.page -= 1
      try {
        let searchRank
        if (this.searchSelect === 8) searchRank = ''
        else searchRank = `condition=rankId=${this.searchSelect}`

        const res = await userList({ ...query, ...{ condition: this.searchMobile ? `mobile=${this.searchMobile}` : '' }}, searchRank)
        this.total = res.page.total || 0
        if (!res.data || res.data.lengt < 1) {
          this.list = []
          this.listLoading = false
          return false
        }
        const tmp = res.data.map(x => x.parentId || false)
        tmp.forEach((item, i) => {
          if (item === false) {
            tmp.splice(i, 1)
          }
        })
        const conditions = await userList({ size: tmp.length, condition: `id in (${tmp.toString()})` })
        const rank = await ranks()
        const data = res.data.map(x => {
          let rankName
          if (!x.rankId) {
            rankName = `${getRankName('v0')}（v0）`
          } else {
            const rankRes = rank.data && rank.data.filter(f => f.id === x.rankId)[0]
            rankName = `${getRankName(rankRes.name)}（${rankRes.name}）`
          }

          const index = conditions.data && conditions.data.findIndex(o => o.id === x.parentId)
          return { ...x,
            ...{
              parentMobile: index > -1 ? conditions.data[index].mobile : '',
              rankName: rankName,
              createdAt: dayjs(x.createdAt).format('YYYY-MM-DD HH:mm:ss')
            }
          }
        })
        this.list = data
        this.listLoading = false
      } catch (error) {
        this.listLoading = false
        console.log(error)
      }
    },
    // 导出excel表格
    handleDownload(downLoadData, excelName) {
      import('@/vendor/Export2Excel').then(excel => {
        const tHeader = ['用户Id', '手机号', '上级ID', '邀请人', '创建时间']
        const filterVal = ['id', 'mobile', 'parentId', 'parentMobile', 'createdAt']
        const data = this.formatJson(filterVal, downLoadData)
        excel.export_json_to_excel({
          header: tHeader,
          data,
          filename: excelName
        })
        this.$message.success('导出成功')
        this.downloadLoading = false
      })
    },
    formatJson(filterVal, jsonData) {
      return jsonData.map(v => filterVal.map(j => {
        return v[j]
      }))
    }
  }

}
</script>

<style lang="scss">

</style>
