<template>
  <div id="app">
    <input type="file" @change="uploadFile">{{ precent }}%
    <button type="button" v-if="!isStop" @click="stopUpload">暂停</button>
    <button type="button" v-else @click="reupload">继续上传</button>
  </div>
</template>

<script>
import SparkMD5 from 'spark-md5'

export default {
  name: 'App',
  data() {
    return {
      remainChunks: [], // 剩余切片
      isStop: false, // 暂停上传控制
      precent: 0, // 上传百分比
      uploadedChunkSize: 0, // 已完成上传的切片数
      chunkSize: 10 * 1024 // 切片大小
    }
  },
  methods: {
    stopUpload() {
      this.isStop = true
    },
    reupload() {
      this.isStop = false
      this.mergeRequest()
    },
    mergeRequest() {
      const chunks = this.remainChunks
      const fileInfo = this.fileInfo
      this.sendRequest(chunks, 6, () => {
        // 请求合并
        this.chunkMerge(fileInfo)
      })
    },
    async uploadFile(e) {
      const file = e.target.files[0]
      this.precent = 0
      this.uploadedChunkSize = 0
      // 如果文件大于分片大小5倍，则进行分片上传
      if (file.size < this.chunkSize * 5) {
        this.sendFile(file)
      } else {
        const chunkInfo = await this.cutBlob(file)
        this.remainChunks = chunkInfo.chunkArr
        this.fileInfo = chunkInfo.fileInfo

        this.mergeRequest()
      }
    },
    cutBlob(file) {
      const chunkArr = [] // 所有切片缓存数组
      const blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice
      const spark = new SparkMD5.ArrayBuffer()
      const chunkNums = Math.ceil(file.size / this.chunkSize) // 切片总数

      return new Promise((resolve, reject) => {
        const reader = new FileReader()
        reader.readAsArrayBuffer(file)
        reader.addEventListener('loadend', () => {
          const content = reader.result

          spark.append(content)
          const hash = spark.end()

          let startIndex = ''
          let endIndex = ''
          let contentItem = ''

          for(let i = 0; i < chunkNums; i++) {
            startIndex = i * this.chunkSize
            endIndex = (i + 1) * this.chunkSize
            endIndex > file.size && (endIndex = file.size)

            contentItem = blobSlice.call(file, startIndex, endIndex)

            chunkArr.push({
              index: i,
              hash,
              total: chunkNums,
              name: file.name,
              size: file.size,
              chunk: contentItem
            })
          }
          resolve({
            chunkArr,
            fileInfo: {
              hash,
              name: file.name,
              size: file.size
            }
          })
        })
        reader.addEventListener('error', function _error(err) {
          reject(err)
        })
      })
    },
    sendRequest(arr, max = 6, callback) {
      let fetchArr = []

      let toFetch = () => {
        if (this.isStop) {
          return Promise.reject('暂停上传')
        }
        if (!arr.length) {
          return Promise.resolve()
        }

        const chunkItem = arr.shift()
        const it = this.sendChunk(chunkItem)
        it.then(() => {
          fetchArr.splice(fetchArr.indexOf(it), 1)
        }, err => {
          this.isStop = true
          arr.unshift(chunkItem)
          Promise.reject(err)
        })
        fetchArr.push(it)

        let p = Promise.resolve()
        if (fetchArr.length >= max) {
          p = Promise.race(fetchArr)
        }

        return p.then(() => toFetch())
      }

      toFetch().then(() => {
        Promise.all(fetchArr).then(() => {
          callback()
        })
      }, err => {
        console.log(err)
      })
    },
    sendChunk(item) {
      if (!item) return
      let formdata = new FormData()
      formdata.append("file", item.chunk)
      formdata.append("hash", item.hash)
      formdata.append("index", item.index)
      formdata.append("name", item.name)

      return this.$http({
        url: "/upload/snippet",
        method: "post",
        data: formdata,
        headers: { "Content-Type": "multipart/form-data" },
        onUploadProgress: (e) => {
          const { loaded, total } = e
          this.uploadedChunkSize += loaded < total ? 0 : +loaded
          this.uploadedChunkSize > item.size && (this.uploadedChunkSize = item.size)

          this.precent = (this.uploadedChunkSize / item.size).toFixed(2) * 1000 / 10
        }
      })
    },
    sendFile(file) {
      let formdata = new FormData()
      formdata.append("file", file)

      this.$http({
        url: "/upload/file",
        method: "post",
        data: formdata,
        headers: { "Content-Type": "multipart/form-data" }
      }).then(({ data }) => {
        console.log(data, 'upload/file')
      })
    },
    chunkMerge(data) {
      this.$http({
        url: "/upload/merge",
        method: "post",
        data,
      }).then(res => {
        console.log(res.data)
      })
    }
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
