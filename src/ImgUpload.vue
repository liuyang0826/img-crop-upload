<template>
  <el-dialog title="修改头像" :close-on-click-modal="false" width="500px" :visible.sync="visible" @close="close">
    <div class="upload">
      <div class="header">
        <div class="label">上传头像</div>
        <input disabled class="path" v-model="imgPathName">
        <input
          ref="file"
          type="file"
          @change.prevent="fileChange"
          accept="image/png,image/jpg,image/jpeg"
          style="display: none;">
        <el-button size="small" type="primary" @click="handleViewImg">浏览...</el-button>
      </div>
      <div class="body">
        <div
          class="wrap"
          ref="wrap"
          @dragover.prevent
          @dragleave.prevent
          @drop.prevent="fileChange"
          @mousedown.prevent.stop="wrapMousedown"
          :style="{width: wrapStyle.width + 'px', height: wrapStyle.height + 'px', cursor: isShowMask ? 'crosshair' : 'default'}">
          <template v-if="isShowMask">
            <div class="around-mask"
                 :style="{left: 0, top: 0, width: centerLeft + 'px', height: wrapStyle.height + 'px'}"></div>
            <div class="around-mask"
                 :style="{left: centerLeft + 'px', top: 0, width: viewLength + 'px', height: centerTop + 'px'}"></div>
            <div class="around-mask"
                 :style="{left: centerLeft + viewLength + 'px', top: 0, width: wrapStyle.width - centerLeft - viewLength + 'px', height: wrapStyle.height + 'px'}"></div>
            <div class="around-mask"
                 :style="{left: centerLeft + 'px', top: centerTop + viewLength + 'px', width: viewLength + 'px', height: wrapStyle.height - centerTop - viewLength + 'px'}"></div>
            <div class="center"
                 :style="{left: centerLeft + 'px', top: centerTop + 'px', width: viewLength + 'px', height: viewLength + 'px'}"
                 @mousedown.prevent.stop="centerMousedown">
              <div class="point n w" style="cursor: nw-resize" @mousedown.stop.prevent="wLineMousedown"></div>
              <div class="point n hc"></div>
              <div class="point n e" style="cursor: ne-resize" @mousedown.stop.prevent="nLineMousedown"></div>
              <div class="point e vc"></div>
              <div class="point s e" style="cursor: se-resize" @mousedown.stop.prevent="sLineMousedown"></div>
              <div class="point s hc"></div>
              <div class="point s w" style="cursor: sw-resize" @mousedown.stop.prevent="wLineMousedown"></div>
              <div class="point w vc"></div>
              <div class="line w"
                   @mousedown.stop.prevent="wLineMousedown"
                   :style="{cursor: 'w-resize', borderLeftWidth: '1px', width: '6px', height: viewLength + 'px'}"></div>
              <div class="line n"
                   @mousedown.stop.prevent="nLineMousedown"
                   :style="{cursor: 'n-resize', borderTopWidth: '1px', width: viewLength + 'px', height: '6px'}"></div>
              <div class="line e"
                   @mousedown.stop.prevent="eLineMousedown"
                   :style="{cursor: 'e-resize', borderRightWidth: '1px', width: '6px', height: viewLength + 'px'}"></div>
              <div class="line s"
                   @mousedown.stop.prevent="sLineMousedown"
                   :style="{cursor: 'n-resize', borderBottomWidth: '1px', width: viewLength + 'px', height: '6px'}"></div>
            </div>
          </template>
          <img class="left-img" :src="sourceImgUrl" alt="" :width="wrapStyle.width" :height="wrapStyle.height">
        </div>
        <div class="right">
          <div class="preview">
            <canvas ref="canvas" :width="preview.width" :height="preview.height"></canvas>
          </div>
        </div>
      </div>
      <div class="footer">
        <el-button size="medium" type="success" @click="sureUpload" :disabled="isDisabled" :loading="loading">确定</el-button>
      </div>
    </div>
  </el-dialog>
</template>

<script>
export default {
  name: 'ImgUpload',
  props: {
    field: {
      type: String,
      'default': 'userHeaderFile'
    },
    // 上传地址
    url: {
      type: String,
      'default': ''
    },
    // 其他要上传文件附带的数据
    params: {
      type: Object,
      'default': null
    },
    // header请求头
    headers: {
      type: Object,
      'default': null
    },
    // 是否支持跨域
    withCredentials: {
      type: Boolean,
      'default': false
    },
    method: {
      type: String,
      'default': 'POST'
    }
  },
  data () {
    return {
      visible: false,
      isDisabled: true,
      loading: false,
      // 配置长宽
      maxWrap: { width: 350, height: 350 },
      preview: { width: 100, height: 100 },
      // 裁剪相关
      centerStart: { x: 0, y: 0 },
      viewLength: 0,
      origin: { x: 0, y: 0 },
      wrapStyle: { width: 280, height: 280 },
      moveStart: { x: 0, y: 0 },
      moved: { x: 0, y: 0 },
      isShowMask: false,
      // 文件相关
      imgPathName: '',
      sourceImgUrl: '',
      imgWH: { width: 0, height: 0 }
    }
  },
  computed: {
    centerLeft () { return this.centerStart.x + this.moved.x },
    centerTop () { return this.centerStart.y + this.moved.y },
    maxLength () { return Math.min(this.wrapStyle.width - this.centerLeft, this.wrapStyle.height - this.centerTop) }
  },
  watch: {
    imgWH: {
      handler ({ width, height }) {
        this.scaleB = this.maxWrap.width / width
        if (width < height) { this.scaleB = this.maxWrap.height / height }
        width *= this.scaleB
        height *= this.scaleB
        this.viewLength = Math.min(width, height)
        this.wrapStyle = { width, height }
      },
      deep: true
    },
    // 实时绘制
    centerLeft () { this.drawPreview() },
    centerTop () { this.drawPreview() },
    viewLength () { this.drawPreview() }
  },
  beforeDestroy () {
    this.ctx = null
  },
  methods: {
    init () {
      this.reset()
      this.visible = true
      this.$nextTick(() => {
        this.origin = this.getOrigin(this.$refs.wrap)
        if (!this.inited) {
          this.ctx = this.$refs.canvas.getContext('2d')
          this.inited = true
        }
      })
    },
    // 重置
    reset () {
      this.wrapStyle = { width: 280, height: 280 }
      this.centerStart = { x: 0, y: 0 }
      this.moveStart = { x: 0, y: 0 }
      this.moved = { x: 0, y: 0 }
      this.imgPathName = ''
      this.sourceImgUrl = ''
      this.viewLength = 0
      this.isShowMask = false
      this.isDisabled = true
    },
    // 关闭
    close () {
      this.$refs.file.value = ''
      this.visible = false
    },
    getOrigin (dom) {
      let offset = {x: 0, y: 0}
      while (dom) {
        offset.x += dom.offsetLeft
        offset.y += dom.offsetTop
        dom = dom.offsetParent
      }
      return offset
    },
    // 图片裁剪交互相关
    // 裁剪
    wrapMousedown (e) {
      if (!this.isShowMask) { return }
      this.centerStart = { x: e.clientX - this.origin.x, y: e.clientY - this.origin.y }
      this.cutStart = { x: e.clientX, y: e.clientY }
      this.moved = { x: 0, y: 0 }
      this.viewLength = 0
      document.addEventListener('mousemove', this.wrapMousemove)
      document.addEventListener('mouseup', this.wrapMouseup)
    },
    wrapMousemove (e) {
      e.preventDefault()
      e.stopPropagation()
      let { clientX, clientY } = e
      let { origin: { x, y } } = this
      let xChange = clientX - this.cutStart.x
      let yChange = clientY - this.cutStart.y
      let length
      if (Math.abs(clientY - y - this.centerTop) > Math.abs(clientX - x - this.centerLeft)) {
        length = this.viewLength + yChange
        this.cutStart = { x: clientX, y: y + this.centerTop + length }
      } else {
        length = this.viewLength + xChange
        this.cutStart = { x: x + this.centerLeft + length, y: e.clientY }
      }
      if (length > this.maxLength) {
        length = this.maxLength
      } else if (length < 0) {
        length = 0
      }
      this.viewLength = length
    },
    wrapMouseup (e) {
      e.preventDefault()
      e.stopPropagation()
      document.removeEventListener('mousemove', this.wrapMousemove)
      document.removeEventListener('mouseup', this.wrapMouseup)
    },
    // 拖动
    centerMousedown (e) {
      this.moveStart = { x: e.clientX - this.moved.x, y: e.clientY - this.moved.y }
      document.addEventListener('mousemove', this.centerMousemove)
      document.addEventListener('mouseup', this.centerMouseup)
    },
    centerMousemove (e) {
      let x = e.clientX - this.moveStart.x
      let y = e.clientY - this.moveStart.y
      if (this.centerStart.x + x < 0) {
        x = -this.centerStart.x
      } else if (this.centerStart.x + x > this.wrapStyle.width - this.viewLength) {
        x = this.wrapStyle.width - this.centerStart.x - this.viewLength
      }
      if (this.centerStart.y + y < 0) {
        y = -this.centerStart.y
      } else if (this.centerStart.y + y > this.wrapStyle.height - this.viewLength) {
        y = this.wrapStyle.height - this.centerStart.y - this.viewLength
      }
      this.moved = {x, y}
    },
    centerMouseup (e) {
      e.preventDefault()
      e.stopPropagation()
      document.removeEventListener('mousemove', this.centerMousemove)
      document.removeEventListener('mouseup', this.centerMouseup)
    },
    // 边框放缩
    // →
    eLineMousedown (e) {
      this.startEX = e.clientX
      document.addEventListener('mousemove', this.eLineMousemove)
      document.addEventListener('mouseup', this.eLineMouseup)
    },
    eLineMousemove (e) {
      let change = e.clientX - this.startEX
      if (change > 0 && this.viewLength === this.maxLength) return
      let length = this.viewLength + change
      if (length > this.maxLength) {
        length = this.maxLength
      } else if (length < 0) {
        length = 0
      }
      this.startEX = this.origin.x + this.centerLeft + length
      this.viewLength = length
    },
    eLineMouseup (e) {
      e.preventDefault()
      e.stopPropagation()
      document.removeEventListener('mousemove', this.eLineMousemove)
      document.removeEventListener('mouseup', this.eLineMouseup)
    },
    // ↓
    sLineMousedown (e) {
      this.startSY = e.clientY
      document.addEventListener('mousemove', this.sLineMousemove)
      document.addEventListener('mouseup', this.sLineMouseup)
    },
    sLineMousemove (e) {
      let change = e.clientY - this.startSY
      if (change > 0 && this.viewLength === this.maxLength) return
      let length = this.viewLength + change
      if (this.viewLength + change > this.maxLength) {
        length = this.maxLength
      } else if (length < 0) {
        length = 0
      }
      this.startSY = this.origin.y + this.centerTop + length
      this.viewLength = length
    },
    sLineMouseup (e) {
      e.preventDefault()
      e.stopPropagation()
      document.removeEventListener('mousemove', this.sLineMousemove)
      document.removeEventListener('mouseup', this.sLineMouseup)
    },
    // ←
    wLineMousedown (e) {
      this.startWX = e.clientX
      document.addEventListener('mousemove', this.wLineMousemove)
      document.addEventListener('mouseup', this.wLineMouseup)
    },
    wLineMousemove (e) {
      let change = e.clientX - this.startWX
      let maxLength = this.wrapStyle.height - this.centerTop
      if (change < 0 && (maxLength === this.viewLength || !this.centerLeft)) return
      this.startWX = e.clientX
      let length = this.viewLength - change
      if (length > maxLength) {
        change = this.viewLength - maxLength
      } else if (length < 0) {
        change = this.viewLength
      }
      this.startWX = this.origin.x + this.centerLeft + change
      let y = this.centerLeft + change
      if (y < 0) {
        change = -this.centerLeft
        this.startWX = this.origin.x
      }
      length = this.viewLength - change
      this.centerStart.x += change
      this.viewLength = length
    },
    wLineMouseup (e) {
      e.preventDefault()
      e.stopPropagation()
      document.removeEventListener('mousemove', this.wLineMousemove)
      document.removeEventListener('mouseup', this.wLineMouseup)
    },
    // ↑
    nLineMousedown (e) {
      this.startNY = e.clientY
      document.addEventListener('mousemove', this.nLineMousemove)
      document.addEventListener('mouseup', this.nLineMouseup)
    },
    nLineMousemove (e) {
      let change = e.clientY - this.startNY
      let maxLength = this.wrapStyle.width - this.centerLeft
      if (change < 0 && (maxLength === this.viewLength || !this.centerTop)) return
      this.startNY = e.clientY
      let length = this.viewLength - change
      if (length > maxLength) {
        change = this.viewLength - maxLength
      } else if (length < 0) {
        change = this.viewLength
      }
      this.startNY = this.origin.y + this.centerTop + change
      let y = this.centerTop + change
      if (y < 0) {
        change = -this.centerTop
        this.startNY = this.origin.y
      }
      length = this.viewLength - change
      this.centerStart.y += change
      this.viewLength = length
    },
    nLineMouseup (e) {
      e.preventDefault()
      e.stopPropagation()
      document.removeEventListener('mousemove', this.nLineMousemove)
      document.removeEventListener('mouseup', this.nLineMouseup)
    },
    // 图片文件相关
    // 点击选择图片文件
    handleViewImg () {
      this.$refs.file.click()
    },
    // 文件读取
    async fileChange ({target, dataTransfer}) {
      let file = (target.files || dataTransfer.files)[0]
      if (!file) { return }
      this.reset()
      if (!file.type.includes('image')) {
        this.imgPathName = '请选择正确的图片格式'
        return
      }
      this.imgPathName = file.name
      // 读取图片
      const data = await this.readImg(file)
      // 获取宽高
      this.imgWH = await this.getWidthHeight(data)
      this.isShowMask = true
      // 设置资源路径
      this.sourceImgUrl = data
      this.isDisabled = false
    },
    // 读取图片
    readImg (file) {
      return new Promise(resolve => {
        let fr = new FileReader()
        fr.onload = ({target}) => {
          fr.onload = null
          resolve(target.result)
        }
        fr.readAsDataURL(file)
      })
    },
    // 获取长宽
    getWidthHeight (data) {
      return new Promise(resolve => {
        const image = new Image()
        image.onload = () => {
          image.onload = null
          let { width, height } = image
          this.image = image
          resolve({ width, height })
        }
        image.src = data
      })
    },
    // 绘制右侧预览图
    drawPreview () {
      let { ctx, image, centerLeft, centerTop, viewLength, scaleB, preview } = this
      ctx.clearRect(0, 0, preview.width, preview.height)
      ctx.drawImage(image, centerLeft / scaleB, centerTop / scaleB, viewLength / scaleB, viewLength / scaleB, 0, 0, preview.width, preview.height)
    },
    // 文件上传相关
    // 确认上传
    async sureUpload () {
      this.loading = true
      const { field, params } = this
      const fmData = new FormData()
      fmData.append(field, this.canvas2Blob(), field + '.' + 'png')
      // 添加其他参数
      if (typeof params === 'object' && params) {
        Object.keys(params).forEach((k) => { fmData.append(k, params[k]) })
      }
      try {
        this.$emit('uploadSuccess', await this.upload(fmData))
      } catch (e) {
        this.$emit('uploadError', e)
      } finally {
        this.loading = false
      }
      // this
    },
    // 创建XHR开始上传
    upload (fmData) {
      const { headers, withCredentials, method, url } = this
      return new Promise(function (resolve, reject) {
        const xhr = new XMLHttpRequest()
        xhr.open(method, url, true)
        xhr.withCredentials = withCredentials
        xhr.onreadystatechange = function () {
          if (this.readyState !== 4) { return }
          if (this.status === 200 || this.status === 201) {
            resolve(JSON.parse(this.responseText))
          } else {
            reject(this.status)
          }
        }
        // 设置header
        if (typeof headers === 'object' && headers) {
          Object.keys(headers).forEach((k) => {
            xhr.setRequestHeader(k, headers[k])
          })
        }
        xhr.send(fmData)
      })
    },
    // canvas转blob数据
    canvas2Blob () {
      let data = window.atob(this.$refs.canvas.toDataURL().split(',')[1])
      var buffer = new Uint8Array(new ArrayBuffer(data.length))
      for (var i = 0; i < data.length; i++) {
        buffer[i] = data.charCodeAt(i)
      }
      let Builder = window.WebKitBlobBuilder || window.MozBlobBuilder || window.MSBlobBuilder
      let blob

      if (Builder) {
        let builder = new Builder()
        builder.append(buffer)
        blob = builder.getBlob('image/png')
      } else {
        blob = new window.Blob([buffer], {type: 'image/png'})
      }
      return blob
    }
  }
}
</script>

<style scoped lang="scss">
  .upload {
    width: 460px;
  }
  .header {
    display: flex;
    align-items: center;
    margin: -10px 0 10px;
    font-size: 14px;
    .path {
      width: 270px;
      height: 32px;
      padding-left: 5px;
      margin-left: 18px;
      margin-right: 10px;
      line-height: 32px;
      cursor: text;
      border: 1px solid #e0e0e0;
      border-radius: 4px;
    }
  }
  .body {
    display: flex;
    justify-content: space-between;
    user-select: none;
    .right {
      position: relative;
      width: 100px;
      min-height: 100px;
    }
  }
  .preview {
    position: absolute;
    top: 50%;
    transform: translateY(-50px);
    width: 100px;
    height: 100px;
    overflow: hidden;
    border-radius: 50px;
  }
  .wrap {
    max-width: 350px;
    flex: none;
    position: relative;
    .around-mask {
      position: absolute;
      z-index: 10;
      background-color: rgba(122, 122, 122, .4);
    }
    .center {
      position: absolute;
      z-index: 10;
      cursor: move;
    }
    .line {
      position: absolute;
      z-index: 20;
      border-style: dashed;
      border-width: 0;
      border-color: #666;
      &.n {
        top: 0;
      }
      &.e {
        right: 0;
      }
      &.s {
        bottom: 0;
      }
      &.w {
        left: 0;
      }
    }
    .point {
      position: absolute;
      z-index: 30;
      width: 7px;
      height: 7px;
      background-color: #777;
      &.n {
        top: -4px;
      }
      &.e {
        right: -4px;
      }
      &.s {
        bottom: -4px;
      }
      &.w {
        left: -4px;
      }
      &.hc {
        left: 50%;
        transform: translateX(-50%);
      }
      &.vc {
        top: 50%;
        transform: translateY(-50%);
      }
    }
  }
  .left-img {
    display: block;
    box-sizing: border-box;
    border: 1px solid #dadada;
  }
  .footer {
    margin-top: 30px;
    display: flex;
    justify-content: flex-end;
  }
</style>
