<template>
  <div>
    <a-form>
      <a-form-item label="個人動態分享照片" v-bind="validateInfos.thumb">
        <div class="clearfix">
          <a-upload
            :action="`${baseURL}/admin/file`"
            list-type="picture-card"
            v-model:file-list="fileList"
            @preview="handlePreview"
          >
            <div v-if="fileList.length < 8">
              <plus-outlined />
              <div class="ant-upload-text">Upload</div>
            </div>
          </a-upload>
          <a-modal
            :visible="previewVisible"
            :footer="null"
            @cancel="handleCancel"
          >
            <img alt="example" style="width: 100%" :src="previewImage" />
          </a-modal>
        </div>
      </a-form-item>
      <a-form-item label="個人動態標題" v-bind="validateInfos.title">
        <a-input v-model:value="modelRef.title" />
      </a-form-item>
      <a-form-item label="個人動態內容" v-bind="validateInfos.content">
        <div id="editVditor"></div>
      </a-form-item>
    </a-form>
    <a-form-item :wrapper-col="{ offset: 20 }">
      <a-button type="primary" @click.prevent="onSubmit">修改</a-button>
    </a-form-item>
  </div>
</template>
<script>
  import { defineComponent, reactive, toRaw, ref, onMounted, unref } from 'vue'
  import { Form, message } from 'ant-design-vue'
  import Vditor from 'vditor'
  import { map } from 'lodash-es'

  import { putNewsfeedApi } from '@/api/newsfeed.js'
  import { baseURL } from '@/config/index.js'
  import { getBase64 } from '@/utils/image.js'
  import mybus from '@/utils/mybus.js'

  export default defineComponent({
    setup() {
      const previewVisible = ref(false)
      const contentEditor = ref('')
      const imageUrl = ref('')
      const loading = ref(false)
      const id = ref(undefined)
      const previewImage = ref('')

      const fileList = ref([])

      const modelRef = reactive({
        title: '',
        content: '',
        image: '',
        place: '發自Windows 10',
      })
      // 表單驗證
      const rulesRef = reactive({
        title: [
          {
            required: true,
            message: '請輸入動態時報標題',
          },
        ],
        content: [
          {
            required: true,
            message: '請輸入動態時報標題',
          },
        ],
      })
      const { resetFields, validate, validateInfos } = Form.useForm(
        modelRef,
        rulesRef
      )
      onMounted(() => {
        initVditor()
      })
      // md 編輯器
      const initVditor = () => {
        contentEditor.value = new Vditor('editVditor', {
          value: '',
          height: 360,
          lang: 'zh_TW',
          input: (value) => (modelRef.content = value === '\n' ? '' : value),
          outline: {
            enable: true,
            position: 'right',
          },
          cache: {
            enable: false,
          },
          preview: {
            actions: ['desktop', 'tablet', 'mobile'],
            markdown: {
              autoSpace: false,
              fixTermTypo: true,
            },
            mode: 'both',
            hljs: {
              lineNumber: true,
            },
          },
          tab: '    ',
          resize: {
            enable: true,
          },
          placeholder: '請輸入 MD 格式',
        })
      }
      // 上傳圖片callback
      const handleChange = (info) => {
        if (info.file.status === 'uploading') {
          loading.value = true
          return
        }
        if (info.file.status === 'done') {
          getBase64(info.file.originFileObj, (base64Url) => {
            imageUrl.value = base64Url
            loading.value = false
          })
          modelRef.thumb = info.file.response.result.url
          fileList.value = info.file
        }
        if (info.file.status === 'error') {
          loading.value = false
          message.error('upload error')
        }
      }

      const handleCancel = () => {
        previewVisible.value = false
      }
      const handlePreview = async (file) => {
        previewImage.value = file.url || file.response.result.url
        previewVisible.value = true
      }

      const onSubmit = () => {
        validate().then(async () => {
          modelRef.image = map(
            toRaw(fileList.value),
            (item) => item.url ?? item.response.result.url
          )
          await putNewsfeedApi(unref(id), toRaw(modelRef))
          message.success('修改動態時報成功')
          // 重置
          resetFields()
          // 關閉 add model
          mybus.emit('newsfeedEdit')
        })
      }

      // 建立事件
      mybus.on('newsfeedEditDate', ({ val }) => {
        id.value = val.id
        modelRef.title = val.title
        modelRef.content = val.content
        id.value = val.id
        fileList.value = map(val.image, (item, key) => {
          return {
            uid: key,
            status: 'done',
            url: item,
          }
        })
        contentEditor.value.setValue(val.content)
      })

      return {
        validateInfos,
        modelRef,
        onSubmit,
        imageUrl,
        handleChange,
        loading,
        fileList,
        handlePreview,
        handleCancel,
        previewVisible,
        previewImage,
        baseURL,
      }
    },
  })
</script>
