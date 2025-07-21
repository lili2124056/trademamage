<template>
  <div class="app-container">
    <el-card class="box-card">
      <div slot="header" class="clearfix">
        <span>快捷客户买单</span>
      </div>
      
      <!-- 筛选区域 -->
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
        <el-button v-waves class="filter-item" type="success" icon="el-icon-plus" @click="handleCreate">
          新增买单
        </el-button>
      </div>
      
      <!-- 表格区域 -->
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
            <el-button v-if="row.status==='待处理'" size="mini" type="primary" @click="handleAccept(row)">
              接单
            </el-button>
            <el-button v-if="row.status==='待处理'" size="mini" type="danger" @click="handleCancel(row)">
              取消
            </el-button>
            <el-button v-if="row.status==='已接单'" size="mini" type="success" @click="handleComplete(row)">
              完成
            </el-button>
            <el-button size="mini" @click="handleUpdate(row)">
              编辑
            </el-button>
            <el-button size="mini" type="info" @click="handleDetail(row)">
              详情
            </el-button>
          </template>
        </el-table-column>
      </el-table>
      
      <pagination v-show="total>0" :total="total" :page.sync="listQuery.page" :limit.sync="listQuery.limit" @pagination="getList" />
    </el-card>
    
    <!-- 新增/编辑对话框 -->
    <el-dialog :title="textMap[dialogStatus]" :visible.sync="dialogVisible" width="500px">
      <el-form ref="dataForm" :rules="rules" :model="currentOrder" label-position="left" label-width="100px">
        <el-form-item label="币种" prop="currency">
          <el-select v-model="currentOrder.currency" placeholder="请选择币种" style="width: 100%">
            <el-option label="BTC" value="BTC" />
            <el-option label="ETH" value="ETH" />
            <el-option label="USDT" value="USDT" />
          </el-select>
        </el-form-item>
        <el-form-item label="数量" prop="amount">
          <el-input v-model="currentOrder.amount" type="number" placeholder="请输入数量" @input="calculateTotal" />
        </el-form-item>
        <el-form-item label="单价" prop="price">
          <el-input v-model="currentOrder.price" type="number" placeholder="请输入单价" @input="calculateTotal" />
        </el-form-item>
        <el-form-item label="总金额">
          <el-input v-model="currentOrder.total" disabled />
        </el-form-item>
        <el-form-item label="支付方式" prop="paymentMethod">
          <el-select v-model="currentOrder.paymentMethod" placeholder="请选择支付方式" style="width: 100%">
            <el-option label="银行转账" value="银行转账" />
            <el-option label="支付宝" value="支付宝" />
            <el-option label="微信支付" value="微信支付" />
          </el-select>
        </el-form-item>
        <el-form-item label="备注">
          <el-input v-model="currentOrder.remark" type="textarea" :rows="3" placeholder="请输入备注信息" />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取消</el-button>
        <el-button type="primary" @click="dialogStatus === 'create' ? createData() : updateData()">确认</el-button>
      </div>
    </el-dialog>
    
    <!-- 详情对话框 -->
    <el-dialog title="买单详情" :visible.sync="detailDialogVisible" width="600px">
      <el-descriptions :column="1" border>
        <el-descriptions-item label="订单号">{{ currentOrder.orderNo }}</el-descriptions-item>
        <el-descriptions-item label="币种">{{ currentOrder.currency }}</el-descriptions-item>
        <el-descriptions-item label="数量">{{ currentOrder.amount }}</el-descriptions-item>
        <el-descriptions-item label="单价">{{ currentOrder.price }}</el-descriptions-item>
        <el-descriptions-item label="总金额">{{ currentOrder.total }}</el-descriptions-item>
        <el-descriptions-item label="支付方式">{{ currentOrder.paymentMethod }}</el-descriptions-item>
        <el-descriptions-item label="状态">
          <el-tag :type="currentOrder.status | statusFilter">{{ currentOrder.status }}</el-tag>
        </el-descriptions-item>
        <el-descriptions-item label="创建时间">{{ currentOrder.createdTime | parseTime('{y}-{m}-{d} {h}:{i}:{s}') }}</el-descriptions-item>
        <el-descriptions-item label="接单时间" v-if="currentOrder.acceptTime">{{ currentOrder.acceptTime | parseTime('{y}-{m}-{d} {h}:{i}:{s}') }}</el-descriptions-item>
        <el-descriptions-item label="完成时间" v-if="currentOrder.completedTime">{{ currentOrder.completedTime | parseTime('{y}-{m}-{d} {h}:{i}:{s}') }}</el-descriptions-item>
        <el-descriptions-item label="备注" v-if="currentOrder.remark">{{ currentOrder.remark }}</el-descriptions-item>
      </el-descriptions>
      <div slot="footer" class="dialog-footer">
        <el-button @click="detailDialogVisible = false">关闭</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import waves from '@/directive/waves'
import Pagination from '@/components/Pagination'
import { parseTime } from '@/utils'

export default {
  name: 'QuickBuy',
  components: { Pagination },
  directives: { waves },
  filters: {
    statusFilter(status) {
      const statusMap = {
        '待处理': 'info',
        '已接单': 'warning',
        '已完成': 'success',
        '已取消': 'danger'
      }
      return statusMap[status]
    }
  },
  data() {
    return {
      tableKey: 0,
      list: [
        { orderNo: 'QB202506240001', currency: 'BTC', amount: '0.5', price: '30000', total: '15000', paymentMethod: '银行转账', status: '待处理', createdTime: Date.now() - 3600 * 1000 },
        { orderNo: 'QB202506240002', currency: 'ETH', amount: '10', price: '2000', total: '20000', paymentMethod: '支付宝', status: '已接单', createdTime: Date.now() - 7200 * 1000, acceptTime: Date.now() - 3600 * 1000 },
        { orderNo: 'QB202506240003', currency: 'USDT', amount: '5000', price: '1', total: '5000', paymentMethod: '微信支付', status: '已完成', createdTime: Date.now() - 86400 * 1000, acceptTime: Date.now() - 82800 * 1000, completedTime: Date.now() - 79200 * 1000 },
        { orderNo: 'QB202506230001', currency: 'BTC', amount: '0.2', price: '29000', total: '5800', paymentMethod: '银行转账', status: '已取消', createdTime: Date.now() - 172800 * 1000, remark: '客户取消订单' }
      ],
      total: 4,
      listLoading: false,
      listQuery: {
        page: 1,
        limit: 20,
        orderNo: '',
        status: ''
      },
      dateRange: '',
      statusOptions: [
        { key: '待处理', display_name: '待处理' },
        { key: '已接单', display_name: '已接单' },
        { key: '已完成', display_name: '已完成' },
        { key: '已取消', display_name: '已取消' }
      ],
      dialogVisible: false,
      detailDialogVisible: false,
      dialogStatus: '',
      textMap: {
        update: '编辑买单',
        create: '新增买单'
      },
      currentOrder: {},
      rules: {
        amount: [{ required: true, message: '请输入购买数量', trigger: 'blur' }],
        price: [{ required: true, message: '请输入购买价格', trigger: 'blur' }],
        currency: [{ required: true, message: '请选择币种', trigger: 'change' }],
        paymentMethod: [{ required: true, message: '请选择支付方式', trigger: 'change' }]
      }
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
    calculateTotal() {
      if (this.currentOrder.amount && this.currentOrder.price) {
        this.currentOrder.total = (parseFloat(this.currentOrder.amount) * parseFloat(this.currentOrder.price)).toFixed(2)
      }
    },
    resetCurrentOrder() {
      this.currentOrder = {
        currency: '',
        amount: '',
        price: '',
        total: '',
        paymentMethod: '',
        remark: '',
        status: '待处理'
      }
    },
    handleCreate() {
      this.resetCurrentOrder()
      this.dialogStatus = 'create'
      this.dialogVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    createData() {
      this.$refs['dataForm'].validate((valid) => {
        if (valid) {
          // 计算总金额
          this.currentOrder.total = (parseFloat(this.currentOrder.amount) * parseFloat(this.currentOrder.price)).toFixed(2)
          
          // 实际项目中，这里应该调用API创建订单
          this.currentOrder.orderNo = 'QB' + parseTime(new Date(), '{y}{m}{d}') + Math.floor(Math.random() * 10000).toString().padStart(4, '0')
          this.currentOrder.createdTime = Date.now()
          
          this.list.unshift(this.currentOrder)
          this.total = this.list.length
          
          this.dialogVisible = false
          this.$notify({
            title: '成功',
            message: '创建买单成功',
            type: 'success',
            duration: 2000
          })
        }
      })
    },
    handleUpdate(row) {
      this.currentOrder = Object.assign({}, row) // 复制对象，避免直接修改
      this.dialogStatus = 'update'
      this.dialogVisible = true
      this.$nextTick(() => {
        this.$refs['dataForm'].clearValidate()
      })
    },
    updateData() {
      this.$refs['dataForm'].validate((valid) => {
        if (valid) {
          // 计算总金额
          this.currentOrder.total = (parseFloat(this.currentOrder.amount) * parseFloat(this.currentOrder.price)).toFixed(2)
          
          // 实际项目中，这里应该调用API更新订单
          const index = this.list.findIndex(v => v.orderNo === this.currentOrder.orderNo)
          this.list.splice(index, 1, this.currentOrder)
          
          this.dialogVisible = false
          this.$notify({
            title: '成功',
            message: '更新买单成功',
            type: 'success',
            duration: 2000
          })
        }
      })
    },
    handleDetail(row) {
      this.currentOrder = Object.assign({}, row)
      this.detailDialogVisible = true
    },
    handleAccept(row) {
      this.$confirm('确认接受此买单?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        // 实际项目中，这里应该调用API接单
        const index = this.list.findIndex(v => v.orderNo === row.orderNo)
        if (index !== -1) {
          this.list[index].status = '已接单'
          this.list[index].acceptTime = Date.now()
        }
        this.$message({
          type: 'success',
          message: '接单成功!'
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消接单'
        })
      })
    },
    handleComplete(row) {
      this.$confirm('确认完成此买单?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        // 实际项目中，这里应该调用API完成订单
        const index = this.list.findIndex(v => v.orderNo === row.orderNo)
        if (index !== -1) {
          this.list[index].status = '已完成'
          this.list[index].completedTime = Date.now()
        }
        this.$message({
          type: 'success',
          message: '订单已完成!'
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消操作'
        })
      })
    },
    handleCancel(row) {
      this.$confirm('确认取消此买单?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        // 实际项目中，这里应该调用API取消订单
        const index = this.list.findIndex(v => v.orderNo === row.orderNo)
        if (index !== -1) {
          this.list[index].status = '已取消'
        }
        this.$message({
          type: 'success',
          message: '订单已取消!'
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消操作'
        })
      })
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
  margin-bottom: 10px;
}
.date-range {
  width: 320px;
}
.el-tag {
  margin-right: 5px;
}
.small-padding {
  padding-left: 8px;
  padding-right: 8px;
}
.fixed-width {
  min-width: 200px;
}
</style>
