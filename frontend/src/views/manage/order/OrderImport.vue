
<template>
  <a-drawer
    title="导入订单"
    :width="720"
    :visible="visible"
    :body-style="{ paddingBottom: '80px' }"
    @close="onClose"
  >
    <a-form :form="form" layout="vertical">
      <!-- 订单类型选择 -->
      <a-form-item label="订单类型">
        <a-select
          v-decorator="['type', { rules: [{ required: true, message: '请选择订单类型' }] }]"
          placeholder="请选择订单类型"
        >
          <a-select-option value="0">正常订单</a-select-option>
          <a-select-option value="1">预付款</a-select-option>
        </a-select>
      </a-form-item>

      <!-- 模板下载区域 -->
      <a-form-item label="导入模板">
        <a-button type="primary" ghost @click="downloadTemplate">
          <a-icon type="download" /> 下载导入模板
        </a-button>
        <p class="template-tip">请先下载模板，按照模板格式填写后进行导入</p>
      </a-form-item>

      <!-- 文件上传区域 -->
      <a-form-item label="选择文件">
        <a-upload
          :file-list="fileList"
          :before-upload="beforeUpload"
          :remove="handleRemove"
          accept=".xlsx,.xls"
        >
          <a-button>
            <a-icon type="upload" /> 选择Excel文件
          </a-button>
        </a-upload>
        <p class="upload-tip">支持.xls和.xlsx格式文件</p>
      </a-form-item>

      <!-- 预览导入结果 -->
      <div v-if="previewData.length > 0" class="preview-section">
        <h3>预览导入数据</h3>
        <a-table
          :columns="previewColumns"
          :data-source="previewData"
          :pagination="false"
          size="small"
          bordered
        >
        </a-table>
      </div>

      <div>
        <!-- 订单类型选择 -->
        <a-form-item label="订单类型">
          <a-select
            v-decorator="['type', { rules: [{ required: true, message: '请选择订单类型' }] }]"
            placeholder="请选择订单类型"
          >
            <a-select-option value="0">正常订单</a-select-option>
            <a-select-option value="1">预付款</a-select-option>
          </a-select>
        </a-form-item>

        <!-- 用户选择 -->
        <a-form-item label="选择用户">
          <a-select
            v-decorator="['userId', { rules: [{ required: true, message: '请选择用户' }] }]"
            placeholder="请选择要采购的用户"
            show-search
            :filter-option="filterOption"
            @focus="loadUsers"
          >
            <a-select-option
              v-for="user in userList"
              :key="user.id"
              :value="user.id"
            >
              {{ user.name }} ({{ user.phone }})
            </a-select-option>
          </a-select>
        </a-form-item>
      </div>

      <!-- 导入按钮 -->
      <div class="drawer-footer">
        <a-button @click="onClose">取消</a-button>
        <a-button
          type="primary"
          :loading="importLoading"
          :disabled="fileList.length === 0"
          @click="handleImport"          style="margin-left: 8px;"
        >
          {{ importLoading ? '导入中...' : '确认导入' }}
        </a-button>
      </div>
    </a-form>
  </a-drawer>
</template>

<script>export default {
  name: 'OrderImport',
  props: {
    visible: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      form: this.$form.createForm(this),
      fileList: [],
      importLoading: false,
      previewData: [],
      userList: [],
      previewColumns: [
        {
          title: '商品名称',
          dataIndex: 'commodityName',
          key: 'commodityName'
        },
        {
          title: '采购价格',
          dataIndex: 'purchasePrice',
          key: 'purchasePrice'
        },
        {
          title: '单价',
          dataIndex: 'unitPrice',
          key: 'unitPrice'
        }
      ]
    }
  },
  mounted() {
    this.loadUsers()
  },
  methods: {
    onClose() {
      this.form.resetFields()
      this.fileList = []
      this.previewData = []
      this.userList = []
      this.$emit('close')
    },

    // 用户筛选方法
    filterOption(input, option) {
      return (
        option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
      )
    },

    // 加载用户列表
    loadUsers() {
      if (this.userList.length === 0) {
        this.$get('/cos/user-info/list').then(res => {
          if (res.data && res.data.data) {
            this.userList = res.data.data
          }
        }).catch(() => {
          this.$message.error('加载用户列表失败')
        })
      }
    },

    // 下载模板
    downloadTemplate() {
      // 创建临时链接触发下载
      const link = document.createElement('a')
      link.href = 'http://127.0.0.1:9527/imagesWeb/template.xlsx'
      link.download = '订单导入模板.xlsx'
      link.target = '_blank'
      document.body.appendChild(link)
      link.click()
      document.body.removeChild(link)
    },

    // 文件上传前处理
    beforeUpload(file) {
      this.fileList = [file]
      this.previewUploadedFile(file)
      return false
    },

    // 移除文件
    handleRemove() {
      this.fileList = []
      this.previewData = []
    },

    // 预览上传的文件内容
    previewUploadedFile(file) {
      const formData = new FormData()
      formData.append('file', file)

      this.$post('/cos/order-info/upload/preview', formData)
        .then(res => {
          if (res.data && res.data.data) {
            this.previewData = res.data.data
            this.$message.success(`成功解析 ${this.previewData.length} 条商品记录`)
          }
        })
        .catch(() => {
          this.$message.error('文件解析失败，请检查文件格式')
        })
    },

    // 执行导入
    handleImport() {
      this.form.validateFields((err, values) => {
        if (!err && this.fileList.length > 0) {
          this.importLoading = true
          const formData = new FormData()
          formData.append('file', this.fileList[0])
          formData.append('type', values.type)
          formData.append('userId', values.userId)

          this.$post('/cos/order/import', formData)
            .then(res => {
              this.importLoading = false
              if (res.data && res.data.code === 200) {
                this.$message.success('订单导入成功')
                this.onClose()
                this.$emit('success')
              } else {
                this.$message.error(res.data.message || '导入失败')
              }
            })
            .catch(() => {
              this.importLoading = false
              this.$message.error('导入失败')
            })
        }
      })
    }
  }
}
</script>

<style scoped>.template-tip {
  margin: 8px 0 0 0;
  color: #666;
  font-size: 12px;
}

.upload-tip {
  margin: 8px 0 0 0;
  color: #999;
  font-size: 12px;
}

.preview-section {
  margin-top: 20px;
  padding: 16px;
  background: #f5f5f5;
  border-radius: 4px;
}

.preview-section h3 {
  margin-bottom: 12px;
  color: #333;
}

.drawer-footer {
  position: absolute;
  right: 0;
  bottom: 0;
  width: 100%;
  border-top: 1px solid #e9e9e9;
  padding: 10px 16px;
  background: #fff;
  text-align: right;
  z-index: 1;
}
</style>
