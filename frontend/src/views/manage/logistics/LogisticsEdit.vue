<template>
  <a-modal v-model="show" title="修改物流" @cancel="onClose" :width="950">
    <template slot="footer">
      <a-button key="back" @click="onClose">
        取消
      </a-button>
      <a-button key="submit" type="primary" :loading="loading" @click="handleSubmit">
        修改
      </a-button>
    </template>
    <div style="font-size: 13px;font-family: SimHei">

      <a-row style="padding-left: 24px;padding-right: 24px;" v-if="logistics != null && logistics.status == 4">
        <a-col style="margin-bottom: 15px">
          <span style="font-size: 15px;font-weight: 650;color: #000c17">运输温度</span>
        </a-col>
        <a-col :span="24">
          <!-- 当前温度展示 -->
          <div style="display: flex; align-items: center;">
            <span style="font-size: 18px; font-weight: bold;">当前温度：</span>
            <span :style="{ color: temperatureWarning ? '#ff4d4f' : '#52c41a', fontSize: '20px', fontWeight: 'bold' }">
        {{ currentTemperature }}°C
        </span>
              <!-- 温度过高提示 -->
              <span v-if="temperatureWarning" style="margin-left: 10px; color: #ff4d4f; font-weight: bold;">
          🔥 温度过高！
        </span>
          </div>
        </a-col>
      </a-row>
      <a-row style="padding-left: 24px;padding-right: 24px;">
        <a-col style="margin-bottom: 15px"><span style="font-size: 15px;font-weight: 650;color: #000c17">当前物流</span></a-col>
         <a-col :span="24">
          <a-table :columns="logisticsColumns" :data-source="logisticsList" :pagination="false">
          </a-table>
        </a-col>
      </a-row>
      <a-divider orientation="left">
        <span style="font-size: 12px;font-family: SimHei">更新物流</span>
      </a-divider>
      <a-row style="padding-left: 24px;padding-right: 24px;" :gutter="50">
        <a-col :span="24">
          <a-form-item label='物流备注' v-bind="formItemLayout">
            <a-textarea :rows="6" v-model="remark"/>
          </a-form-item>
        </a-col>
      </a-row>
    </div>
  </a-modal>
</template>

<script>
import {mapState} from 'vuex'
function getBase64 (file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.readAsDataURL(file)
    reader.onload = () => resolve(reader.result)
    reader.onerror = error => reject(error)
  })
}
const formItemLayout = {
  labelCol: { span: 24 },
  wrapperCol: { span: 24 }
}
export default {
  name: 'logisticsEdit',
  props: {
    logisticsEditVisiable: {
      default: false
    }
  },
  computed: {
    ...mapState({
      currentUser: state => state.account.user
    }),
    show: {
      get: function () {
        return this.logisticsEditVisiable
      },
      set: function () {
      }
    },
    logisticsColumns () {
      return [{
        title: '物流信息',
        dataIndex: 'remark'
      }, {
        title: '操作时间',
        dataIndex: 'createDate'
      }]
    }
  },
  data () {
    return {
      rowId: null,
      formItemLayout,
      form: this.$form.createForm(this),
      loading: false,
      fileList: [],
      logistics: null,
      previewVisible: false,
      previewImage: '',
      logisticsList: [],
      remark: '',
      currentTemperature: 0, // 当前温度
      temperatureWarning: false, // 是否温度过高
      temperatureTimer: null
    }
  },
  mounted() {
    // 组件挂载后启动定时器
    this.startTemperatureUpdate();
  },
  beforeDestroy() {
    // 组件销毁前清除定时器
    if (this.temperatureTimer) {
      clearInterval(this.temperatureTimer);
    }
  },
  methods: {
    // 启动温度更新定时器
    startTemperatureUpdate() {
      this.temperatureTimer = setInterval(() => {
        // 生成 0-10 的随机温度
        this.currentTemperature = Math.floor(Math.random() * 11);

        // 判断是否温度过高
        this.temperatureWarning = this.currentTemperature > 7;
      }, 5000); // 每 5 秒更新一次
    },
    selectLogistics (orderId) {
      this.$get(`/cos/logistics-info/order/${orderId}`).then((r) => {
        this.logisticsList = r.data.data
      })
    },
    handleCancel () {
      this.previewVisible = false
    },
    async handlePreview (file) {
      if (!file.url && !file.preview) {
        file.preview = await getBase64(file.originFileObj)
      }
      this.previewImage = file.url || file.preview
      this.previewVisible = true
    },
    picHandleChange ({ fileList }) {
      this.fileList = fileList
    },
    imagesInit (images) {
      if (images !== null && images !== '') {
        let imageList = []
        images.split(',').forEach((image, index) => {
          imageList.push({uid: index, name: image, status: 'done', url: 'http://127.0.0.1:9527/imagesWeb/' + image})
        })
        this.fileList = imageList
      }
    },
    setFormValues ({...logistics}) {
      this.logistics = logistics
      this.rowId = logistics.orderId
      this.selectLogistics(logistics.orderId)
    },
    reset () {
      this.loading = false
      this.form.resetFields()
    },
    onClose () {
      this.remark = ''
      this.reset()
      this.$emit('close')
    },
    handleSubmit () {
      this.form.validateFields((err, values) => {
        values.id = this.rowId
        if (!err) {
          this.loading = true
          this.$put('/cos/logistics-info/updateLogisticsOrder', {
            'orderId': this.rowId,
            'remark': this.remark
          }).then((r) => {
            this.reset()
            this.remark = ''
            this.$emit('success')
          }).catch(() => {
            this.loading = false
          })
        }
      })
    }
  }
}
</script>

<style scoped>

</style>
