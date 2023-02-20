# 1. 图片加载框架
## 1.1. fresco/glide
https://juejin.cn/post/6844903986412126216  
三级缓存机制
## 1.2. coil 
轻量级协程图片库
# 2. [图片格式](https://juejin.cn/post/6912217043009798157)
jpg vs png vs webp vs heic  
gif vs webp  
svg  
jpg或者png格式改为webp，webp改为heic带来的带宽提升。
# 3. 图片监控  
## 3.1. 大图监控  

可以参考这个文章的5.4 https://juejin.cn/post/7074762489736478757
### 3.1.1. 图片框架侧的监控
### 3.1.2. ImageView的监控
## 3.2. 空窗率监控（相对、绝对）  


# 4. 图片本地压缩  
无侵入式图片压缩：  
参见这篇文章的5.3.2：https://juejin.cn/post/7074762489736478757  
参见这个开源库：https://github.com/smallSohoSolo/McImage  

# 5. 图片渲染优化
比如某些场景大图无法支持小的，但是渲染的区域又远小于图片实际大小，可以做下采样处理，提高性能。