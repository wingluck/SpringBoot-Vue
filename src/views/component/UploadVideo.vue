<template>
    <div class="upload">
        <div id="video_container" class="container">
            <el-button id="pickfiles" type="primary"><i class="el-icon-upload"></i>&nbsp;上传视频</el-button>
            <el-progress :percentage="filePercent"></el-progress>
            <p>
                共{{ fileSize }}MB &nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp; 已上传{{ fileLoaded }}&nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;
                上传速度{{ fileSpeed }}/s
            </p>
            <div class="btn-container">
                <el-button @click="pauseUpload" type="primary"><i class="el-icon-video-pause"></i>&nbsp;暂停上传</el-button>
                <el-button @click="continueUpload" type="primary"><i class="el-icon-video-play">&nbsp;</i>继续上传</el-button>
            </div>
        </div>
        <div class="video-url" v-if="videoUrl">
            <video style="width: 320px; height: 180px" :src="videoUrl" controls="controls"></video>
        </div>
    </div>
</template>

<script>
require('qiniu-js/dist/qiniu.min.js');
export default {
    name: 'uploadvideo',
    data() {
        return {
            fileSize: 0,
            fileLoaded: 0,
            fileSpeed: 0,
            filePercent: 0,
            uploader: null,
            token: '',
            filename: '',
            hash: '',
            resFileName: '',
            coverPic: '',
            videoUrl: '',
            phoneType: null
        };
    },
    methods: {
        pauseUpload() {
            this.uploader.stop();
        },
        continueUpload() {
            this.uploader.start();
        },
        toDecimal(size) {
            size = size / 1024 / 1024;
            var f = parseFloat(size);
            if (isNaN(f)) {
                return;
            }
            f = Math.round(size * 10) / 10;
            var s = f.toString();
            var rs = s.indexOf('.');
            if (rs < 0) {
                rs = s.length;
                s += '.';
            }
            while (s.length <= rs + 1) {
                s += '0';
            }
            return s;
        },
        initUploader() {
            let plupload = window.plupload;
            let Qiniu = global.Qiniu;
            let _this = this;
            // domain为七牛空间对应的域名，选择某个空间后，可通过 空间设置->基本设置->域名设置 查看获取
            let isIphone5 = this.phoneType === '5';
            // uploader为一个plupload对象，继承了所有plupload的方法
            this.uploader = Qiniu.uploader({
                runtimes: 'html5,flash,html4', // 上传模式，依次退化
                browse_button: 'pickfiles', // 上传选择的点选按钮，必需
                uptoken: this.token, // uptoken是上传凭证，由其他程序生成
                uptoken_url: '', // Ajax请求uptoken的Url，强烈建议设置（接口地址服务端提供，需要修改，必填）
                uptoken_func: function () {
                    // 在需要获取uptoken时，该方法会被调用
                    // do something(一般是发送手动发送ajax获取到token，如果后端返回格式不跟官方一致又不想改动源代码可以通过这个方式调整)
                    return uptoken;
                },
                get_new_uptoken: true, // 设置上传文件的时候是否每次都重新获取新的uptoken
                bucket_name: '', // （需要修改，必填）
                unique_names: false, // 默认false，key为文件名。若开启该选项，JS-SDK会为每个文件自动生成key（文件名）
                save_key: false, // 默认false。若在服务端生成uptoken的上传策略中指定了sava_key，则开启，SDK在前端将不对key进行任何处理
                domain: 'http://qiniu-plupload.qiniudn.com', // bucket domain eg:http://qiniu-plupload.qiniudn.com/
                container: 'video_container', // 上传区域DOM ID，默认是browser_button的父元素
                max_file_size: '300mb', // 最大文件体积限制
                dragdrop: false, // 开启可拖曳上传
                drop_element: 'video_container', // 拖曳上传区域元素的ID，拖曳文件或文件夹后可触发上传
                chunk_size: '4mb', // 分块上传时，每块的体积
                max_retries: 3, // 上传失败最大重试次数
                auto_start: true, // 选择文件后自动上传，若关闭需要自己绑定事件触发上传
                multi_selection: !isIphone5,
                init: {
                    Key: function (up, files) {
                        _this.fileSize = 0;
                        _this.fileLoaded = 0;
                        _this.fileSpeed = 0;
                        _this.filePercent = 0;
                        return _this.filename;
                    },
                    FilesAdded: function (up, files) {
                        plupload.each(files, function (file) {
                            // 文件添加进队列后，处理相关的事情
                            console.log('FilesAdded');
                            _this.fileSize = _this.toDecimal(file.size);
                        });
                    },
                    BeforeUpload: function (up, file) {
                        console.log('BeforeUpload');
                        if (file.type !== 'video/mp4') {
                            _this.$message.error('上传视频格式只能是 mp4 格式!');
                            return false;
                        }
                    },
                    ChunkUploaded: function (up, file, info) {
                        console.log('ChunkUploaded');
                    },
                    // 获取上传文件的进度
                    UploadProgress: function (up, file) {
                        // 每个文件上传时，处理相关的事情
                        _this.filePercent = parseInt(file.percent);
                        _this.fileLoaded = plupload.formatSize(file.loaded).toUpperCase();
                        _this.fileSpeed = plupload.formatSize(file.speed).toUpperCase();
                    },
                    FileUploaded: function (up, file, info) {
                        console.log('FileUploaded');
                        let response = JSON.parse(info.response);
                        // console.log(response);
                        _this.hash = response.hash;
                        _this.resFileName = response.key;
                        _this.videoUrl = 'http://qiniu-plupload.qiniudn.com/' + response.key; // 需要修改，必填
                        _this.coverPic = 'http://qiniu-plupload.qiniudn.com/' + response.key + '?vframe/png/offset/7/w/480'; // 需要修改，必填
                    },
                    Error: function (up, err, errTip) {
                        console.log('Error');
                        // 上传出错时，处理相关的事情
                        if (err.message === 'File size error.') {
                            // 当文件大小超过限制 走这里
                            _this.$message.error('上传视频大小不能超过 100MB!');
                            return false;
                        }
                        _this.$message.error('上传视频失败，请关闭页面重新上传!');
                        return false;
                        // 上传出错时，处理相关的事情
                        // _this.$toast('Error =');
                        // _this.$toast(err);
                        // _this.$toast('errTip =');
                        // _this.$toast(errTip);
                    },
                    UploadComplete: function () {
                        console.log('UploadComplete');
                        // 队列文件处理完毕后，处理相关的事情
                        // _this.$toast('UploadComplete');
                    }
                }
            });
        },
        getPhoneType() {
            // 正则,忽略大小写
            var patternPhone = new RegExp('iphone', 'i');
            var patternAndroid = new RegExp('Android', 'i');
            var userAgent = navigator.userAgent.toLowerCase();
            var isAndroid = patternAndroid.test(userAgent);
            var isIphone = patternPhone.test(userAgent);
            var phoneType = 'phoneType';
            if (isAndroid) {
                var zhCnIndex = userAgent.indexOf('-');
                var spaceIndex = userAgent.indexOf('build', zhCnIndex + 4);
                var fullResult = userAgent.substring(zhCnIndex, spaceIndex);
                phoneType = fullResult.split('')[1];
            } else if (isIphone) {
                // 6   w=375    6plus w=414   5s w=320     5 w=320
                var wigth = window.innerWidth || document.documentElement.clientWidth || document.body.clientWidth;
                if (wigth > 400) {
                    phoneType = 'iphone6 plus';
                } else if (wigth > 370) {
                    phoneType = 'iphone6';
                } else if (wigth > 315) {
                    phoneType = '5';
                } else {
                    phoneType = 'iphone 4s';
                }
            } else {
                phoneType = '您的设备太先进了';
            }
            return phoneType;
        }
    },
    mounted() {
        this.phoneType = this.getPhoneType();
        this.initUploader();
    }
};
</script>

<style lang="scss">
.upload {
    .el-progress {
        width: 400px;
        margin: 20px 0;
    }

    .btn-container {
        margin-top: 20px;
    }

    .video-url {
        margin-top: 20px;
    }
}
</style>