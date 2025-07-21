<template>
  <div class="app-container">
    <el-card class="box-card">
      <div slot="header" class="clearfix">
        <span>自选交易订单</span>
      </div>
      <div class="filter-container">
        <el-input
          v-model="listQuery.orderNo"
          placeholder="订单号"
          style="width: 200px;"
          class="filter-item"
          clearable
          @keyup.enter.native="handleFilter"
        />
        <el-select v-model="listQuery.status" placeholder="状态" clearable style="width: 120px" class="filter-item">
          <el-option v-for="item in statusOptions" :key="item.key" :label="item.display_name" :value="item.key" />
        </el-select>
        <el-select v-model="listQuery.type" placeholder="交易类型" clearable style="width: 120px" class="filter-item">
          <el-option v-for="item in typeOptions" :key="item.key" :label="item.display_name" :value="item.key" />
        </el-select>
        <el-date-picker
          v-model="dateRange"
          type="daterange"
          range-separator="至"
          start-placeholder="开始日期"
          end-placeholder="结束日期"
          value-format="yyyy-MM-dd"
          class="filter-item date-range"
        />
        <el-button v-waves class="filter-item" type="primary" icon="el-icon-search" @click="handleFilter">
          搜索
        </el-button>
        <el-button v-waves class="filter-item" type="success" icon="el-icon-download" @click="handleDownload">
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
        <el-table-column label="订单号" align="center" width="180">
          <template slot-scope="{row}">
            <span>{{ row.orderNo }}</span>
          </template>
        </el-table-column>
        <el-table-column label="交易类型" width="100px" align="center">
          <template slot-scope="{row}">
            <el-tag :type="row.type === '买入' ? 'success' : 'danger'">{{ row.type }}</el-tag>
          </template>
        </el-table-column>
        <el-table-column label="币种" width="80px" align="center">
          <template slot-scope="{row}">
            <span>{{ row.currency }}</span>
          </template>
        </el-table-column>
        <el-table-column label="数量" width="100px" align="center">
          <template slot-scope="{row}">
            <span>{{ row.amount }}</span>
          </template>
        </el-table-column>
        <el-table-column label="单价" width="100px" align="center">
          <template slot-scope="{row}">
            <span>{{ row.price }}</span>
          </template>
        </el-table-column>
        <el-table-column label="总金额" width="120px" align="center">
          <template slot-scope="{row}">
            <span>{{ row.total }}</span>
          </template>
        </el-table-column>
        <el-table-column label="交易方" width="120px" align="center">
          <template slot-scope="{row}">
            <span>{{ row.counterparty }}</span>
          </template>
        </el-table-column>
        <el-table-column label="支付方式" width="100px" align="center">
          <template slot-scope="{row}">
            <span>{{ row.paymentMethod }}</span>
          </template>
        </el-table-column>
        <el-table-column label="状态" class-name="status-col" width="100">
          <template slot-scope="{row}">
            <el-tag :type="row.status | statusFilter">
              {{ row.status }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column label="创建时间" width="180px" align="center">
          <template slot-scope="{row}">
            <span>{{ row.createdTime | parseTime('{y}-{m}-{d} {h}:{i}') }}</span>
          </template>
        </el-table-column>
        <el-table-column label="操作" align="center" width="230" class-name="small-padding fixed-width">
          <template slot-scope="{row}">
            <el-button v-if="row.status==='待付款'" size="mini" type="success" @click="handlePay(row)">
              付款
            </el-button>
            <el-button v-if="row.status==='待确认'" size="mini" type="primary" @click="handleConfirm(row)">
              确认
            </el-button>
            <el-button v-if="row.status==='待付款'" size="mini" type="danger" @click="handleCancel(row)">
              取消
            </el-button>
            <el-button size="mini" @click="handleDetail(row)">
              详情
            </el-button>
          </template>
        </el-table-column>
      </el-table>

      <pagination v-show="total>0" :total="total" :page.sync="listQuery.page" :limit.sync="listQuery.limit" @pagination="getList" />
    </el-card>

    <!-- 订单详情对话框 -->
    <el-dialog title="订单详情" :visible.sync="dialogVisible" width="600px">
      <el-descriptions :column="1" border>
        <el-descriptions-item label="订单号">{{ currentOrder.orderNo }}</el-descriptions-item>
        <el-descriptions-item label="交易类型">
          <el-tag :type="currentOrder.type === '买入' ? 'success' : 'danger'">{{ currentOrder.type }}</el-tag>
        </el-descriptions-item>
        <el-descriptions-item label="币种">{{ currentOrder.currency }}</el-descriptions-item>
        <el-descriptions-item label="数量">{{ currentOrder.amount }}</el-descriptions-item>
        <el-descriptions-item label="单价">{{ currentOrder.price }}</el-descriptions-item>
        <el-descriptions-item label="总金额">{{ currentOrder.total }}</el-descriptions-item>
        <el-descriptions-item label="交易方">{{ currentOrder.counterparty }}</el-descriptions-item>
        <el-descriptions-item label="支付方式">{{ currentOrder.paymentMethod }}</el-descriptions-item>
        <el-descriptions-item label="状态">
          <el-tag :type="currentOrder.status | statusFilter">{{ currentOrder.status }}</el-tag>
        </el-descriptions-item>
        <el-descriptions-item label="创建时间">{{ currentOrder.createdTime | parseTime('{y}-{m}-{d} {h}:{i}:{s}') }}</el-descriptions-item>
        <el-descriptions-item label="付款时间" v-if="currentOrder.payTime">{{ currentOrder.payTime | parseTime('{y}-{m}-{d} {h}:{i}:{s}') }}</el-descriptions-item>
        <el-descriptions-item label="完成时间" v-if="currentOrder.completedTime">{{ currentOrder.completedTime | parseTime('{y}-{m}-{d} {h}:{i}:{s}') }}</el-descriptions-item>
        <el-descriptions-item label="备注" v-if="currentOrder.remark">{{ currentOrder.remark }}</el-descriptions-item>
      </el-descriptions>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">关闭</el-button>
      </div>
    </el-dialog>

    <!-- 付款对话框 -->
    <el-dialog title="付款确认" :visible.sync="payDialogVisible" width="500px">
      <el-form :model="payForm" :rules="payRules" ref="payForm" label-width="100px">
        <el-form-item label="收款方式">
          <span>{{ currentOrder.paymentMethod }}</span>
        </el-form-item>
        <el-form-item label="收款账号">
          <span>{{ paymentAccount }}</span>
          <el-button type="text" icon="el-icon-document-copy" @click="copyPaymentAccount">复制</el-button>
        </el-form-item>
        <el-form-item label="收款人">
          <span>{{ paymentName }}</span>
        </el-form-item>
        <el-form-item label="付款金额">
          <span>{{ currentOrder.total }}</span>
        </el-form-item>
        <el-form-item label="付款凭证" prop="paymentProof">
          <el-upload
            class="upload-demo"
            action="https://jsonplaceholder.typicode.com/posts/"
            :on-preview="handlePreview"
            :on-remove="handleRemove"
            :before-upload="beforeUpload"
            :on-success="handleUploadSuccess"
            :limit="1"
            list-type="picture"
          >
            <el-button size="small" type="primary">点击上传</el-button>
            <div slot="tip" class="el-upload__tip">只能上传jpg/png文件，且不超过2MB</div>
          </el-upload>
        </el-form-item>
        <el-form-item label="备注" prop="remark">
          <el-input type="textarea" v-model="payForm.remark" :rows="3" placeholder="请输入付款备注信息"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="payDialogVisible = false">取消</el-button>
        <el-button type="primary" @click="submitPayment">确认付款</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import waves from '@/directive/waves'
import Pagination from '@/components/Pagination'
import clipboard from '@/directive/clipboard/index.js'
import { parseTime } from '@/utils'

export default {
  name: 'TradeOrder',
  components: { Pagination },
  directives: { waves, clipboard },
  filters: {
    statusFilter(status) {
      const statusMap = {
        '待付款': 'warning',
        '待确认': 'info',
        '已完成': 'success',
        '已取消': 'danger',
        '申诉中': 'danger'
      }
      return statusMap[status]
    }
  },
  data() {
    return {
      tableKey: 0,
      list: [
        { orderNo: 'TO202506240001', type: '买入', currency: 'BTC', amount: '0.5', price: '30000', total: '15000', counterparty: 'trader001', paymentMethod: '银行转账', status: '待付款', createdTime: Date.now() - 3600 * 1000 },
        { orderNo: 'TO202506240002', type: '卖出', currency: 'ETH', amount: '5', price: '2000', total: '10000', counterparty: 'trader002', paymentMethod: '支付宝', status: '待确认', createdTime: Date.now() - 7200 * 1000, payTime: Date.now() - 3600 * 1000 },
        { orderNo: 'TO202506240003', type: '买入', currency: 'BTC', amount: '0.2', price: '30000', total: '6000', counterparty: 'trader003', paymentMethod: '微信支付', status: '已完成', createdTime: Date.now() - 86400 * 1000, payTime: Date.now() - 82800 * 1000, completedTime: Date.now() - 79200 * 1000 },
        { orderNo: 'TO202506230001', type: '卖出', currency: 'BTC', amount: '0.3', price: '29000', total: '8700', counterparty: 'trader004', paymentMethod: '银行转账', status: '已取消', createdTime: Date.now() - 172800 * 1000, remark: '买家超时未付款' }
      ],
      total: 4,
      listLoading: false,
      listQuery: {
        page: 1,
        limit: 20,
        orderNo: '',
        status: '',
        type: ''
      },
      dateRange: '',
      statusOptions: [
        { key: '待付款', display_name: '待付款' },
        { key: '待确认', display_name: '待确认' },
        { key: '已完成', display_name: '已完成' },
        { key: '已取消', display_name: '已取消' },
        { key: '申诉中', display_name: '申诉中' }
      ],
      typeOptions: [
        { key: '买入', display_name: '买入' },
        { key: '卖出', display_name: '卖出' }
      ],
      dialogVisible: false,
      currentOrder: {},
      payDialogVisible: false,
      payForm: {
        paymentProof: '',
        remark: ''
      },
      payRules: {
        paymentProof: [
          { required: true, message: '请上传付款凭证', trigger: 'change' }
        ]
      },
      paymentAccount: '6222021234567890123',
      paymentName: '张三'
    }
  },
  created() {
    this.getList()
  },
  methods: {
    getList() {
      // 实际项目中，这里应该调用API获取数据
      this.listLoading = true
      setTimeout(() => {
        this.listLoading = false
      }, 500)
    },
    handleFilter() {
      this.listQuery.page = 1
      this.getList()
    },
    handleDetail(row) {
      this.currentOrder = Object.assign({}, row)
      this.dialogVisible = true
    },
    handlePay(row) {
      this.currentOrder = Object.assign({}, row)
      this.payForm = {
        paymentProof: '',
        remark: ''
      }
      this.payDialogVisible = true
    },
    handleConfirm(row) {
      this.$confirm('确认已收到付款?', '确认提示', {
        confirmButtonText: '确认',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        // 实际项目中，这里应该调用API确认订单
        const index = this.list.findIndex(item => item.orderNo === row.orderNo)
        if (index !== -1) {
          this.list[index].status = '已完成'
          this.list[index].completedTime = Date.now()
        }
        this.$message({
          type: 'success',
          message: '订单已确认完成'
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消确认'
        })
      })
    },
    handleCancel(row) {
      this.$confirm('确认取消此订单?', '取消提示', {
        confirmButtonText: '确认',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        // 实际项目中，这里应该调用API取消订单
        const index = this.list.findIndex(item => item.orderNo === row.orderNo)
        if (index !== -1) {
          this.list[index].status = '已取消'
        }
        this.$message({
          type: 'success',
          message: '订单已取消'
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消操作'
        })
      })
    },
    copyPaymentAccount() {
      this.$copyText(this.paymentAccount).then(() => {
        this.$message({
          message: '收款账号已复制',
          type: 'success'
        })
      }).catch(() => {
        this.$message({
          message: '复制失败',
          type: 'error'
        })
      })
    },
    handlePreview(file) {
      console.log(file)
    },
    handleRemove(file, fileList) {
      this.payForm.paymentProof = ''
    },
    beforeUpload(file) {
      const isJPGOrPNG = file.type === 'image/jpeg' || file.type === 'image/png'
      const isLt2M = file.size / 1024 / 1024 < 2

      if (!isJPGOrPNG) {
        this.$message.error('上传图片只能是 JPG 或 PNG 格式!')
      }
      if (!isLt2M) {
        this.$message.error('上传图片大小不能超过 2MB!')
      }
      return isJPGOrPNG && isLt2M
    },
    handleUploadSuccess(response, file, fileList) {
      this.payForm.paymentProof = file.url || URL.createObjectURL(file.raw)
    },
    submitPayment() {
      this.$refs.payForm.validate(valid => {
        if (valid) {
          // 实际项目中，这里应该调用API提交付款信息
          const index = this.list.findIndex(item => item.orderNo === this.currentOrder.orderNo)
          if (index !== -1) {
            this.list[index].status = '待确认'
            this.list[index].payTime = Date.now()
          }
          this.payDialogVisible = false
          this.$message({
            type: 'success',
            message: '付款信息已提交，请等待卖家确认'
          })
        } else {
          return false
        }
      })
    },
    handleDownload() {
      this.listLoading = true
      import('@/vendor/Export2Excel').then(excel => {
        const tHeader = ['订单号', '交易类型', '币种', '数量', '单价', '总金额', '交易方', '支付方式', '状态', '创建时间']
        const filterVal = ['orderNo', 'type', 'currency', 'amount', 'price', 'total', 'counterparty', 'paymentMethod', 'status', 'createdTime']
        const data = this.formatJson(filterVal)
        excel.export_json_to_excel({
          header: tHeader,
          data,
          filename: '自选交易订单'
        })
        this.listLoading = false
      })
    },
    formatJson(filterVal) {
      return this.list.map(v => filterVal.map(j => {
        if (j === 'createdTime') {
          return parseTime(v[j])
        } else {
          return v[j]
        }
      }))
    }
  }
}
</script>

<style scoped>
.filter-container {
  padding-bottom: 10px;
}
.filter-item {
  margin-right: 10px;
}
.date-range {
  width: 320px;
}
</style>
