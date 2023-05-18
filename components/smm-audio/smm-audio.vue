<template>
	<view class="audioPlay">
		<view class="audioBox">
			<!-- video标签 -->
			<video id="videoPlayer" ref="videoPlayer" :src="audioSrc" autoplay="true" :muted="false"
				style="width: 10rpx;height:10rpx;opacity: .1;" class="videoPlay" @loadedmetadata="readyStateFn"
				@play="playFn" @pause='pauseFn' @timeupdate="timeupdateFn" @error="errorFn" @ended='endedFn'></video>

			<view class="imgBox" @click="playSwitch">
				<view :class="{stopBox:true,open:over}">
					<view class="stop"></view>
				</view>
				<image :src="audioImgSrc" mode="widthFix"></image>
			</view>
			<view class="audioControlBox">
				<view class="songNameBox" @click.stop>
					<text class="songName">{{audioName}}</text>
					<text class="songsinger">{{audioSinger}}</text>
				</view>
				<slider class="audio-slider" block-size="12" :max="fullTime" :value="nowTime" @change="sliderChange"
					@changing="changingFn">
				</slider>
				<view class="songInfoBox" @click.stop>
					<view class="songTime">
						{{nowTime | timeFormat}} / {{fullTime | timeFormat}}
					</view>
					<view class="playItem" v-if="srcList.length>1">
						<view class="palyUp" @click="playUp">
							上一首
						</view>
						<view class="palydown" @click="playDown">
							下一首
						</view>
					</view>
					<view class="addControl">
						<view class="add" @click="openMore=!openMore">
							{{openMore?'-':'+'}}
						</view>
					</view>
				</view>
			</view>
		</view>
		<view :class="{dlcBox:true,see:openMore}">
			<view class="speed" @click="changeSpeed">
				播放速度: x{{speed}}
			</view>
			<view class="loopPlayback" @click="loopPlayback=!loopPlayback">
				循环播放: {{loopPlayback?'开':'关'}}
			</view>
			<view class="lyric" @click="clearAllReadListIndex">
				清除阅读历史
			</view>
		</view>
		<view class="songListBox" v-if="srcList.length && directory" :style="{height:olddirHight+'px'}">
			<scroll-view scroll-y="true" class="songListitem">
				<view :class="{songListItem:true,clickOn:nowAudioArrIndex==index,readOver:item.readOver}"
					v-for="(item,index) in srcList" :key="index" @click="jumpSongPlay(index)">
					{{item.name}}
				</view>
			</scroll-view>
		</view>
	</view>
</template>

<script>
	/*
	
	 【参数接收】
	  src            音频的播放链接
	  srcList        多个音频的播放信息 (会显示上一个下一个) [{src:'音频播放地址',name:'音频名字'...}]
	  name           音频的名字
	  singer         音频的作者名 | 音频的章节名
	  imgSrc         音频对应的图片
	  directory      是否显示歌曲目录列表
	  dirHight       歌曲目录列表展示高度
	  lastSongIndex  指定播放传入音频列表中下标的音频
	  
	  
     【事件向外提供】
	  readyState          音频部署完成事件 - 可获取相关数据
	  play                音频开始播放事件
	  pause               音频暂停播放事件
	  timeupdate          音频持续播放事件
	  error               音频加载失败事件
	  ended               音频播放完成事件
	  playChangeIndex     音频列表播放下标发生改变
	  
	  【后台音频播放器切换】
	   uni.getBackgroundAudioManager()
	   相关文档 https://uniapp.dcloud.net.cn/api/media/background-audio-manager.html#getbackgroundaudiomanager
	  
	*/

	let that = null;

	export default {
		name: "smm-audio",
		data() {
			return {
				audioSrc: this.src,
				audioName: this.name,
				audioSinger: this.singer,
				audioImgSrc: this.imgSrc,
				videoCtx: null,
				// 判断是否正在播放
				over: false,
				// 总时长
				fullTime: 0,
				// 当前时长
				nowTime: 0,
				speed: 1,
				nowAudioArrIndex: 0,
				timer: null,
				// 后台音频控制器
				backAudio: null,
				//背景音乐是否启用
				bgAudioIsPlay: false,
				// 工具台切换
				openMore: false,
				olddirHight: this.dirHight,
				loopPlayback: true
			};
		},
		watch: {
			// 为 app 独立写的脚本
			// #ifdef APP-PLUS
			audioSrc() {
				this.fullTime = 0;
			},
			// #endif
			// 开关工具栏 减少歌曲列表高度
			openMore(val) {
				if (val) {
					this.olddirHight = this.dirHight - 40.8;
				} else {
					this.olddirHight = this.dirHight;
				}
			},
			nowAudioArrIndex(val) {
				console.log(val);
				this._putArrToAudioIndex(val);
				this.$emit('playChangeIndex', this.srcList[val]);
			}
		},
		// 界面传值
		props: {
			src: {
				type: String,
				default: ''
			},
			srcList: {
				type: Array,
				default: () => []
			},
			name: {
				type: String,
				default: '未知'
			},
			singer: {
				type: String,
				default: '未知'
			},
			imgSrc: {
				type: String,
				default: ''
			},
			directory: {
				type: Boolean,
				default: false
			},
			dirHight: {
				type: Number,
				default: 450
			},
			lastSongIndex: {
				type: Number,
				default: 0
			},

		},
		created() {
			// 获取 video 对象
			this.videoCtx = uni.createVideoContext("videoPlayer", this);

			// #ifndef H5
			// 创建 背景音频 对象
			this.backAudio = uni.getBackgroundAudioManager();
			// #endif

			that = this;

			// 循环等待音频数据传入
			this.timer = setInterval(() => {

				if (this.srcList.length) {
					// 为播放器内容区域 进行赋值
					this._putArrToAudioIndex(this.nowAudioArrIndex);
					// 记录之前内容
					this.nowAudioArrIndex = this.lastSongIndex || 0;

					// 查找历史记录
					this.findReadoffIndex();
					clearInterval(this.timer);
				}

				if (this.audioSrc !== '') {
					clearInterval(this.timer);
				}

			}, 500);
		},
		methods: {
			// 点击图片 事件处理函数 - 控制音频开关
			playSwitch() {
				this.over = !this.over;
				// 如果 video 对象存在
				if (this.videoCtx) {
					// 由 over 状态来决定 音频播放开关
					this.over ? this.videoCtx.pause() : this.videoCtx.play();
				}
			},
			// 音频加载成功 事件处理函数 - 获取总时长
			readyStateFn(e) {
				// 播放的总时长
				this.fullTime = Math.floor(e.detail.duration);
				this.$emit('readyState');
			},
			loadedmetadata() {
				uni.showToast({
					title: '加载完成！'
				})
			},
			// 音频播放事件处理函数 (只触发一次) - 提示
			playFn(e) {
				uni.showToast({
					title: "开始播放",
					icon: "none"
				});

				this.over = false;
				this.$emit('play');
			},
			// 音频暂停事件处理函数 (只触发一次) - 提示
			pauseFn() {
				uni.showToast({
					title: "暂停播放",
					icon: "none"
				});
				this.over = true;
				this.$emit('pause');
			},
			// 音频播放事件处理函数 (持续触发) - 获取当前播放时长
			timeupdateFn(e) {
				// 为 app 独立写的脚本
				// #ifdef APP-PLUS
				if (!this.fullTime) {
					this.fullTime = e.detail.duration - 0;
				}
				// #endif

				this.nowTime = Math.floor(e.detail.currentTime);
				this.$emit('timeupdate', e);
			},
			// 用户完成移动滑块事件 - 修改当前播放进度
			sliderChange(e) {
				this.videoCtx.play();
				this.videoCtx.seek(e.detail.value);
			},
			// 用户进行移动滑块事件 - 暂停播放 
			changingFn(e) {
				this.nowTime = e.detail.value;
				this.videoCtx.pause();
			},
			// 改变音频播放速度
			changeSpeed() {
				this.speed = this.speed < 2 ? Math.floor(this.speed + 1) : 0.5;
				this.videoCtx.playbackRate(this.speed);
			},
			// 音频加载失败事件
			errorFn() {
				this.$emit('error');
			},
			// 音频播放结束事件
			endedFn() {
				if (this.srcList.length > 1) {

					console.log('播放结束');

					// 存储在本地该条记录
					const songData = JSON.parse(uni.getStorageSync('readOff') || '[]');

					// 如果用户点击的歌曲项目 已存在历史记录中
					if (!this.overReadIndexList.includes(this.nowAudioArrIndex)) {
						songData.push(this.srcList[this.nowAudioArrIndex]);
						console.log(songData);
						uni.setStorageSync('readOff', JSON.stringify(songData));

						this.findReadoffIndex();
					}

					if (this.loopPlayback) {
						// 下一首
						this.playDown();
					}
				}
				this.$emit('ended');
			},
			// 改变播放列表的下标
			_putArrToAudioIndex(index) {
				this.clearReadListIndex(index);

				this.audioSrc = this.srcList[index].src ? this.srcList[index].src : '';
				this.audioName = this.srcList[index].name ? this.srcList[index].name : '未知';
				this.audioSinger = this.srcList[index].singer ? this.srcList[index].singer : '未知';
				this.audioImgSrc = this.srcList[index].imgSrc ? this.srcList[index].imgSrc :
					'http://smmcat.cn/wp-content/uploads/2023/05/defultImg.png';
				this.findReadoffIndex(index);
			},
			// 上一首
			playUp() {
				this.nowAudioArrIndex != 0 ? this.nowAudioArrIndex = this.nowAudioArrIndex - 1 : this.nowAudioArrIndex =
					this.srcList.length - 1;
				this._putArrToAudioIndex(this.nowAudioArrIndex);
			},
			// 下一首
			playDown() {
				if (this.srcList.length) {
					this.nowAudioArrIndex < this.srcList.length - 1 ? this.nowAudioArrIndex = this.nowAudioArrIndex + 1 :
						this.nowAudioArrIndex = 0;
					this._putArrToAudioIndex(this.nowAudioArrIndex);
				}
			},
			// 歌曲列表跳转指定章节
			jumpSongPlay(index) {
				this.nowAudioArrIndex = index;
				// this._putArrToAudioIndex(index);
			},
			// 为背景音频器 传入参数
			_bgAudioSetData() {
				that.backAudio.title = that.audioName;
				that.backAudio.singer = that.audioSinger;
				that.backAudio.coverImgUrl = that.audioImgSrc;
				that.backAudio.src = that.audioSrc;
			},
			// 清除所有本地历史记录
			clearAllReadListIndex() {
				uni.setStorageSync('readOff', '[]');
				this.findReadoffIndex();
			},
			// 当重新点击了已听完内容 清除本地存储该条记录
			clearReadListIndex(i) {

				// 获取本地数据
				const oldList = JSON.parse(uni.getStorageSync('readOff') || '[]');
				const index = oldList.findIndex(e => e.src === this.srcList[i].src);

				// 如果查到历史记录中有对应数据
				if (index !== -1) {
					oldList.splice(index, 1);
					uni.setStorageSync('readOff', JSON.stringify(oldList));
					this.findReadoffIndex();
				}
			},
			// 查找历史记录 返回下标数组 并重新渲染界面
			findReadoffIndex() {
				// 初始化历史记录
				this.overReadIndexList = [];

				// 获取本地数据
				const oldList = JSON.parse(uni.getStorageSync('readOff') || '[]');
				const len = oldList.length;

				// 查找下标
				for (let i = 0; i < len; i++) {
					this.overReadIndexList.push(this.srcList.findIndex(e => e.src === oldList[i].src));
				}
				this.buildReadListStatus();
			},
			// 创建历史记录对象 并改变界面对应项 样式显示
			buildReadListStatus() {
				const len = this.overReadIndexList.length;
				this.srcList.forEach(e => e.readOver = false);
				for (let i = 0; i < len; i++) {
					this.srcList[this.overReadIndexList[i]].readOver = true;
				}
			},
			openInfomation() {
				uni.showModal({
					title: '联系我们',
					content: '-',
					confirmText: "确定", // 确认按钮文字 
					confirmColor: '#F54E40', //确认字体的颜色
					cancelColor: '#000', //取消字体的颜色
					success: function(res) {
						if (res.confirm) {
							return
						} else if (res.cancel) {
							console.log('用户点击取消');
							return
						}
					}
				});
			}
		},
		filters: {
			timeFormat(value) {
				var result = typeof value === "string" ? parseFloat(value) : value;
				if (isNaN(result))
					return "";
				let h = Math.floor(result / 3600) < 10 ?
					"0" + Math.floor(result / 3600) :
					Math.floor(result / 3600);
				let m = Math.floor((result / 60) % 60) < 10 ?
					"0" + Math.floor((result / 60) % 60) :
					Math.floor((result / 60) % 60) + h * 60;
				let s = Math.floor(result % 60) < 10 ?
					"0" + Math.floor(result % 60) :
					Math.floor(result % 60);

				return h == '00' ? `${m}:${s}` : `${h}:${m}:${s}`;
			}
		},
		mounted() {
			uni.$on('onShow', function(data) {

				// #ifndef H5
				// 判断后台是否开启背景音乐
				if (that.bgAudioIsPlay) {
					that.backAudio.stop();
					that.videoCtx.seek(that.backAudio.currentTime);
					that.bgAudioIsPlay = false;
				}

				// 延时开启
				setTimeout(() => {
						that.videoCtx.play();
					},
					200);

				// #endif
			});
			uni.$on('onHide', function(data) {

				// #ifndef H5
				// 判断前台是否开启音乐
				if (!that.over) {
					that.bgAudioIsPlay = true;

					// 启动背景音频器 传入参数
					that._bgAudioSetData();
					//跳转指定播放位置
					that.backAudio.seek(that.nowTime + 0.5);
					// 当该内容结束后 播放下一首
					that.backAudio.onEnded(() => {
						that.playDown();
						that.backAudio = uni.getBackgroundAudioManager();
						that._bgAudioSetData();
					});
					that.backAudio.onTimeUpdate(() => {

						if (!that.over) {
							that.videoCtx.pause();
						}
					});
				}
				// #endif
			});
		},
		beforeDestroy() {
			// 清除事件监听
			uni.$off('onHide');
			uni.$off('onShow');
		}
	}
</script>

<style lang="scss">
	.audioPlay {
		display: flex;
		flex-direction: column;

		.audioBox {
			position: relative;
			display: flex;
			width: 100%;
			height: 200rpx;
			background-color: #ccc;

			.videoPlay {
				position: absolute;
				top: 0;
				right: 0;
			}

			.imgBox {
				position: relative;
				width: 200rpx;
				height: 200rpx;
				background-color: #66ccff;
				overflow: hidden;

				image {
					width: 100%;
				}

				.stopBox {
					position: absolute;
					display: none;
					width: 100%;
					height: 100%;
					z-index: 9;
					background-color: rgba(10, 10, 10, .4);

					&.open {
						display: block;
					}

					.stop {
						position: absolute;
						top: 50%;
						left: 44%;
						transform: translate(0, -50%);
						width: 0;
						height: 0;
						border: 40rpx solid transparent;
						border-left-color: #ccc;
					}
				}

			}

			.audioControlBox {
				flex: 1;
				display: flex;
				flex-direction: column;
				justify-content: space-between;

				.songNameBox {
					margin: 5px auto;
					width: 500rpx;
					height: 150rpx;
					overflow: hidden;
					display: flex;
					flex-direction: row;
					justify-content: center;
					align-items: center;

					.songName {
						font-size: 30rpx;
						flex: 5;
						font-weight: 800;
						overflow: hidden;
						// background-color: #66ccff;
						text-overflow: ellipsis;
						white-space: nowrap;
					}

					.songsinger {
						flex: 2;
						white-space: nowrap;
						background-color: orange;
						padding: 10rpx;
						display: flex;
						justify-content: center;
						align-items: center;
						border-radius: 4rpx;
						font-size: 20rpx;
						margin: 5rpx;
					}
				}

				.audio-slider {
					margin-top: 5rpx;
				}

				.songInfoBox {
					display: flex;
					padding: 0px 25rpx;
					justify-content: space-between;
					align-items: center;
					height: 200rpx;
					background-color: #4e4e4e;

					.songTime {
						font-size: 12px;
						display: inline-block;
						padding: 8rpx 20rpx;
						background-color: #66cc;
						border-radius: 10rpx;
						white-space: nowrap;
						color: #fff;
					}
				}

				.playItem {
					width: 220rpx;
					height: 50rpx;
					display: flex;
					font-size: 12px;
					justify-content: space-around;
					align-items: center;
					background-color: #4e4e4e;

					view {
						background-color: darkorange;
						padding: 8rpx 15rpx;
						color: #fff;
						border-radius: 5rpx;
					}
				}

				.addControl {
					display: flex;
					width: 60rpx;
					height: 50rpx;
					background-color: green;
					font-size: 12px;
					justify-content: center;
					align-items: center;
					color: #fff;

					.add {
						font-size: 40rpx;
						font-weight: 800;
						padding-bottom: 5rpx;
					}
				}

			}
		}

		.songListBox {
			border-bottom: 3px solid #ccc;
			width: 100%;
			// height: 450rpx;
			background-color: #ccc;

			.songListitem {
				width: 100%;
				height: 100%;

				.songListItem {
					position: relative;
					padding: 20rpx 40rpx;
					background-color: #fff;
					box-shadow: 1px 0px 2px #4e4e4e;
					border: 1px solid #ccc;
					margin: 8rpx 0;
					font-size: 15px;
					color: #4e4e4e;
					overflow: hidden;
					text-overflow: ellipsis;
					white-space: nowrap;

					&.clickOn {
						color: orangered !important;
					}

					&.readOver {
						color: #ccc;
					}

					&.readOver::after {
						position: absolute;
						right: 25rpx;
						font-style: oblique;
						content: '已听完';
					}
				}
			}
		}

		.dlcBox {
			width: 100%;
			height: 0rpx;
			transition: all .3s;
			border-bottom: 0px solid #000;
			background-color: #ccc;
			overflow: hidden;

			&.see {
				height: 70rpx;
				display: flex;
				justify-content: left;
				align-items: center;
				border-bottom: 5px solid #66cc;
			}

			.speed {
				display: inline-block;
				padding: 10rpx 20rpx;
				margin: 0 5px;
				color: #fff;
				background-color: orange;
			}

			.lyric {
				display: inline-block;
				padding: 10rpx 20rpx;
				margin: 0 5px;
				color: #fff;
				background-color: royalblue;
			}

			.loopPlayback {
				display: inline-block;
				padding: 10rpx 20rpx;
				margin: 0 5px;
				color: #fff;
				background-color: deepskyblue;
			}
		}
	}
</style>