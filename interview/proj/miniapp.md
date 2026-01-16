## 小程序分包
分包预下载
分包异步化：降低包体积，对代码（JS/CSS）及其依赖资源的按需加载

## 声音管理
- 音乐与音效管理：对 InnerAudioContext 的封装，实现音乐与音效的控制管理
    - InnerAudioContext 可跨页面播放
    - 背景采用一个实例
    - 音效采用音效池
- 音效池：允许多个音效的同时存在，并在播放后统一回收
    - 对某一音效预先创建多个 InnerAudioContext 放入缓存
    - 播放该音效时从缓存中获取进行播放，不够时再进行创建
    - 音效播放完毕再压入缓存，缓存超出则销毁

## 骨骼动画兼容
背景：活动小程序需展示游戏内的人物动画
最初使用 spine-canvas，绘制后有白线问题，后改为 spine-webgl，使之能在小程序环境运行
网络请求兼容：new XMLHttpRequest -> wx.Request
canvas 兼容：new Image()->canvas.createImage()
...

## 分享图绘制
基于 canvas2d 与 wx.canvasToTempFilePath
要考虑到 dpr，根据物理像素对图片进行采用，若图片分辨率不足会产生模糊问题

## 登录
业务侧登录 + 微信侧登录
微信侧登录：wx.login，返回唯一标识 openId，用于微信号和业务号的绑定
    
## 一次性消息订阅
微信侧订阅：wx.requestSubscribeMessage
订阅成功后，将消息模板id + openId 发给服务端进行存储
满足通知条件时调用通知接口进行通知

## 小程序码打开小程序
活动页面调用服务端接口生成小程序码
扫码进入小程序，根据场景值判断登录来源