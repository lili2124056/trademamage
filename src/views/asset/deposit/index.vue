<template>
  <div class="app-container">
    <div class="filter-container">
      <el-button v-waves class="filter-item" type="success" icon="el-icon-plus" @click="handleAddDeposit">
        添加存币
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
      <el-table-column label="确认数" width="100" align="center">
        <template slot-scope="{row}">
          <span>{{ row.confirmations }}/{{ row.requiredConfirmations }}</span>
        </template>
      </el-table-column>
      <el-table-column label="地址" min-width="220" align="center">
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

    <el-dialog title="存币详情" :visible.sync="detailDialogVisible" width="50%">
      <el-descriptions :column="2" border>
        <el-descriptions-item label="交易ID">{{ currentDeposit.txId }}</el-descriptions-item>
        <el-descriptions-item label="币种">{{ currentDeposit.currency }}</el-descriptions-item>
        <el-descriptions-item label="数量">{{ currentDeposit.amount }}</el-descriptions-item>
        <el-descriptions-item label="确认数">{{ currentDeposit.confirmations }}/{{ currentDeposit.requiredConfirmations }}</el-descriptions-item>
        <el-descriptions-item label="状态">
          <el-tag :type="currentDeposit.status | statusFilter">{{ currentDeposit.statusText }}</el-tag>
        </el-descriptions-item>
        <el-descriptions-item label="手续费">{{ currentDeposit.fee || '0' }}</el-descriptions-item>
        <el-descriptions-item label="地址" :span="2">{{ currentDeposit.address }}</el-descriptions-item>
        <el-descriptions-item label="区块高度" v-if="currentDeposit.blockHeight">{{ currentDeposit.blockHeight }}</el-descriptions-item>
        <el-descriptions-item label="区块哈希" v-if="currentDeposit.blockHash" :span="2">{{ currentDeposit.blockHash }}</el-descriptions-item>
        <el-descriptions-item label="创建时间">{{ currentDeposit.createTime | parseTime }}</el-descriptions-item>
        <el-descriptions-item label="完成时间">{{ currentDeposit.status === 'completed' ? (currentDeposit.completeTime | parseTime) : '-' }}</el-descriptions-item>
        <el-descriptions-item label="备注" :span="2">{{ currentDeposit.remark || '-' }}</el-descriptions-item>
      </el-descriptions>
      <span slot="footer" class="dialog-footer">
        <el-button @click="detailDialogVisible = false">关闭</el-button>
      </span>
    </el-dialog>

    <el-dialog title="添加存币" :visible.sync="addDepositDialogVisible" width="50%">
      <el-form :model="addDepositForm" :rules="addDepositRules" ref="addDepositForm" label-width="100px">
        <el-form-item label="币种" prop="currency">
          <el-select v-model="addDepositForm.currency" placeholder="请选择币种" @change="refreshQrCode">
            <el-option v-for="item in currencyOptions" :key="item.key" :label="item.display_name" :value="item.key" />
          </el-select>
        </el-form-item>
        <el-form-item label="网络" v-if="addDepositForm.currency === 'ETH' || addDepositForm.currency === 'USDT'">
          <el-select v-model="addDepositForm.network" placeholder="请选择网络">
            <el-option v-for="item in networkOptions[addDepositForm.currency]" :key="item" :label="item" :value="item" />
          </el-select>
        </el-form-item>
        <el-form-item label="地址">
          <el-input v-model="depositAddress" readonly placeholder="地址" />
          <el-button type="primary" size="mini" @click="copyAddress">复制</el-button>
        </el-form-item>
        <el-form-item label="二维码">
          <img :src="qrCodeUrl" alt="二维码" style="width: 100px; height: 100px;">
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="addDepositDialogVisible = false">取消</el-button>
        <el-button type="primary" @click="handleAddDepositSubmit">确定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
import waves from '@/directive/waves' // waves directive
import { parseTime } from '@/utils'
import Pagination from '@/components/Pagination' // secondary package based on el-pagination

export default {
  name: 'AssetDeposit',
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
        { key: 'USDT', display_name: 'USDT (ERC20)' },
        { key: 'USDT_TRC20', display_name: 'USDT (TRC20)' },
        { key: 'LTC', display_name: 'LTC' },
        { key: 'EOS', display_name: 'EOS' },
        { key: 'SOL', display_name: 'SOL' },
        { key: 'TORN', display_name: 'TORN' }
      ],
      statusOptions: [
        { key: 'pending', display_name: '待处理' },
        { key: 'processing', display_name: '处理中' },
        { key: 'completed', display_name: '已完成' },
        { key: 'rejected', display_name: '已拒绝' },
        { key: 'failed', display_name: '失败' }
      ],
      detailDialogVisible: false,
      currentDeposit: {},
      addDepositDialogVisible: false,
      addDepositForm: {
        currency: '',
        network: ''
      },
      addDepositRules: {
        currency: [{ required: true, message: '请选择币种', trigger: 'change' }]
      },
      depositAddress: '',
      qrCodeUrl: '',
      confirmationTime: {
        'BTC': 6,
        'ETH': 12,
        'USDT': 12,
        'USDT_TRC20': 20,
        'LTC': 6,
        'EOS': 12,
        'SOL': 32,
        'TORN': 12
      },
      minDeposit: {
        'BTC': 0.0001,
        'ETH': 0.01,
        'USDT': 1,
        'USDT_TRC20': 1,
        'LTC': 0.01,
        'EOS': 0.1,
        'SOL': 0.01,
        'TORN': 0.1
      },
      networkOptions: {
        'ETH': ['ETH', 'BSC'],
        'USDT': ['ERC20', 'TRC20', 'BSC'],
        'SOL': ['Solana']
      },
      chainInfo: {
        'BTC': { name: 'Bitcoin', addressPrefix: '1', addressLength: 34 },
        'ETH': { name: 'Ethereum', addressPrefix: '0x', addressLength: 42 },
        'USDT': { name: 'Ethereum (ERC20)', addressPrefix: '0x', addressLength: 42 },
        'USDT_TRC20': { name: 'Tron (TRC20)', addressPrefix: 'T', addressLength: 34 },
        'LTC': { name: 'Litecoin', addressPrefix: 'L', addressLength: 34 },
        'EOS': { name: 'EOS', addressPrefix: '', addressLength: 12 },
        'SOL': { name: 'Solana', addressPrefix: '', addressLength: 44 },
        'TORN': { name: 'Ethereum (ERC20)', addressPrefix: '0x', addressLength: 42 }
      },
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
      this.currentDeposit = Object.assign({}, row)
      this.detailDialogVisible = true
    },
    handleAddDeposit() {
      this.addDepositDialogVisible = true
      this.addDepositForm.currency = ''
      this.addDepositForm.network = ''
      this.depositAddress = ''
      this.qrCodeUrl = ''
      this.$nextTick(() => {
        if (this.$refs.addDepositForm) {
          this.$refs.addDepositForm.resetFields()
        }
      })
    },
    generateQrCode() {
      if (!this.addDepositForm.currency) return
      
      this.qrCodeUrl = ''
      this.depositAddress = ''
      
      // 模拟生成地址的延迟
      setTimeout(() => {
        const currency = this.addDepositForm.currency
        const chainData = this.chainInfo[currency]
        
        if (currency === 'BTC') {
          this.depositAddress = this.generateBtcAddress()
        } else if (currency === 'ETH' || currency === 'TORN') {
          this.depositAddress = this.generateEthAddress()
        } else if (currency === 'USDT') {
          this.depositAddress = this.generateEthAddress() // ERC20地址
        } else if (currency === 'USDT_TRC20') {
          this.depositAddress = this.generateTronAddress()
        } else if (currency === 'LTC') {
          this.depositAddress = this.generateLtcAddress()
        } else if (currency === 'EOS') {
          this.depositAddress = this.generateEosAccount()
        } else if (currency === 'SOL') {
          this.depositAddress = this.generateSolAddress()
        }
        
        // 生成二维码URL，实际项目中可能需要调用后端API
        this.qrCodeUrl = `https://example.com/qr-code/${this.addDepositForm.currency}/${this.depositAddress}`
      }, 1000)
    },
    refreshQrCode() {
      this.generateQrCode()
    },
    copyAddress() {
      this.$copyText(this.depositAddress).then(() => {
        this.$message({
          message: '复制成功',
          type: 'success'
        })
      })
    },
    getConfirmationTime(currency) {
      return this.confirmationTime[currency] || 12
    },
    getMinDeposit(currency) {
      return this.minDeposit[currency] || 0.01
    },
    getNetworkName(currency) {
      if (currency === 'USDT_TRC20') return 'Tron (TRC20)'
      return this.chainInfo[currency]?.name || currency
    },
    // 生成不同格式的地址
    generateBtcAddress() {
      return '1' + this.generateRandomString(33, '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz')
    },
    generateEthAddress() {
      return '0x' + this.generateRandomString(40, '0123456789abcdef')
    },
    generateTronAddress() {
      return 'T' + this.generateRandomString(33, '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz')
    },
    generateLtcAddress() {
      return 'L' + this.generateRandomString(33, '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz')
    },
    generateEosAccount() {
      return this.generateRandomString(12, '12345abcdefghijklmnopqrstuvwxyz')
    },
    generateSolAddress() {
      return this.generateRandomString(44, '123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz')
    },
    generateRandomString(length, chars) {
      chars = chars || '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'
      let result = ''
      for (let i = 0; i < length; i++) {
        result += chars.charAt(Math.floor(Math.random() * chars.length))
      }
      return result
    },
    handleAddDepositSubmit() {
      this.$refs.addDepositForm.validate((valid) => {
        if (valid) {
          // 模拟API调用
          setTimeout(() => {
            this.addDepositDialogVisible = false
            this.$message({
              message: '添加存币成功',
              type: 'success'
            })
          }, 500)
        }
      })
    },
    handleDownload() {
      this.downloadLoading = true
      import('@/vendor/Export2Excel').then(excel => {
        const tHeader = ['交易ID', '币种', '数量', '确认数', '地址', '状态', '创建时间', '完成时间']
        const filterVal = ['txId', 'currency', 'amount', 'confirmationsText', 'address', 'statusText', 'createTime', 'completeTimeText']
        const data = this.formatJson(filterVal)
        excel.export_json_to_excel({
          header: tHeader,
          data,
          filename: '存币记录'
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
        } else if (j === 'confirmationsText') {
          return `${v.confirmations}/${v.requiredConfirmations}`
        } else {
          return v[j] || '-'
        }
      }))
    },
    generateMockData() {
      const mockData = []
      const statusTextMap = {
        'pending': '待处理',
        'processing': '处理中',
        'completed': '已完成',
        'rejected': '已拒绝',
        'failed': '失败'
      }
      
      // 生成20条模拟数据
      for (let i = 1; i <= 20; i++) {
        const currency = this.currencyOptions[Math.floor(Math.random() * this.currencyOptions.length)].key
        const amount = (Math.random() * 10).toFixed(currency === 'BTC' ? 8 : 4)
        const status = this.statusOptions[Math.floor(Math.random() * this.statusOptions.length)].key
        const confirmations = status === 'completed' ? 6 : (status === 'failed' ? 0 : Math.floor(Math.random() * 5))
        const requiredConfirmations = 6
        
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
        const completeTime = status === 'completed' ? new Date(createTime.getTime() + Math.floor(Math.random() * 24) * 60 * 60 * 1000) : null
        
        mockData.push({
          id: i,
          txId: this.generateRandomString(64),
          currency: currency,
          amount: amount,
          confirmations: confirmations,
          requiredConfirmations: requiredConfirmations,
          address: address,
          status: status,
          statusText: statusTextMap[status],
          blockHeight: status === 'completed' ? 700000 + Math.floor(Math.random() * 10000) : null,
          blockHash: status === 'completed' ? this.generateRandomString(64) : null,
          fee: (Math.random() * 0.001).toFixed(8),
          remark: '',
          createTime: createTime,
          completeTime: completeTime
        })
      }
      
      return mockData
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
  max-width: 200px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
</style>
