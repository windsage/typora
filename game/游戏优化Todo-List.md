# 游戏优化Todo-List

## 知识学习

###  Powerhal

> [ PowerHAL\_command\_introduction.pdf](https://transsioner.feishu.cn/file/M4lWbos4No0Tv6xaoOYcVdkWn3d)  password: 5029096788  这个主要是参数的介绍
>
> [ understanding powerhal](https://transsioner.feishu.cn/docx/ZquudQdKWo1RL7xU9srcK7Lrn9e)  powerhal 文档

```python
// decrypt 
python D:\powerhal_encrypt.py -d C:\Users\yufeng.liu\powerscntbl.xml powerscntbl2.xml
//after modifying powerscntbl2.xml
// encrypt
python D:\powerhal_encrypt.py -e powerscntbl2.xml powerscntbl.xml 
// push into /vendor/etc/
// ps -ef | grep power
kill -9 $(pifof vendor.mediatek.hardware.mtkpower_applist-service.mediatek) $(pidof vendor.mediatek.hardware.mtkpower-service.mediatek)
```



* power\_app\_cfg.xml

Push into `/data/vendor/powerhal`, this file can be effected only after reboot.

power\_mode.cfg

powercontable.xml

powerscntbl.xml




###  调度

[www.zhihu.com](https://www.zhihu.com/question/432162968/answer/2749849155)





## 方案思路

[C语言获取MTK平台系统资源信息(CPU/GPU/fps/温度等)，保存为表格形式输出\_mtk-cl-adp-fps-CSDN博客](https://blog.csdn.net/wukongmingjing/article/details/81604479)

https://github.com/samarxie/fetch-sys-info 这是上述的实现方案



***



小米内核的实现

https://github.com/TheSanty/kernel\_xiaomi\_xaga/tree/xaga-s-oss/drivers/misc/mediatek/performance/fpsgo\_v3



***



去问下董军军/闫浩场景的识别

[ 游戏场景的细分](https://transsioner.feishu.cn/docx/UzQqd9CyJoySzBxlAWCcNz8Wn7b)



***

[MediaTek 提升了 Android SoC 的动态性能](https://developer.android.com/stories/games/mediatek-adpf?hl=zh-cn)

[MediaTek | MediaTek Adaptive Game Technology](https://www.mediatek.com/tek-talk-blogs/mediatek-and-google-take-mobile-gaming-to-the-next-level)

[Android Performance Tuner (APT)](https://developer.android.com/games/sdk/performance-tuner?hl=zh-cn)

[Android 动态性能框架(ADPF)](https://developer.android.com/games/optimize/adpf?hl=zh-cn)



[提升移动游戏体验:性能和功耗的双重优化策略-CSDN博客](https://blog.csdn.net/feelabclihu/article/details/145941843?spm=1001.2014.3001.5501)

[Linux EAS介绍-CSDN博客](https://blog.csdn.net/feelabclihu/article/details/130437253?spm=1001.2014.3001.5501)

[利用ADPF性能提示优化Android应用体验-CSDN博客](https://blog.csdn.net/feelabclihu/article/details/143247135?spm=1001.2014.3001.5501)

[dm-verity原理剖析-CSDN博客](https://blog.csdn.net/feelabclihu/article/details/120232172#comments_36459790)

##  引用文档

1. [ CPU/GPU/DDR/Thermal常用adb查询](https://transsioner.feishu.cn/docx/doxcn42u7Zy0DcFDmhwfHFwjC6d)

2. [ 游戏性能分析SOP](https://transsioner.feishu.cn/wiki/NEXwwf7mdiqjo7ksQIrca5H8nQh)

3. [ \[Internal\]\[USD\]游戏优化原理与基础.pptx](https://transsioner.feishu.cn/file/LKjQbHNFRoQ1t9xCczmclxz3nlh) &#x20;

4. [ MTK-GameMode介绍](https://transsioner.feishu.cn/docx/MTkwdjFjXoZ60jxIjnpcF3h3n3d)

5. 工具使用看这个
   相关工具使用：
   <https://online.mediatek.com/apps/quickstart/QS00288#QSS03385>
