## 计划更新的内容

- [x] 歌曲列表播放
- [x] 歌曲倍速播放
- [x] 歌曲后台播放
- [x] 歌曲播放完成历史记录
- [ ] 歌词列表同步播放

## 组件介绍

#### 基于 uni.createVideoContext() 创建的 video 实例

解决了网络请求得到的音频地址无法播放的问题，索性发布了；

思路是 使用了等待机制，当有数据后，才进行赋值操作；
所以 porp 不再对 video 传入空值

如若是异步得到的数据，不建议使用 ``:src、:name...`` 对播放器传值，
推荐任何情况下都使用 ``:srcList`` 进行对象数组传值；即使只是播放单个音频

## 参数传递

|参数 |说明|
|:-|:-:|
| src|  音频的播放链接                          |
 | srcList   |  多个音频的播放信息 (会显示上一个下一个) [{src:'音频播放地址',name:'音频名字'...}]   |
| name    |  音频的名字                                              |
| singer    |  音频的作者名 & 音频的章节名                   |
| imgSrc    |  音频对应的图片             |
|directory  |  是否显示歌曲目录列表 |
|dirHight   |  歌曲目录列表展示高度 |

## 事件向外提供

|事件名 |说明|
|:-|:-:|
	|  readyState   | 音频部署完成事件 - 可获取相关数据      |
	|  play         | 音频开始播放事件                      |
	|  pause        | 音频暂停播放事件                      |
	|  timeupdate   | 音频持续播放事件                      |
	|  error        | 音频加载失败事件                      |
	|  ended        | 音频播放完成事件                      |
	
	
## 使用演示

### 传入多个

```
<template>
    <view>
        <smm-audio :srcList='arrList'></smm-audio>
    </view>
</template>

<script>
export default {
    data() {
        return {
            arr: [{
                src: '音频真实地址',
                name: '音频名',
                singer: '作者名 & 系列名',
                imgSrc: '图片链接'
            },
            {
                src: '音频真实地址',
                name: '音频名',
                singer: '作者名 & 系列名',
                imgSrc: '图片链接'
            },
            {
                src: '音频真实地址',
                name: '音频名',
                singer: '作者名 & 系列名',
                imgSrc: '图片链接'
            }],
        };
    }
}
</script>
```

### 传入单个
```
<template>
    <view>
        <smm-audio 
        :src='songSrc' 
        :name="ongName"
        :singer="songSinger"
        :imgSrc="songImgSrc"
        ></smm-audio>
    </view>
</template>

<script>
export default {
    data() {
        return {
            songSrc: '音频真实地址',
            songName: '音频名',
            songSinger: '作者名 & 系列名',
            songImgSrc: '图片链接'
        };
    }
}
</script>
```
#### 后台播放

组件会监听页面切换后台后自动调用 ``getBackgroundAudioManager`` 继续播放音频

但由于组件不能直接监听页面的切换后台事件，需要从页面向组件传入对应事件

```
<template>
	<view>
		<smm-audio  :srcList='arrList'></smm-audio>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				srcList: [{
                  src: '音频真实地址',
                  name: '音频名',
                  singer: '作者名 & 系列名',
                  imgSrc: '图片链接'
                }]
			};
		},
		onShow: function() {
			console.log('App Show')
			uni.$emit('onShow');
		},
		onHide: function() {
			console.log('App Hide')
			uni.$emit('onHide');
		}
	}
</script>
```
#### 实现下次启动自动跳转到上次播放的音频
```
<template>
  <view>
    <smm-audio :srcList='arrList' :lastSongIndex='befoIndex' :directory="true" :dirHight="wh"
      @playChangeIndex='storageIndex'></smm-audio>
  </view>
</template>

<script>
export default {
  data() {
    return {
      arrList: [],
      wh: 0,
      nowTime: 0,
      // 上一个记录
      befoIndex: 0
    };
  },
  onLoad() {
    // 获取设备信息
    const info = uni.getSystemInfoSync();
    // 为当前 可用高度对应变量进行赋值
    this.wh = info.windowHeight - 100;
  },
  created() {
    this.getSongList();
  },
  methods: {
    getSongList() {
      uni.request({
        url: '网络请求接口',
        method: 'GET',
        success: ({
          data: res
        }) => {
          this.arrList = res.map(function (item, index) {
            let src = item.songUrl;
            // let name = item.songName;
            let name = `第 ${index + 1} 课`;
            let singer = '张老师特别课程';
            let imgSrc = 'xxx.jpg';
            // 封装成 播放器支持的对象
            return {
              src,
              name,
              singer,
              imgSrc
            }
          });
          // 获取本地存储的最后播放的音频内容
          const historyItem = JSON.parse(uni.getStorageSync('lastPlay') || '{}');
          // 如若没有 从头开始播放
          if (!historyItem) {
            return this.befoIndex = 0;
          }
          // 若有 对请求得到的数据进行比较 尝试获取下标
          const index = this.arrList.findIndex(e => e.src === historyItem.src);
          this.befoIndex = index === -1 ? 0 : index;
        }
      })
    },
  },
  // 组件中音频播放完成后的事件
  storageIndex(item) {
    // 为本地存储 lastPlay 下存储该条对象
    uni.setStorageSync('lastPlay', JSON.stringify(item));
  }
}
</script>
```
