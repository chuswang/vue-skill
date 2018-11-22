<template>
	<div>
		<div id="fileUpload" class="btn-action" style="height:40px"></div>
	</div>
</template>
<script>
export default {
	props: {
		chapterId: {
			required: false,
		}

	},
	data() {
		return {
			uploader: null,
			fileMd5: '',
			suffix: '',
			fileId: '',
		}
	},
	created() {

	},
	watch: {
		fileMd5(v) {
		}
	},
	mounted() {
		const thiz = this;

		this.initWebUpload();
	},
	methods: {
		initWebUpload() {
			let thiz = this;
			WebUploader.Uploader.register({
				// "before-send-file": "beforeSendFile",
				"before-send": "beforeSend",
				// "after-send-file": "afterSendFile",
			}, {
				beforeSendFile: function(file) {},
				beforeSend: function(block, data) {
					// data.fileMd5 = thiz.fileMd5;
					// data.suffix = thiz.suffix;
					// data.chapterId = thiz.chapterId;
					var deferred = WebUploader.Deferred();
					$.ajax({
						type: 'POST',
						url: `${thiz.$store.state.apiConfig.baseUrl+thiz.$store.state.apiConfig.about.videoUploadSuss}?action=checkChunk`,
						data: {
							fileMd5: thiz.fileMd5,
							chunk: block.chunk,
							chunkSize: block.end - block.start,
						},
						success: function(res) {
							let data = JSON.parse(res);
							if (data.ifExist) {
								deferred.reject();
							} else {
								deferred.resolve();
							}
						}
					})
					return deferred.promise();
				},
				afterSendFile: function(file) {
					// thiz.uploadSuccess(file);
				}
			})
			this.uploader = WebUploader.create({
				swf: '../../../static/Uploader.swf',
				server: `${this.$store.state.apiConfig.baseUrl+this.$store.state.apiConfig.about.videoUpload}`,
				pick: {
					id: '#fileUpload',
					label: this.$t('message.EL_upload')
				},
				disableGlobalDnd: true,
				auto: false,
				chunked: true,
				chunkSize: 5 * 1024 * 1024,
				fileSingleSizeLimit: 1024 * 1024 * 1024,
				threads: 1,
				headers: {
					"token": sessionStorage.getItem('token')
				},
				formData: {},
				accept: {
					title: 'Video',
					extensions: 'flv,mp4',
					mimeTypes: 'demo/video/*'
				},
			});
			this.uploader.on('fileQueued', (file) => {
				this.fileQueued(file);
			});
			this.uploader.on('uploadBeforeSend', function(block, data) {
				data.fileMd5 = thiz.fileMd5;
				data.suffix = thiz.suffix;
				data.chapterId = thiz.chapterId;
			});
			this.uploader.on('uploadProgress', (file, progress) => {
				this.$emit('uploadProgress', progress)
			})
			this.uploader.on('uploadSuccess', (file) => {
				this.uploadSuccess(file);

			})
			this.uploader.on('uploadError', (file, reason) => {
				this.$emit('uploadError');
			})
			this.uploader.on('error', (type) => {
				if (type === 'Q_TYPE_DENIED') {
					this.$layer.msg(this.$t('message.EL_upload_error'))
				}
			})
		},
		uploadSuccess(file) {
			let thiz = this;
			// if(!this.fileMd5||!this.suffix||!this.chapterId||!file.name) {
			// 	setTimeout(()=>{
			// 		this.up
			// 	})
			// }
			// console.log(this.fileMd5,this.suffix,this.chapterId);
			// return;
			$.ajax({
				type: 'POST',
				url: `${thiz.$store.state.apiConfig.baseUrl+thiz.$store.state.apiConfig.about.videoUploadSuss}?action=mergeChunks`,
				data: {
					fileMd5: thiz.fileMd5,
					suffix: thiz.suffix,
					chapterId: thiz.chapterId,
					oldName:file.name
				},
				success: function(res) {
					thiz.uploader.reset();
					thiz.$emit('uploadImgSuccess', res)
				},
				error: function(err) {
					thiz.$emit('mergeError');
					thiz.$layer.msg(thiz.$t('message.EL_upload_fail'))
				}
			})
		},
		retry() {
			this.uploader.retry(this.uploader.getFile(this.fileId));
		},
		reset() {
			this.uploader.reset();
		},
		fileQueued(file) {
			this.fileId = file.id;
			this.$emit('changeFile', file.name);
			this.suffix = '.' + file.ext;
			let thiz = this;
			this.uploader.md5File(file, 0, 5 * 1024 * 1024)


				// 及时显示进度
				.progress(function(percentage) {
					console.log('Percentage:', percentage);
				})
				// 完成
				.then(function(val) {
					thiz.fileMd5 = val;

				});
		},
		submit() {
			this.uploader.upload();
		}
	}
}

</script>
<style scoped lang="less">
#fileUpload input[type="file"] {
	display: none;
}

</style>
