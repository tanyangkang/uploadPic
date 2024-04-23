<template>
    <div :style="{'--compWidth': width, '--compHeight': height}">
        <el-upload :action="action" :file-list="fileList" :disabled="disabled" list-type="picture-card" :headers="headers" :limit="limit" :accept="accept" :class="(limit <= fileList.length) || uploading ? `hideUpload ${uploadCompSize}`  : `${uploadCompSize}`" :on-error="handleError"
            :before-upload="beforeUpload" :on-success="handleSuccess">
            <i slot="default" class="el-icon-plus"></i>
            <div slot="file" slot-scope="{ file }">
                <img v-if="file.fileType == 'image'" class="el-upload-list__item-thumbnail" :src="file.url" alt="" />
                <video v-if="file.fileType == 'video'" controls="controls" :src="file.url" style="max-width:100%;max-height:100%"></video>
                <span v-if="file.status === 'success'" class="el-upload-list__item-actions">
                    <span v-if="preview" class="el-upload-list__item-preview" @click="handleImgPreview(file)">
                        <i class="el-icon-zoom-in"></i>
                    </span>
                    <span v-if="download" class="el-upload-list__item-delete" @click="handleDownload(file)">
                        <i class="el-icon-download"></i>
                    </span>
                    <span v-if="deleted" class="el-upload-list__item-delete" @click="handleRemove(file)">
                        <i class="el-icon-delete"></i>
                    </span>
                </span>
                <span v-else :class="[uploading ? 'uploading' : '','el-upload-list__item-actions']">
                    <i class="el-icon-loading" /><i style="font-size: 14px">上传中</i>
                </span>
            </div>
            <div slot="tip" v-if="tips" class="el-upload__tip">{{tips}}</div>

        </el-upload>
        <el-dialog :visible.sync="previewVisible" title="预览" width="fit-content" append-to-body>
            <img v-if="previewImg.fileType == 'image'" :src="previewImg.url" style="height:60vh;margin: 0 auto;display: block;">
            <video v-if="previewImg.fileType == 'video'" controls="controls" autoplay="autoplay" :src="previewImg.url"  style="height:60vh;margin: 0 auto;display: block;"></video>
            <div slot="footer">
                <el-button @click="previewVisible=false">关闭</el-button>
            </div>
        </el-dialog>
    </div>
</template>
 
<script>
import { getToken } from "@/util/auth";
export default {
    name: 'UploadPic',
    props: {
        // 上传的地址
        action: {
            type: String,
            default: '/api/blade-resource/oss/endpoint/put-file',
        },
        // 禁用
        disabled: {
            type: Boolean,
            default: false
        },
        // 组件大小，也可以传compWidth、compHeight自定义
        uploadCompSize: {
            type: String,
            default: 'small'
        },
        width: {
            type: String,
            default: '100px'
        },
        height: {
            type: String,
            default: '100px'
        },
        // 提示建议
        tips: {
            type: String,
            default: ''
        },
        // v-model 双向绑定的value
        value: {
            type: [Array, String],
            default: () => ([])
        },
        // 设置上传的请求头部
        headers: {
            type: Object,
            default: () => {
                return {
                    Authorization: "Basic c2FiZXI6c2FiZXJfc2VjcmV0",
                    "Blade-Auth": "Bearer " + getToken(),
                }
            },
        },
        // 上传文件大小限制， 默认5120 不限制，单位M
        size: {
            type: [Number, String],
            default: 5120,
        },
        // 文件上传格式， 默认jpeg, png,jpg
        accept: {
            type: String,
            default: 'image/jpeg,image/png,image/jpg,image/gif',
        },
        // 是否显示删除操作按钮
        deleted: {
            type: Boolean,
            default: true,
        },
        // 是否显示预览操作按钮
        preview: {
            type: Boolean,
            default: true,
        },
        // 是否显示下载操作按钮
        download: {
            type: Boolean,
            default: false,
        },
        // 上传文件个数限制，默认1 不限制
        limit: {
            type: [Number, String],
            default: 1,
        },
    },
    data() {
        return {
            fileList: [], // 默认文件列表
            hideUpload: false, // 超出限制隐藏上传按钮
            uploading: false, // 是否上传中,上传时隐藏上传按钮
            previewImg: {url: ''}, // 预览图片地址
            previewVisible: false, // 是否显示预览
        }
    },
    watch: {
        value: {
            handler(newVal) {
                if (newVal && newVal.length) {
                    if (Array.isArray(newVal)) {
                        this.fileList = newVal.filter(Boolean).map((url) => {
                            let res = this.handleCheckFileType(url)
                            return { url,fileType: res}
                        });
                    } else {
                        this.fileList = Array.from([newVal]).filter(Boolean).map((url) => {
                            let res = this.handleCheckFileType(url)
                            return { url,fileType: res }
                        })
                    }
                }
            },
            immediate: true,
            deep: true
        }
    },
    mounted() {
    },
    methods: {
        // 上传前文件大小判断
        beforeUpload(file) {
            const { size,accept } = this;
            const type = file.type;
            if (accept && accept.indexOf(type) === -1) {
                this.$message.error(`上传文件失败，文件只支持${accept}类型！`)
                return false
            }
            const overSize = size > 0 && file.size < 1024 * 1024 * size
            if (!overSize) this.$message.error(`上传文件大小不能超过 ${size}MB!`)
            this.uploading = overSize // 是否上传中
            return overSize
        },
        // 上传出错返回
        handleError(event, file, fileList) {
            this.$message.error('上传失败！')
            this.handleChange()
        },
        handleSuccess(res, file, list) {
            this.fileList.push({ url: res.data.link })
            this.handleChange()
        },
        // 删除图片
        handleRemove(file) {
            const { fileList } = this
            this.fileList = fileList.filter((item) => item.uid !== file.uid)
            this.handleChange();
        },
        handleCheckFileType(file) {
            const index = file.lastIndexOf('.');
            const fileType = file.substr(index + 1)
            let type = '';
            if (['mp4','flv','avi','mov','wmv','mpg'].includes(fileType)) {
                type = 'video'
            } else {
                type = 'image'
            }
            return type
        },
        // 图片预览
        handleImgPreview(file) {
            this.previewImg = file
            this.$nextTick(() => {
                this.previewVisible = true
            })
        },
        handleChange() {
            const { limit, fileList } = this
            if (limit > 0 && fileList.length >= limit) {
                this.hideUpload = true
            } else { this.hideUpload = false }
            this.uploading = false
            let fileArr = []
            if (limit === 1 && fileList.length) {
                fileArr = fileList[0].url || '';
            } else if (!fileList.length) {
                // 解决不必填情况下 默认抛出空字符串
                fileArr = '';
            } else {
                fileList.forEach((item) => fileArr.push(item.url))
            }
            this.$emit('input', fileArr)
        },
        handleDownload(file) {
            const a = document.createElement('a')
            a.href = file.url
            a.click()
        },
    },
}
</script>
 
<style lang="scss">
:root {
    --compWidth: 100px;
    --compHeight: 100px;
}
.hideUpload .el-upload--picture-card {
    display: none;
}
.el-upload--picture-card {
    width: 100px;
    height: 100px;
    line-height: 100px;
}
.mini .el-upload-list--picture-card .el-upload-list__item,
.mini .el-upload--picture-card {
    width: 70px;
    height: 70px;
    line-height: 70px;
}
.small .el-upload-list--picture-card .el-upload-list__item,
.small .el-upload--picture-card {
    width: var(--compWidth);
    height: var(--compHeight);
    line-height: var(--compHeight);
}
.large .el-upload-list--picture-card .el-upload-list__item,
.large .el-upload--picture-card {
    width: 150px;
    height: 150px;
    line-height: 150px;
}
.el-upload-list--picture-card .uploading.el-upload-list__item-actions {
    opacity: 1;
}
.el-upload-list--picture-card .el-upload-list__item {
    width: 100px;
    height: 100px;
}
.el-upload__tip {
    line-height: 20px;
    font-size: 12px;
    color: #999999;
}
/*添加、删除文件时去掉动画过渡*/
.el-upload-list__item {
    transition: none !important;
}
</style>