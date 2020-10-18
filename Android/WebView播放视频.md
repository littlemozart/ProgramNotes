# WebView播放视频的方式和坑点记录

## 方式一：采用 `video` 标签自带能力

**场景**： 直接播放网页视频，无其它复杂交互（如倍速，手势，下载等）

**难点**： 适配全屏播放

**方案**： 实现 `WebChromeClient` 的 `onShowCustomView` 和 `onHideCustomView` 方法，把网页视频对应的 View 放到 `DecorView` 或通过 `WindowManager.addView()` 添加到 window 层。 

**坑点**： 在测试过程中，发现一些视频直接切换到横屏全屏模式后，会自动切回竖屏非全屏状态。如下图所示：

![image](https://github.com/littlemozart/ProgramNotes/blob/master/assets/sample_1.gif)

**定位**： 发现出现上述现象的视频都是高比宽大的，即竖直方向的视频。看了谷歌 chrome 浏览的表现，发现它会根据视频的宽高比选择全屏后手机横竖屏的模式。所以得出结论是，不能直接切换到横屏全屏模式，
应根据视频的宽高判断，如果源视频的宽度大于高度，则可横屏全屏，否则须竖屏横屏。

**思路**： 在收到 `onShowCustomView` 回调时先竖屏全屏，然后通过调用 js 方法获取原视频的宽高，在判断是否要切到横屏模式。下面是 js 获取视频宽高的方法：

```
function getVideoRect() {
    let tags = document.getElementsByTagName('video');
    if (tags.length > 0) {
        let video = tags[0];
        return {
            'width': video.videoWidth,
            'height': video.videoHeight
        };
    }
    return {};
}
```

**实现**： 

参考 [web-video](https://github.com/littlemozart/web-video) 中的 `WebVideoActivity`

## 方式二：用原生视频代替 WebView 中的视频

**场景**： 知道原视频地址和格式的情况下；有复杂的交互，对性能要求交高。

**背景**： 通常用在 hybrid 应用中，网页部分不直接用 `video` 标签，而是采用一个占位图显示。当点击播放按钮时通过调用原生的方法，由原生视频来实现播放功能。

**难点**： 确定原生视频在 WebView 中的布局位置和大小。

**思路**： 因为 WebView 是继承绝对布局，也是个 `ViewGroup` 。所以可以通过调用 js 方法获取 WebView 中的视频占位图的位置和大小（x, y, width, height），
然后用这些参数构造出一个 VideoView 添加到 WebView 中。

```
// 在知道占位组件 id 的情况下获取组件位置和宽高
function getElementRect(id) {
    let el = window.document.getElementById(id);
    let rect = el.getBoundingClientRect();
    return {'top': rect.top, 'left': rect.left, 'width': rect.width, 'height': rect.height};
}
```

**实现**

参考 [web-video](https://github.com/littlemozart/web-video) 中的 `ExoVideoActivity`
