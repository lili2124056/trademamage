<template>
  <div class="app-container">
    <el-card class="box-card">
      <div slot="header" class="clearfix">
        <span>安全设置</span>
      </div>
      <div class="security-content">
        <el-form ref="form" :model="form" label-width="120px">
          <el-form-item label="登录密码">
            <div class="security-item">
              <span>用于保护账户安全</span>
              <el-button type="text" @click="showPasswordDialog">修改</el-button>
            </div>
          </el-form-item>
          <el-form-item label="手机验证">
            <div class="security-item">
              <span>{{ form.phone ? '已绑定: ' + maskPhone(form.phone) : '未绑定' }}</span>
              <el-button type="text" @click="showPhoneDialog">{{ form.phone ? '修改' : '绑定' }}</el-button>
            </div>
          </el-form-item>
          <el-form-item label="邮箱验证">
            <div class="security-item">
              <span>{{ form.email ? '已绑定: ' + maskEmail(form.email) : '未绑定' }}</span>
              <el-button type="text" @click="showEmailDialog">{{ form.email ? '修改' : '绑定' }}</el-button>
            </div>
          </el-form-item>
          <el-form-item label="谷歌验证">
            <div class="security-item">
              <span>{{ form.googleAuth ? '已开启' : '未开启' }}</span>
              <el-button type="text" @click="showGoogleAuthDialog">{{ form.googleAuth ? '关闭' : '开启' }}</el-button>
            </div>
          </el-form-item>
        </el-form>
      </div>
    </el-card>

    <!-- 修改密码对话框 -->
    <el-dialog title="修改密码" :visible.sync="passwordDialogVisible" width="500px">
      <el-form ref="passwordForm" :model="passwordForm" :rules="passwordRules" label-width="100px">
        <el-form-item label="原密码" prop="oldPassword">
          <el-input v-model="passwordForm.oldPassword" type="password" show-password></el-input>
        </el-form-item>
        <el-form-item label="新密码" prop="newPassword">
          <el-input v-model="passwordForm.newPassword" type="password" show-password></el-input>
        </el-form-item>
        <el-form-item label="确认新密码" prop="confirmPassword">
          <el-input v-model="passwordForm.confirmPassword" type="password" show-password></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="passwordDialogVisible = false">取消</el-button>
        <el-button type="primary" @click="updatePassword">确认</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  name: 'AccountSecurity',
  data() {
    // 验证确认密码是否一致
    const validateConfirmPassword = (rule, value, callback) => {
      if (value !== this.passwordForm.newPassword) {
        callback(new Error('两次输入的密码不一致'))
      } else {
        callback()
      }
    }
    return {
      form: {
        phone: '',
        email: '',
        googleAuth: false
      },
      passwordDialogVisible: false,
      passwordForm: {
        oldPassword: '',
        newPassword: '',
        confirmPassword: ''
      },
      passwordRules: {
        oldPassword: [
          { required: true, message: '请输入原密码', trigger: 'blur' }
        ],
        newPassword: [
          { required: true, message: '请输入新密码', trigger: 'blur' },
          { min: 6, message: '密码长度不能少于6个字符', trigger: 'blur' }
        ],
        confirmPassword: [
          { required: true, message: '请再次输入新密码', trigger: 'blur' },
          { validator: validateConfirmPassword, trigger: 'blur' }
        ]
      }
    }
  },
  created() {
    this.fetchUserSecurityInfo()
  },
  methods: {
    fetchUserSecurityInfo() {
      // 这里应该从API获取用户的安全信息
      // 模拟数据
      this.form = {
        phone: '13800138000',
        email: 'user@example.com',
        googleAuth: true
      }
    },
    maskPhone(phone) {
      if (!phone) return ''
      return phone.replace(/(\d{3})\d{4}(\d{4})/, '$1****$2')
    },
    maskEmail(email) {
      if (!email) return ''
      const parts = email.split('@')
      if (parts.length !== 2) return email
      const name = parts[0]
      const domain = parts[1]
      const maskedName = name.length > 2 
        ? name.substring(0, 1) + '*'.repeat(name.length - 2) + name.substring(name.length - 1)
        : name
      return maskedName + '@' + domain
    },
    showPasswordDialog() {
      this.passwordDialogVisible = true
      this.passwordForm = {
        oldPassword: '',
        newPassword: '',
        confirmPassword: ''
      }
    },
    showPhoneDialog() {
      this.$message({
        message: '手机验证功能正在开发中',
        type: 'info'
      })
    },
    showEmailDialog() {
      this.$message({
        message: '邮箱验证功能正在开发中',
        type: 'info'
      })
    },
    showGoogleAuthDialog() {
      this.$message({
        message: 'Google验证功能正在开发中',
        type: 'info'
      })
    },
    updatePassword() {
      this.$refs.passwordForm.validate(valid => {
        if (valid) {
          // 这里应该调用API更新密码
          this.$message({
            message: '密码修改成功',
            type: 'success'
          })
          this.passwordDialogVisible = false
        } else {
          return false
        }
      })
    }
  }
}
</script>

<style scoped>
.security-content {
  max-width: 600px;
}
.security-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
</style>
