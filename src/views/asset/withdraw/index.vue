<template>
  <div class="app-container">
    <div class="filter-container">
      <el-button v-waves class="filter-item" type="success" icon="el-icon-plus" @click="handleApplyWithdraw">
        申请提币
      </el-button>
      <el-input v-model="listQuery.txId" placeholder="交易ID" style="width: 200px;" class="filter-item" @keyup.enter.native="handleFilter" />
      <el-select v-model="listQuery.currency" placeholder="币种" clearable style="width: 120px" class="filter-item">
        <el-option v-for="item in currencyOptions" :key="item.key" :label="item.display_name" :value="item.key" />
      </el-select>
      <el-select v-model="listQuery.status" placeholder="状态" clearable style="width: 120px" class="filter-item">
        <el-option v-for="item in statusOptions" :key="item.key" :label="item.display_name" :value="item.key" />
      </el-select>
      <el-date-picker
        v-model="dateRange"
        type="daterange"
        align="right"
        unlink-panels
        range-separator="至"
        start-placeholder="开始日期"
        end-placeholder="结束日期"
        :picker-options="pickerOptions"
        class="filter-item"
        style="width: 350px"
      />
      <el-button v-waves class="filter-item" type="primary" icon="el-icon-search" @click="handleFilter">
        搜索
      </el-button>
      <el-button v-waves :loading="downloadLoading" class="filter-item" type="primary" icon="el-icon-download" @click="handleDownload">
        导出
      </el-button>
    </div>

    <el-table
      :key="tableKey"
      v-loading="listLoading"
      :data="list"
      border
      fit
      highlight-current-row
      style="width: 100%;"
    >
      <el-table-column label="交易ID" align="center" width="180">
        <template slot-scope="{row}">
          <span>{{ row.txId }}</span>
        </template>
      </el-table-column>
      <el-table-column label="币种" width="100" align="center">
        <template slot-scope="{row}">
          <span>{{ row.currency }}</span>
        </template>
      </el-table-column>
      <el-table-column label="数量" width="120" align="center">
        <template slot-scope="{row}">
          <span>{{ row.amount }}</span>
        </template>
      </el-table-column>
      <el-table-column label="手续费" width="120" align="center">
        <template slot-scope="{row}">
          <span>{{ row.fee }}</span>
        </template>
      </el-table-column>
      <el-table-column label="提币地址" min-width="220" align="center">
        <template slot-scope="{row}">
          <el-tooltip class="item" effect="dark" :content="row.address" placement="top-start">
            <span class="address-text">{{ row.address }}</span>
          </el-tooltip>
        </template>
      </el-table-column>
      <el-table-column label="状态" width="100" align="center">
        <template slot-scope="{row}">
          <el-tag :type="row.status | statusFilter">
            {{ row.statusText }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column label="创建时间" width="180" align="center">
        <template slot-scope="{row}">
          <span>{{ row.createTime | parseTime }}</span>
        </template>
      </el-table-column>
      <el-table-column label="完成时间" width="180" align="center">
        <template slot-scope="{row}">
          <span>{{ row.status === 'completed' ? (row.completeTime | parseTime) : '-' }}</span>
        </template>
      </el-table-column>
      <el-table-column label="操作" align="center" width="120" class-name="small-padding fixed-width">
        <template slot-scope="{row}">
          <el-button type="primary" size="mini" @click="handleDetail(row)">
            详情
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination v-show="total>0" :total="total" :page.sync="listQuery.page" :limit.sync="listQuery.limit" @pagination="getList" />

    <el-dialog title="提币详情" :visible.sync="detailDialogVisible" width="50%">
      <el-descriptions :column="2" border>
        <el-descriptions-item label="交易ID">{{ currentWithdraw.txId }}</el-descriptions-item>
        <el-descriptions-item label="币种">{{ currentWithdraw.currency }}</el-descriptions-item>
        <el-descriptions-item label="数量">{{ currentWithdraw.amount }}</el-descriptions-item>
        <el-descriptions-item label="手续费">{{ currentWithdraw.fee }}</el-descriptions-item>
        <el-descriptions-item label="状态">
          <el-tag :type="currentWithdraw.status | statusFilter">{{ currentWithdraw.statusText }}</el-tag>
        </el-descriptions-item>
        <el-descriptions-item label="审核状态">
          <el-tag :type="currentWithdraw.auditStatus | auditStatusFilter">{{ currentWithdraw.auditStatusText }}</el-tag>
        </el-descriptions-item>
        <el-descriptions-item label="提币地址" :span="2">{{ currentWithdraw.address }}</el-descriptions-item>
        <el-descriptions-item label="区块高度" v-if="currentWithdraw.blockHeight">{{ currentWithdraw.blockHeight }}</el-descriptions-item>
        <el-descriptions-item label="区块哈希" v-if="currentWithdraw.blockHash" :span="2">{{ currentWithdraw.blockHash }}</el-descriptions-item>
        <el-descriptions-item label="创建时间">{{ currentWithdraw.createTime | parseTime }}</el-descriptions-item>
        <el-descriptions-item label="审核时间" v-if="currentWithdraw.auditTime">{{ currentWithdraw.auditTime | parseTime }}</el-descriptions-item>
        <el-descriptions-item label="完成时间" v-if="currentWithdraw.completeTime">{{ currentWithdraw.completeTime | parseTime }}</el-descriptions-item>
        <el-descriptions-item label="拒绝原因" v-if="currentWithdraw.rejectReason" :span="2">{{ currentWithdraw.rejectReason }}</el-descriptions-item>
        <el-descriptions-item label="备注" :span="2">{{ currentWithdraw.remark || '-' }}</el-descriptions-item>
      </el-descriptions>
      <span slot="footer" class="dialog-footer">
        <el-button @click="detailDialogVisible = false">关闭</el-button>
      </span>
    </el-dialog>
    <el-dialog title="申请提币" :visible.sync="applyDialogVisible" width="50%">
      <el-form ref="applyForm" :model="applyForm" :rules="applyRules" label-width="120px">
        <el-form-item label="币种" prop="currency">
          <el-select v-model="applyForm.currency" placeholder="请选择币种">
            <el-option v-for="item in currencyOptions" :key="item.key" :label="item.display_name" :value="item.key" />
          </el-select>
        </el-form-item>
        <el-form-item label="数量" prop="amount">
          <el-input v-model="applyForm.amount" placeholder="请输入数量" />
        </el-form-item>
        <el-form-item label="提币地址" prop="address">
          <el-input v-model="applyForm.address" placeholder="请输入提币地址" />
        </el-form-item>
        <el-form-item label="交易密码" prop="password">
          <el-input v-model="applyForm.password" placeholder="请输入交易密码" type="password" />
        </el-form-item>
        <el-form-item label="验证码" prop="verificationCode">
          <el-input v-model="applyForm.verificationCode" placeholder="请输入验证码" />
          <el-button type="primary" @click="getCode" :disabled="codeCooldown > 0">{{ codeCooldown > 0 ? `重新发送(${codeCooldown}s)` : '获取验证码' }}</el-button>
        </el-form-item>
        <el-form-item label="备注" prop="remark">
          <el-input v-model="applyForm.remark" placeholder="请输入备注" />
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="applyDialogVisible = false">取消</el-button>
        <el-button type="primary" @click="handleApplyWithdrawConfirm" :loading="applyLoading">确定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
import waves from '@/directive/waves' // waves directive
import { parseTime } from '@/utils'
import Pagination from '@/components/Pagination' // secondary package based on el-pagination

export default {
  name: 'AssetWithdraw',
  components: { Pagination },
  directives: { waves },
  filters: {
    statusFilter(status) {
      const statusMap = {
        'pending': 'info',
        'processing': 'warning',
        'completed': 'success',
        'rejected': 'danger',
        'failed': 'danger'
      }
      return statusMap[status]
    },
    auditStatusFilter(status) {
      const statusMap = {
        'pending': 'info',
        'approved': 'success',
        'rejected': 'danger'
      }
      return statusMap[status]
    },
    parseTime
  },
  data() {
    return {
      tableKey: 0,
      list: null,
      total: 0,
      listLoading: true,
      listQuery: {
        page: 1,
        limit: 10,
        txId: undefined,
        currency: undefined,
        status: undefined,
        startTime: undefined,
        endTime: undefined
      },
      dateRange: [],
      pickerOptions: {
        shortcuts: [{
          text: '最近一周',
          onClick(picker) {
            const end = new Date()
            const start = new Date()
            start.setTime(start.getTime() - 3600 * 1000 * 24 * 7)
            picker.$emit('pick', [start, end])
          }
        }, {
          text: '最近一个月',
          onClick(picker) {
            const end = new Date()
            const start = new Date()
            start.setTime(start.getTime() - 3600 * 1000 * 24 * 30)
            picker.$emit('pick', [start, end])
          }
        }, {
          text: '最近三个月',
          onClick(picker) {
            const end = new Date()
            const start = new Date()
            start.setTime(start.getTime() - 3600 * 1000 * 24 * 90)
            picker.$emit('pick', [start, end])
          }
        }]
      },
      currencyOptions: [
        { key: 'BTC', display_name: 'BTC' },
        { key: 'ETH', display_name: 'ETH' },
        { key: 'USDT', display_name: 'USDT' },
        { key: 'LTC', display_name: 'LTC' },
        { key: 'EOS', display_name: 'EOS' }
      ],
      statusOptions: [
        { key: 'pending', display_name: '待审核' },
        { key: 'processing', display_name: '处理中' },
        { key: 'completed', display_name: '已完成' },
        { key: 'rejected', display_name: '已拒绝' },
        { key: 'failed', display_name: '失败' }
      ],
      detailDialogVisible: false,
      currentWithdraw: {},
      applyDialogVisible: false,
      applyForm: {
        currency: '',
        address: '',
        amount: 0,
        password: '',
        verificationCode: '',
        remark: ''
      },
      applyRules: {
        currency: [{ required: true, message: '请选择币种', trigger: 'change' }],
        address: [{ required: true, message: '请输入提币地址', trigger: 'blur' }],
        amount: [{ required: true, message: '请输入提币数量', trigger: 'blur' }],
        password: [{ required: true, message: '请输入交易密码', trigger: 'blur' }],
        verificationCode: [{ required: true, message: '请输入验证码', trigger: 'blur' }]
      },
      codeCooldown: 0,
      applyLoading: false,
      downloadLoading: false
    }
  },
  watch: {
    dateRange(val) {
      if (val) {
        this.listQuery.startTime = val[0]
        this.listQuery.endTime = val[1]
      } else {
        this.listQuery.startTime = undefined
        this.listQuery.endTime = undefined
      }
    }
  },
  created() {
    this.getList()
  },
  methods: {
    getList() {
      this.listLoading = true
      // 模拟API调用
      setTimeout(() => {
        this.list = this.generateMockData()
        this.total = this.list.length
        this.listLoading = false
      }, 500)
    },
    handleFilter() {
      this.listQuery.page = 1
      this.getList()
    },
    handleDetail(row) {
      this.currentWithdraw = Object.assign({}, row)
      this.detailDialogVisible = true
    },
    handleDownload() {
      this.downloadLoading = true
      import('@/vendor/Export2Excel').then(excel => {
        const tHeader = ['交易ID', '币种', '数量', '手续费', '提币地址', '状态', '审核状态', '创建时间', '完成时间']
        const filterVal = ['txId', 'currency', 'amount', 'fee', 'address', 'statusText', 'auditStatusText', 'createTime', 'completeTimeText']
        const data = this.formatJson(filterVal)
        excel.export_json_to_excel({
          header: tHeader,
          data,
          filename: '提币记录'
        })
        this.downloadLoading = false
      })
    },
    formatJson(filterVal) {
      return this.list.map(v => filterVal.map(j => {
        if (j === 'createTime' || j === 'completeTimeText') {
          if (j === 'completeTimeText') {
            return v.status === 'completed' ? parseTime(v.completeTime) : '-'
          }
          return parseTime(v[j.replace('Text', '')])
        } else {
          return v[j] || '-'
        }
      }))
    },
    generateMockData() {
      const mockData = []
      const statusTextMap = {
        'pending': '待审核',
        'processing': '处理中',
        'completed': '已完成',
        'rejected': '已拒绝',
        'failed': '失败'
      }
      
      const auditStatusTextMap = {
        'pending': '待审核',
        'approved': '已通过',
        'rejected': '已拒绝'
      }
      
      // 生成20条模拟数据
      for (let i = 1; i <= 20; i++) {
        const currency = this.currencyOptions[Math.floor(Math.random() * this.currencyOptions.length)].key
        const amount = (Math.random() * 10).toFixed(currency === 'BTC' ? 8 : 4)
        const fee = (amount * 0.001).toFixed(currency === 'BTC' ? 8 : 4)
        const status = this.statusOptions[Math.floor(Math.random() * this.statusOptions.length)].key
        
        // 审核状态
        let auditStatus = 'pending'
        if (status === 'completed' || status === 'processing') {
          auditStatus = 'approved'
        } else if (status === 'rejected') {
          auditStatus = 'rejected'
        }
        
        // 生成随机地址
        let address = ''
        if (currency === 'BTC') {
          address = `bc1q${this.generateRandomString(38)}`
        } else if (currency === 'ETH' || currency === 'USDT') {
          address = `0x${this.generateRandomString(40)}`
        } else if (currency === 'LTC') {
          address = `L${this.generateRandomString(33)}`
        } else {
          address = this.generateRandomString(34)
        }
        
        const createTime = new Date(Date.now() - Math.floor(Math.random() * 30) * 24 * 60 * 60 * 1000)
        const auditTime = auditStatus !== 'pending' ? new Date(createTime.getTime() + Math.floor(Math.random() * 24) * 60 * 60 * 1000) : null
        const completeTime = status === 'completed' ? new Date(auditTime.getTime() + Math.floor(Math.random() * 24) * 60 * 60 * 1000) : null
        
        mockData.push({
          id: i,
          txId: status !== 'pending' && status !== 'rejected' ? this.generateRandomString(64) : null,
          currency: currency,
          amount: amount,
          fee: fee,
          address: address,
          status: status,
          statusText: statusTextMap[status],
          auditStatus: auditStatus,
          auditStatusText: auditStatusTextMap[auditStatus],
          blockHeight: status === 'completed' ? 700000 + Math.floor(Math.random() * 10000) : null,
          blockHash: status === 'completed' ? this.generateRandomString(64) : null,
          rejectReason: status === 'rejected' ? '地址无效或不支持该币种' : null,
          remark: '',
          createTime: createTime,
          auditTime: auditTime,
          completeTime: completeTime
        })
      }
      
      return mockData
    },
    generateRandomString(length) {
      const chars = '0123456789abcdef'
      let result = ''
      for (let i = 0; i < length; i++) {
        result += chars.charAt(Math.floor(Math.random() * chars.length))
      }
      return result
    },
    handleApplyWithdraw() {
      this.applyDialogVisible = true
      // 重置表单
      if (this.$refs.applyForm) {
        this.$refs.applyForm.resetFields()
      }
      this.applyForm = {
        currency: 'BTC',
        address: '',
        amount: 0.001,
        password: '',
        verificationCode: '',
        remark: ''
      }
    },
    getCode() {
      if (this.codeCooldown > 0) return
      this.codeCooldown = 60
      const intervalId = setInterval(() => {
        this.codeCooldown--
        if (this.codeCooldown === 0) {
          clearInterval(intervalId)
        }
      }, 1000)
    },
    handleApplyWithdrawConfirm() {
      this.$refs.applyForm.validate((valid) => {
        if (valid) {
          this.applyLoading = true
          // TODO: 实现申请提币逻辑
          console.log('申请提币')
          setTimeout(() => {
            this.applyLoading = false
            this.applyDialogVisible = false
          }, 2000)
        }
      })
    }
  }
}
</script>

<style lang="scss" scoped>
.filter-container {
  padding-bottom: 10px;
  .filter-item {
    display: inline-block;
    vertical-align: middle;
    margin-bottom: 10px;
    margin-right: 10px;
  }
}

.address-text {
  display: inline-block;
  width: 200px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.el-tag {
  margin-right: 5px;
}
</style>
