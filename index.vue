<template>
  <div class="upload-box">
    <template v-if="type === 'single'">
      <Button v-show="!fileList[0]" :id="btnId" class="upload-btn" :loading="uplaodIng" :disabled="disabled">
        <span style="margin-left: 5px;">{{ btnName }}</span>
      </Button>
      <div class="progress-container" v-if="fileList[0]">
        <div class="info-box flex" style="position: relative;">
          <i class="iconfont icon-attachment" style="margin-right: 10px;"></i>
          <span class="link-name" :style="{'max-width': textMaxWidth, 'min-width': textMinWidth}">{{fileList[0].name}}</span>
          <i class="iconfont icon-delete" style="margin-left: 20px;" @click="handleRemove(fileList[0].name)"></i>
        </div>
        <!-- 成功后不显示进度条 -->
        <Progress v-if="fileList[0].percent !== 101" :percent="fileList[0].percent" hide-info :stroke-width="2" :stroke-color="fileList[0].status === 4 ? '#EB4245' : '#3B7EFF' " />
      </div>
    </template>
  </div>
</template>

<script>
import plupload from 'plupload'
import { uploadGetAuth } from '../../api/auth'
import { runEnv, dateFormat } from '../../utils/utils'
import Cookie from 'js-cookie'

export default {
  name: 'TalOssUploadUi',
  props: {
    value: {
      type: Array,
      default: () => []
    },
    btnName: {
      type: String,
      default: '上传图片'
    },
    // 是否必传
    required: {
      type: Boolean,
      default: false
    },
    // 员工编号
    empNo: {
      type: String,
      default: JSON.parse(Cookie.get('columbusUserInfo')).id
    },
    // 平台,标识文件存放位置
    platform: {
      type: String,
      required: true
    },
    // 类型，多选or单选
    type: {
      type: String,
      default: 'single' // mutil
    },
    // 文件大小限制
    maxFileSize: {
      type: [Number, String],
      default: '5mb'
    },
    // 是否开启文件限制,开启后可以一次选择多个文件
    multiSelection: {
      type: Boolean,
      default: false
    },
    // 同时上传文件限制
    multiLength: {
      type: Number,
      default: 1
    },
    // 最多上传文件个数
    maxLength: {
      type: Number,
      default: 99
    },
    // 文件支持类型
    extensions: {
      type: Array,
      default: function () {
        return [
          {
            title: 'mime supprot',
            extensions:
              'pptx,ppt,pptm,xls,xlt,xlsx,xltx,xlsb,xlsm,xltm,doc,dot,wps,docx,dotx,docm,dotm,pdf,txt,jpg,png,pdf,jpeg,PPTX,PPT,PPTM,XLS,XLT,XLSX,XLTX,XLSB,XLSM,XLTM,DOC,DOT,WPS,DOCX,DOTX,DOCM,DOTM,PDF,TXT,JPG,PNG,PDF,JPEG'
          }
        ]
      }
    },
    // 文件文本区域宽度
    textMaxWidth: {
      type: String,
      default: '990'
    },
    textMinWidth: {
      type: String,
      default: '150'
    }
  },
  watch: {
    fileList () {
      this.$emit('input', this.fileList)
    },
    value () {
      this.fileList = this.value
    }
  },
  data () {
    const btnId = 'upload_btn_' + Math.random() * 10000
    return {
      btnId,
      uplaodIng: false,
      disabled: false,
      fileList: []
    }
  },
  mounted () {
    this.fileList = [...this.value]
    // 避免过早初始化，dom没有找到
    setTimeout(() => {
      if (this.type === 'single') {
        this.initSingleUploadFile()
      }
    }, 500)
  },
  methods: {
    // 验证是否有正在上传中文件
    validate () {
      return new Promise((resolve, reject) => {
        if (this.fileList.length === 0 && this.required) {
          return reject(new Error('请上传文件'))
        }
        let isOk = true
        this.fileList.forEach((item) => {
          if (item.status !== 5) {
            isOk = false
          }
        })
        if (isOk) {
          resolve()
        } else {
          reject(new Error('文件正在上传中'))
        }
      })
    },
    guid () {
      return 'xxxxxxxxxxxx4xxxyxxxxxxxxxxxxxxx'.replace(/[xy]/g, function (c) {
        let r = (Math.random() * 16) | 0
        let v = c === 'x' ? r : (r & 0x3) | 0x8
        return v.toString(16)
      })
    },
    handleRemove (fileName) {
      // 从上传队列中删除当前文件
      this.uploader.files.forEach((f, i) => {
        if (f.name === fileName) {
          this.uploader.splice(i, 1)
        }
      })
      this.uplaodIng = false
      this.fileList = this.fileList.filter((f) => f.name !== fileName)
      this.$emit('on-change', this.fileList)
    },
    // 上传单文件
    initSingleUploadFile () {
      // File.status 1 QUEUED 2 UPLOADING 4 FAILED 5 DONE
      let that = this
      let uploader = (this.uploader = new plupload.Uploader({
        runtimes: 'html5,html4',
        browse_button: that.btnId, // 绑定上传的按钮id
        multi_selection: that.type === 'single' ? false : that.multiSelection, // 是否多选
        url: `${runEnv() === 'prod' ? 'https' : 'http'}://oss.aliyuncs.com`, // 上传url
        filters: {
          mime_types: that.extensions, // 文件类型
          max_file_size: that.maxFileSize, // 最大只能上传5mb的文件
          prevent_duplicates: false // 不允许选取重复文件
        },
        max_retries: 3, // 上传失败，重试次数
        init: {
          PostInit: function () {
            // 当Init事件发生后触发
          },
          FilesAdded: async function (up, files) {
            if (files.length > that.multiLength) {
              that.$Message.warning({
                content: `最多同时上传${that.multiLength}个`,
                duration: 5
              })
              return
            }
            that.uplaodIng = true
            try {
              const authObj = await uploadGetAuth({
                fileName: files[0].name,
                empNo: that.empNo,
                systemName: that.platform
              })
              that.authObj = authObj
              let hasNewFile = false
              files.forEach((file) => {
                let flag =
                  that.fileList.filter((f) => f.name === file.name).length > 0
                if (flag) {
                  that.$Message.warning({
                    content: `图片${file.name}已上传，请勿重复上传`,
                    duration: 5
                  })
                  up.removeFile(file)
                } else {
                  hasNewFile = true
                  file.key =
                    authObj.dir +
                    dateFormat(Date.now(), 'HH:mm:ss') +
                    '-' +
                    file.name
                  that.fileList.push({
                    path: `${authObj.host}/${file.key}`,
                    name: file.name,
                    status: 2,
                    percent: 0,
                    key: file.key
                  })
                }
                if (hasNewFile) {
                  up.start()
                }
              })
            } catch (error) {
              that.uplaodIng = false
              that.$Message.warning({
                content: `文件上传失败: ${error.message}`,
                duration: 5
              })
            }
          },
          // 上传前 setOption在此配置
          async BeforeUpload (up, file) {
            // const authObj = that.authObj
            let authObj = that.authObj
            // 当签名有效期小于10s时重新获取
            if (authObj.expire - Date.now() / 1000 < 10) {
              uploadGetAuth({
                fileName: file.name,
                empNo: that.empNo,
                systemName: that.platform
              })
                .then((res) => {
                  that.authObj = res
                })
                .catch((error) => {
                  that.$Message.warning({
                    content: `文件上传失败：${error.message}`,
                    duration: 5
                  })
                })
            }
            file.key =
              authObj.dir + dateFormat(Date.now(), 'HH:mm:ss') + '-' + file.name
            that.fileList.forEach((f) => {
              if (f.name === file.name) {
                f.path = `${authObj.host}/${file.key}`
              }
            })
            let newMultipartParams = {
              key: file.key,
              policy: authObj.policy,
              expire: authObj.expire,
              OSSAccessKeyId: authObj.accessid,
              success_action_status: '200', // 让服务端返回200,不然，默认会返回204
              // 'callback': res.callback, // 设置回调
              signature: authObj.signature
            }
            up.setOption({
              url: authObj.host,
              multipart_params: newMultipartParams
            })
          },
          // 上传进度 更新上传进度
          UploadProgress: function (up, file) {
            let fileItem = that.fileList.filter((f) => f.name === file.name)[0]
            if (fileItem) {
              fileItem.percent = file.percent
            }
          },
          // 一个文件上传成功
          FileUploaded: function (up, file, info) {
            let fileItem = that.fileList.filter((f) => f.name === file.name)[0]
            if (fileItem) {
              fileItem.status = 5
              setTimeout(() => {
                fileItem.percent = 101 // 判断进度条是否展示， 如果判断100太快，看不到进度条
              }, 1000)

              that.$emit('on-file-uploaded', fileItem)
            }
          },
          // 全部文件上传成功
          UploadComplete (up, files) {
            that.uplaodIng = false
          },
          Error: function (up, err) {
            that.uplaodIng = false
            // 错误文件冒红
            that.fileList.forEach((f) => {
              if (f.name === err.file.name) {
                f.status = 4
              }
            })
            // 错误回调
            that.$Message.warning({
              content: '上传失败: ' + err.message,
              duration: 5
            })
          }
        }
      }))
      uploader.init()
    },
    //   初始化上传
    initUploadFile () {
      const that = this
      let uploader = (this.uploader = new plupload.Uploader({
        runtimes: 'html5,html4',
        browse_button: that.btnId,
        multi_selection: that.multiSelection, // 是否多选
        url: `${runEnv() === 'prod' ? 'https' : 'http'}://oss.aliyuncs.com`,
        filters: {
          mime_types: [
            // 只允许上传图片和zip文件
            { title: 'extensions supprot', extensions: that.extensions }
          ],
          max_file_size: that.maxFileSize, // 最大只能上传5mb的文件
          prevent_duplicates: false // 不允许选取重复文件
        },
        init: {
          PostInit: function () {
            // 当Init事件发生后触发
          },
          FilesAdded: async function (up, files) {
            if (files.length > that.multiLength) {
              that.$Message.warning({
                content: `最多同时上传${that.multiLength}个`,
                duration: 5
              })
              return
            }
            const empNo = JSON.parse(Cookie.get('columbusUserInfo')).id

            try {
              const authObj = await uploadGetAuth({
                fileName: files[0].name,
                empNo
              })
              that.authObj = authObj
              let hasNewFile = false
              files.forEach((file) => {
                let flag =
                  that.fileList.filter((f) => f.name === file.name).length > 0
                if (flag) {
                  that.$Message.warning({
                    content: `图片${file.name}已上传，请勿重复上传`,
                    duration: 5
                  })
                  up.removeFile(file)
                } else {
                  hasNewFile = true
                  let key =
                    authObj.dir + that.guid() + '.' + file.name.split('.')[1]
                  file.key = key
                  that.fileList.push({
                    path: `${authObj.host}/${key}`,
                    name: file.name,
                    status: 2,
                    percent: 0,
                    key: file.key
                  })
                }

                if (hasNewFile) {
                  up.start()
                }
              })
            } catch (error) {
              that.$Message.warning({
                content: '文件上传失败，请重新操作',
                duration: 5
              })
            }
          },
          // 上传前 setOption在此配置
          async BeforeUpload (up, file) {
            // const authObj = that.authObj
            let authObj = that.authObj
            // 当签名有效期小于10s时重新获取
            if (authObj.expire - Date.now() / 1000 < 10) {
              const empNo = JSON.parse(Cookie.get('columbusUserInfo')).id
              uploadGetAuth({ fileName: file.name, empNo })
                .then((res) => {
                  that.authObj = res
                })
                .catch((error) => {
                  that.$Message.warning({
                    content: `文件上传失败：${error.message}`,
                    duration: 5
                  })
                })
            }

            file.key = authObj.dir + that.guid() + '.' + file.name.split('.')[1]

            that.fileList.forEach((f) => {
              if (f.name === file.name) {
                f.path = `${authObj.host}/${file.key}`
              }
            })

            let newMultipartParams = {
              key: file.key,
              policy: authObj.policy,
              expire: authObj.expire,
              OSSAccessKeyId: authObj.accessid,
              success_action_status: '200', // 让服务端返回200,不然，默认会返回204
              // 'callback': res.callback, // 设置回调
              signature: authObj.signature
            }
            up.setOption({
              url: authObj.host,
              multipart_params: newMultipartParams
            })
          },
          // 上传进度
          UploadProgress: function (up, file) {
            let fileItem = that.fileList.filter((f) => f.name === file.name)[0]
            if (fileItem) {
              fileItem.percent = file.percent
            }
          },
          // 上传完成
          FileUploaded: function (up, file, info) {
            let fileItem = that.fileList.filter((f) => f.name === file.name)[0]
            if (fileItem) {
              fileItem.status = 'finished'
              that.$emit('on-change', that.fileList)
            }
          },
          Error: function (up, err, ...rest) {
            console.log('err', err)
            that.fileList = that.fileList.filter(
              (f) => f.name !== err.file.name
            )
            // 错误回调
            if (err.code === -600) {
              that.$Message.warning({
                content: '您上传的文件超过大小限制' + that.maxFileSize,
                duration: 5
              })
            } else if (err.code === -601) {
              that.$Message.warning({
                content: '选择的文件格式错误，目前仅支持' + ExtensionsSupport,
                duration: 5
              })
            } else if (err.code === -602) {
              that.$Message.warning({
                content: '这个文件已经上传过一遍了',
                duration: 5
              })
            } else {
              that.$Message.warning({
                content: '上传错误，请稍后再试',
                duration: 5
              })
            }
          }
        }
      }))
      uploader.init()
    },
    handleClick (e) {
      this.$Message.warning({
        content: `最多上传${this.maxLength}个文件`,
        duration: 10
      })
    }
  }
}
</script>

<style lang="less" scoped>
.upload-box {
  position: relative;
  min-height: 32px;
}
.upload-btn {
  font-size: 14px;
  border: 1px solid #ccc;
  min-width: 112px;
  height: 32px;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  position: relative;
  border-radius: 4px !important;
}

// 上传进度条
.progress-container {
  min-width: 260px;
  position: absolute;
  top: 0;
  left: 0;
  background: #fff;
  z-index: 100;
  height: 32px;
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.info-box {
  height: 20px;
}

.link-name {
  font-size: 14px;
  min-width: 150px;
  overflow-x: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex-grow: 1;
}

.icon-delete,
.icon-attachment {
  font-size: 18px;
  cursor: pointer;
  color: #7b848c;
}

.prevent {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0);
}

.flex {
  display: flex;
  align-items: center;
  // justify-content: space-between;
}
</style>
