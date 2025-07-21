<template>
  <div class="app-container">
    <div class="filter-container">
      <el-input v-model="listQuery.recordId" placeholder="流水号" style="width: 200px;" class="filter-item" @keyup.enter.native="handleFilter" />
      <el-select v-model="listQuery.type" placeholder="类型" clearable style="width: 120px" class="filter-item">
        <el-option v-for="item in typeOptions" :key="item.key" :label="item.display_name" :value="item.key" />
      </el-select>
      <el-select v-model="listQuery.currency" placeholder="币种" clearable style="width: 120px" class="filter-item">
        <el-option v-for="item in currencyOptions" :key="item.key" :label="item.display_name" :value="item.key" />
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
      <el-table-column label="流水号" align="center" width="180">
        <template slot-scope="{row}">
          <span>{{ row.recordId }}</span>
        </template>
      </el-table-column>
      <el-table-column label="类型" width="120" align="center">
        <template slot-scope="{row}">
          <el-tag :type="row.type | typeFilter">
            {{ row.typeText }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column label="币种" width="120" align="center">
        <template slot-scope="{row}">
          <span>{{ row.currency }}</span>
        </template>
      </el-table-column>
      <el-table-column label="数量" width="120" align="center">
        <template slot-scope="{row}">
          <span>{{ row.amount }}</span>
        </template>
      </el-table-column>
      <el-table-column label="余额" width="120" align="center">
        <template slot-scope="{row}">
          <span>{{ row.balance }}</span>
        </template>
      </el-table-column>
      <el-table-column label="关联订单" width="180" align="center">
        <template slot-scope="{row}">
          <span v-if="row.orderId">{{ row.orderId }}</span>
          <span v-else>-</span>
        </template>
      </el-table-column>
      <el-table-column label="备注" min-width="150" align="center">
        <template slot-scope="{row}">
          <span>{{ row.remark || '-' }}</span>
        </template>
      </el-table-column>
      <el-table-column label="创建时间" width="180" align="center">
        <template slot-scope="{row}">
          <span>{{ row.createTime | parseTime }}</span>
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

    <el-dialog title="流水详情" :visible.sync="detailDialogVisible" width="50%">
      <el-descriptions :column="2" border>
        <el-descriptions-item label="流水号">{{ currentRecord.recordId }}</el-descriptions-item>
        <el-descriptions-item label="类型">
          <el-tag :type="currentRecord.type | typeFilter">{{ currentRecord.typeText }}</el-tag>
        </el-descriptions-item>
        <el-descriptions-item label="币种">{{ currentRecord.currency }}</el-descriptions-item>
        <el-descriptions-item label="数量">{{ currentRecord.amount }}</el-descriptions-item>
        <el-descriptions-item label="余额">{{ currentRecord.balance }}</el-descriptions-item>
        <el-descriptions-item label="关联订单">{{ currentRecord.orderId || '-' }}</el-descriptions-item>
        <el-descriptions-item label="备注" :span="2">{{ currentRecord.remark || '-' }}</el-descriptions-item>
        <el-descriptions-item label="创建时间" :span="2">{{ currentRecord.createTime | parseTime }}</el-descriptions-item>
      </el-descriptions>
      <span slot="footer" class="dialog-footer">
        <el-button @click="detailDialogVisible = false">关闭</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
import waves from '@/directive/waves' // waves directive
import { parseTime } from '@/utils'
import Pagination from '@/components/Pagination' // secondary package based on el-pagination

export default {
  name: 'AssetRecord',
  components: { Pagination },
  directives: { waves },
  filters: {
    typeFilter(type) {
      const typeMap = {
        'deposit': 'success',
        'withdraw': 'warning',
        'buy': 'primary',
        'sell': 'info',
        'fee': 'danger'
      }
      return typeMap[type]
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
        recordId: undefined,
        type: undefined,
        currency: undefined,
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
      typeOptions: [
        { key: 'deposit', display_name: '充值' },
        { key: 'withdraw', display_name: '提现' },
        { key: 'buy', display_name: '买入' },
        { key: 'sell', display_name: '卖出' },
        { key: 'fee', display_name: '手续费' }
      ],
      currencyOptions: [
        { key: 'BTC', display_name: 'BTC' },
        { key: 'ETH', display_name: 'ETH' },
        { key: 'USDT', display_name: 'USDT' },
        { key: 'LTC', display_name: 'LTC' },
        { key: 'EOS', display_name: 'EOS' }
      ],
      detailDialogVisible: false,
      currentRecord: {},
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
      this.currentRecord = Object.assign({}, row)
      this.detailDialogVisible = true
    },
    handleDownload() {
      this.downloadLoading = true
      import('@/vendor/Export2Excel').then(excel => {
        const tHeader = ['流水号', '类型', '币种', '数量', '余额', '关联订单', '备注', '创建时间']
        const filterVal = ['recordId', 'typeText', 'currency', 'amount', 'balance', 'orderId', 'remark', 'createTime']
        const data = this.formatJson(filterVal)
        excel.export_json_to_excel({
          header: tHeader,
          data,
          filename: '资产流水记录'
        })
        this.downloadLoading = false
      })
    },
    formatJson(filterVal) {
      return this.list.map(v => filterVal.map(j => {
        if (j === 'createTime') {
          return parseTime(v[j])
        } else {
          return v[j] || '-'
        }
      }))
    },
    generateMockData() {
      const mockData = []
      const typeTextMap = {
        'deposit': '充值',
        'withdraw': '提现',
        'buy': '买入',
        'sell': '卖出',
        'fee': '手续费'
      }
      
      // 生成20条模拟数据
      for (let i = 1; i <= 20; i++) {
        const type = this.typeOptions[Math.floor(Math.random() * this.typeOptions.length)].key
        const currency = this.currencyOptions[Math.floor(Math.random() * this.currencyOptions.length)].key
        const amount = (Math.random() * 10).toFixed(4)
        const balance = (Math.random() * 100).toFixed(4)
        
        mockData.push({
          id: i,
          recordId: `AR${String(Date.now()).slice(-8)}${i}`,
          type: type,
          typeText: typeTextMap[type],
          currency: currency,
          amount: type === 'withdraw' || type === 'sell' || type === 'fee' ? `-${amount}` : amount,
          balance: balance,
          orderId: type === 'buy' || type === 'sell' ? `OR${String(Date.now()).slice(-8)}${i}` : null,
          remark: type === 'fee' ? '交易手续费' : (type === 'deposit' ? '充值' : (type === 'withdraw' ? '提现' : '交易')),
          createTime: new Date(Date.now() - Math.floor(Math.random() * 30) * 24 * 60 * 60 * 1000)
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
</style>
