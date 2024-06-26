<template>
  <div>
    <a-form :labelCol="{ span: 6 }">
      <a-form-item label="文章縮略圖" v-bind="validateInfos.thumb">
        <a-radio-group v-model:value="thumbSelect">
          <a-radio-button value="input">輸入URL</a-radio-button>
          <a-radio-button value="upload">自行上傳</a-radio-button>
        </a-radio-group>
        <div v-if="thumbSelect === 'input'">
          <a-input v-model:value="modelRef.thumb" />
          <a-image :width="300" :src="modelRef.thumb" />
        </div>
        <a-upload
          v-if="thumbSelect === 'upload'"
          list-type="picture-card"
          class="avatar-uploader"
          :show-upload-list="false"
          :action="`${baseURL}/admin/file`"
          @change="handleChange"
        >
          <a-image
            v-if="imageUrl"
            :width="300"
            :src="imageUrl"
            :preview="false"
          />
          <div v-else>
            <loading-outlined v-if="loading"></loading-outlined>
            <plus-outlined v-else></plus-outlined>
            <div class="ant-upload-text">Upload</div>
          </div>
        </a-upload>
      </a-form-item>
      <a-row class="form-row">
        <a-col :lg="12">
          <a-form-item label="類別區塊" v-bind="validateInfos.categorieId">
            <a-tree-select
              :treeDefaultExpandAll="true"
              :replaceFields="replaceFields"
              v-model:value="modelRef.categorieId"
              style="width: 100%"
              :dropdown-style="{ maxHeight: '400px', overflow: 'auto' }"
              :tree-data="treeData"
              placeholder="請選擇類別區塊"
            ></a-tree-select>
          </a-form-item>
        </a-col>
        <a-col :lg="12">
          <a-form-item label="文章標題" v-bind="validateInfos.title">
            <a-input v-model:value="modelRef.title" />
          </a-form-item>
        </a-col>
        <a-col :lg="12">
          <a-form-item label="文章描述" v-bind="validateInfos.description">
            <a-input v-model:value="modelRef.description" />
          </a-form-item>
        </a-col>
        <a-col :lg="12">
          <a-form-item label="Tag" v-bind="validateInfos.tag">
            <a-select
              v-model:value="modelRef.tag"
              mode="tags"
              style="width: 100%"
              :token-separators="[',']"
              placeholder="請輸入文章Tag"
              :options="tagOptions"
            ></a-select>
          </a-form-item>
        </a-col>
        <a-col :lg="12">
          <a-form-item label="文章作者" v-bind="validateInfos.author">
            <a-input v-model:value="modelRef.author" />
          </a-form-item>
        </a-col>
        <a-col :lg="12">
          <a-form-item label="文章推薦狀態">
            <a-radio-group name="radioGroup" v-model:value="modelRef.recommend">
              <a-radio value="0">未推薦</a-radio>
              <a-radio value="1">加入推薦</a-radio>
            </a-radio-group>
          </a-form-item>
        </a-col>
      </a-row>
      <a-form-item
        label="文章內容"
        v-bind="validateInfos.content"
        :labelCol="{ offset: 1 }"
      >
        <div id="vditor"></div>
      </a-form-item>
    </a-form>
    <a-form-item :wrapper-col="{ span: 10, offset: 14 }">
      <a-button type="primary" @click.prevent="onSubmit">創建</a-button>
      <a-button style="margin-left: 10px" @click="resetFields">重置</a-button>
    </a-form-item>
  </div>
</template>
<script>
  import { defineComponent, reactive, toRaw, ref, onMounted } from 'vue'
  import { Form, message } from 'ant-design-vue'
  import Vditor from 'vditor'
  import { useRouter } from 'vue-router'
  import { baseURL } from '@/config/index.js'
  import { getCategorie } from '@/api/categories.js'
  import { createArticle } from '@/api/article.js'

  import { getBase64 } from '@/utils/image.js'

  export default defineComponent({
    setup() {
      const contentEditor = ref()
      const thumbSelect = ref('input')
      const imageUrl = ref('')
      const loading = ref(false)
      const router = useRouter()

      const treeData = ref([
        {
          id: 0,
          name: '頂級目錄',
          children: '',
        },
      ])
      const replaceFields = {
        title: 'name',
        value: 'id',
        key: 'id',
      }

      const tagOptions = ref([
        {
          value: 'js',
          label: 'js',
        },
        {
          value: 'vue',
          label: 'vue',
        },
        {
          value: 'laravel',
          label: 'laravel',
        },
      ])

      const modelRef = reactive({
        title: '',
        description: '',
        categorieId: undefined,
        author: '',
        tag: undefined,
        thumb: '',
        content: '',
        recommend: '0',
      })
      // 表單驗證
      const rulesRef = reactive({
        title: [
          {
            required: true,
            message: '請輸入文章標題',
          },
        ],
        description: [
          {
            required: true,
            message: '請輸入文章描述',
          },
        ],
        categorieId: [
          {
            required: true,
            message: '請選擇文章類別',
          },
        ],
        author: [
          {
            required: true,
            message: '請輸入作者名稱',
          },
        ],
        tag: [
          {
            required: true,
            message: '請輸入文章標籤',
          },
        ],
        // thumb: [
        //   {
        //     required: true,
        //     message: '請上傳文章縮略圖',
        //   },
        // ],
        content: [
          {
            required: true,
            message: '請輸入文章內容',
          },
        ],
      })
      const { resetFields, validate, validateInfos } = Form.useForm(
        modelRef,
        rulesRef
      )
      onMounted(() => {
        initVditor()
        getCategorieApi()
      })
      // md 編輯器
      const initVditor = () => {
        contentEditor.value = new Vditor('vditor', {
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
        }
        if (info.file.status === 'error') {
          loading.value = false
          message.error('upload error')
        }
      }

      const dealWithTreeData = (val) => {
        for (const iterator of val) {
          if (iterator.children) {
            iterator.disabled = true
            dealWithTreeData(iterator.children)
          }
        }
      }
      // 獲取 Categorie 數據
      const getCategorieApi = async () => {
        const { result } = await getCategorie()
        treeData.value[0].children = result
        dealWithTreeData(treeData.value)
      }
      const onSubmit = () => {
        validate().then(async () => {
          await createArticle(toRaw(modelRef))
          message.success('新增文章成功')
          router.push({ name: 'articleList' })
        })
      }

      return {
        validateInfos,
        resetFields,
        modelRef,
        onSubmit,
        treeData,
        replaceFields,
        imageUrl,
        handleChange,
        loading,
        tagOptions,
        baseURL,
        thumbSelect,
      }
    },
  })
</script>
<style>
  @import url(~vditor/dist/index.css);
</style>
