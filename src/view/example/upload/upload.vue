<template>
  <div v-loading.fullscreen.lock="fullscreenLoading">
    <div class="gva-table-box">
      <warning-bar
        title="点击“文件名/备注”可以编辑文件名或者备注内容。"
      />
      <div class="gva-btn-list">
        <upload-common
          v-model:imageCommon="imageCommon"
          @on-success="getTableData"
        />
        <upload-image
          v-model:imageUrl="imageUrl"
          :file-size="512"
          :max-w-h="1080"
          @on-success="getTableData"
        />
        <span style="width: 120;height: 50;border: 1px solid lightblue;">已选资源:{{ ids }}</span>
        <el-input
          v-model="caption"
          type="text"
          style="width: 600px;white-space: pre-line"
          @mouseleave="showCaption"
        />
        <!-- <el-input v-model="ids" /> -->
        <el-button
          type="primary"
          icon="upload"
          @click="sendTeleBot"
        >发送给bot</el-button>
      </div>

      <!-- 以下是图片/视频 展示区 -->
      <el-table :data="tableData">
        <el-table-column
          align="left"
          label="主键"
          prop="ID"
          min-width="100"
        />
        <el-table-column
          align="left"
          label="预览"
          width="100"
        >
          <template #default="scope">
            <CustomPic
              pic-type="file"
              :pic-src="scope.row.url"
              preview
            />
          </template>
        </el-table-column>
        <el-table-column
          align="left"
          label="日期"
          prop="UpdatedAt"
          width="180"
        >
          <template #default="scope">
            <div>{{ formatDate(scope.row.UpdatedAt) }}</div>
          </template>
        </el-table-column>
        <el-table-column
          align="left"
          label="文件名/备注"
          prop="name"
          width="180"
        >
          <template #default="scope">
            <div
              class="name"
              @click="editFileNameFunc(scope.row)"
            >{{ scope.row.name }}</div>
          </template>
        </el-table-column>
        <el-table-column
          align="left"
          label="链接"
          prop="url"
          min-width="300"
        />
        <el-table-column
          align="left"
          label="标签mike"
          prop="tag"
          width="100"
        >
          <template #default="scope">
            <el-tag
              :type="scope.row.tag === 'jpg' ? 'primary' : 'success'"
              disable-transitions
            >{{ scope.row.tag }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column
          align="left"
          label="操作"
          width="160"
        >
          <template #default="scope">
            <el-button
              icon="download"
              type="primary"
              link
              @click="downloadFile(scope.row)"
            >下载</el-button>
            <el-button
              icon="TurnOff"
              type="primary"
              link
              @click="addForBot(scope.row)"
            >选择</el-button>
          </template>
        </el-table-column>
      </el-table>
      <div class="gva-pagination">
        <el-pagination
          :current-page="page"
          :page-size="pageSize"
          :page-sizes="[10, 30, 50, 100]"
          :style="{ float: 'right', padding: '20px' }"
          :total="total"
          layout="total, sizes, prev, pager, next, jumper"
          @current-change="handleCurrentChange"
          @size-change="handleSizeChange"
        />
      </div>
    </div>
  </div>
</template>

<script setup>
import { getFileList, deleteFile, editFileName } from '@/api/fileUploadAndDownload'
import { downloadImage } from '@/utils/downloadImg'
import CustomPic from '@/components/customPic/index.vue'
import UploadImage from '@/components/upload/image.vue'
import UploadCommon from '@/components/upload/common.vue'
import { formatDate } from '@/utils/format'
import WarningBar from '@/components/warningBar/warningBar.vue'

import { ref } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'

defineOptions({
  name: 'Upload',
})

const ids = ref([])
const mediaPaths = ref([])

const path = ref(import.meta.env.VITE_BASE_API)

const imageUrl = ref('')
const imageCommon = ref('')

const page = ref(1)
const caption = ref('请输入图片集文字描述')
const total = ref(0)
const pageSize = ref(10)
const search = ref({})
const tableData = ref([])

// change tg bot album caption
const showCaption = () => {
  console.log(`描述是${caption.value}`)
}
// 分页
const handleSizeChange = (val) => {
  pageSize.value = val
  getTableData()
}

const handleCurrentChange = (val) => {
  page.value = val
  getTableData()
}

// 查询
const getTableData = async() => {
  const table = await getFileList({ page: page.value, pageSize: pageSize.value, ...search.value })
  if (table.code === 0) {
    tableData.value = table.data.list
    total.value = table.data.total
    page.value = table.data.page
    pageSize.value = table.data.pageSize
  }
}
getTableData()

// 发送tgBot, 借用delete接口
const sendTeleBot = async() => {
  const data = {}
  data['paths'] = mediaPaths.value
  data['caption'] = caption.value
  const respData = await deleteFile(data)
  console.log(`receive tg resp:${respData}`)
}

const downloadFile = (row) => {
  if (row.url.indexOf('http://') > -1 || row.url.indexOf('https://') > -1) {
    downloadImage(row.url, row.name)
  } else {
    debugger
    downloadImage(path.value + '/' + row.url, row.name)
  }
}

const addForBot = (row) => {
  ids.value.push(row.ID)
  mediaPaths.value.push(row.url)
}

/**
 * 编辑文件名或者备注
 * @param row
 * @returns {Promise<void>}
 */
const editFileNameFunc = async(row) => {
  ElMessageBox.prompt('请输入文件名或者备注', '编辑', {
    confirmButtonText: '确定',
    cancelButtonText: '取消',
    inputPattern: /\S/,
    inputErrorMessage: '不能为空',
    inputValue: row.name
  }).then(async({ value }) => {
    row.name = value
    // console.log(row)
    const res = await editFileName(row)
    if (res.code === 0) {
      ElMessage({
        type: 'success',
        message: '编辑成功!',
      })
      getTableData()
    }
  }).catch(() => {
    ElMessage({
      type: 'info',
      message: '取消修改'
    })
  })
}
</script>

<style scoped>
.name {
  cursor: pointer;
}

</style>
