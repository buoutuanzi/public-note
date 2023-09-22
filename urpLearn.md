# urpLearn

## 一些Assets

### URP Asset

一些与具体渲染管线无关的设置放在这里，是否开启某些功能或者设置等，多个渲染管线可以共用这些数据与设置

### Universal Renderer

与具体渲染管线相关的设置，如渲染路径，会渲染那些 layer 的object，包含哪些 render pass等

### URP Renderer Feature

可以往 Renderer 中加一些 Render Pass

所以 Renderer Feature 起作用的时间都是某些东西渲染完成或开始渲染前，因为同一时间只能有一个 Render Pass 在运行

### Render Pass

可以在这里定义好接收一个 RT 然后输出一个 RT 这个过程内的所有操作，包括如何获得新的 RT 新的 rt 输出到哪等等，自定义的 Render Pass 需要指定在哪个 InjectPoint 被 Renderer 执行

在这里切换的 Render Target 只会影响这一个 Pass

[InjectPoint 列表](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@14.0/manual/customize/custom-pass-injection-points.html)

## Camera Stack

**以下内容都是 urp 新加的，访问时需要时用 `camera.GetUniversalAdditionalCameraData()` 获取 urp 特制的相机组件**

在 urp 中相机有两种 Render Type 可选，Base 和 Overlay

![image-20230920093420340](https://buoutuanzi-picture.oss-cn-guangzhou.aliyuncs.com/tuchuang/image-20230920093420340.png)

![image-20230920093427110](https://buoutuanzi-picture.oss-cn-guangzhou.aliyuncs.com/tuchuang/image-20230920093427110.png)

+ Base Camera：普通的输出相机，会渲染到 render target 上
+ Overlay Camera：会基于其他相机的输出**覆盖**叠加自己的渲染结果，可以把一或多个 Overlay Camera 的输出叠加到 Base Camera 上。无法单独使用，如果不把 Overlay Camera 叠加到一个 Base Camera 上，则这个 Overlay Camera 不会执行渲染

### 多相机

#### 一个 base 多个 Overlay

简单来说 Camera Stack 用一组相机代替了之前的一个相机，之前对一个相机能做的操作，对一个 Camera Stack 也能做

相机在 Camera Stack 中的 **顺序决定了相机的渲染顺序**，从上到下渲染叠加

![image-20230920095613076](https://buoutuanzi-picture.oss-cn-guangzhou.aliyuncs.com/tuchuang/image-20230920095613076.png)

此时的画面层级是: Base Camera -> Overlay Camera 2 -> Overlay Camera 1

#### 多个 base

多个 Base Camera 或者 多个 Camera Stack 时，由 相机 的 priority 属性决定 渲染顺序，越高则越后渲染，渲染出的画面就会在其他相机上面

##### 分屏

两个base相机渲染到屏幕上，并且设置相机的 Output 属性的 Viewport Rect 

左半屏

![image-20230920101412759](https://buoutuanzi-picture.oss-cn-guangzhou.aliyuncs.com/tuchuang/image-20230920101412759.png)

右半屏

![image-20230920101420511](https://buoutuanzi-picture.oss-cn-guangzhou.aliyuncs.com/tuchuang/image-20230920101420511.png)

+ X，Y：从视口坐标的哪里作为左下角开始渲染

+ W，H：渲染的图占 **整个视口** 的多大。当从 x = 0.5 开始渲染时，此时宽度最大也只能到视口的 0.5，因此把宽度设到 0.5 以上也不会有变化

### 渲染顺序及清除

+ Base Camera：渲染循环开始时清除 color buffer 和 depth buffer
+ Overlay Camera：渲染开始时接收一个 color buffer，不会清除。接收一个 depth buffer，可选要不要清除

1. 把 Camera Stack 分成两类，渲染到 rt 的和 渲染到 屏幕 的
2. 渲染 rt 的先做渲染，按 priority 排序
   1. 裁剪base Camera
   2. 渲染 rt
   3. 裁剪 overlay camera
   4. 渲染覆盖到rt
   5. 对每个 overlay camera 重复 3，4
3. 渲染到 屏幕 的渲染，排序流程等和 rt 相机相同

**即使一个 overlay camera 在同一帧中需要渲染多次， unity也不会复用结果**

**在使用 camera stack 或 多个 base camera 时，unity 没有做优化，有多少个相机画多少次**

## 自定义RenderFeature

RenderFeature 是干嘛的？往 Renderer 中加 Pass 的，因此 RenderFeature 有两个方法

![image-20230920152719669](https://buoutuanzi-picture.oss-cn-guangzhou.aliyuncs.com/tuchuang/image-20230920152719669.png)

但是要加 Pass 我们首先得有 Pass，所以还需要自定义 RenderPass

```c#
class LensFlarePass : ScriptableRenderPass
{
    private Material _mat;
    private Mesh _mesh;

    public LensFlarePass(Material mat, Mesh mesh)
    {
        _mat = mat;
        _mesh = mesh;
    }

    public override void Execute(ScriptableRenderContext context, ref RenderingData renderingData)
    {
        CommandBuffer cmd = CommandBufferPool.Get("LensFlareCommand");
        context.ExecuteCommandBuffer(cmd);
        CommandBufferPool.Release(cmd);
    }
};
```

Execute 方法就是 Renderer 调用的用来实现这个自定义 Pass 要干嘛的

就在 Feature 的 Create 方法中创建Pass 然后再 AddRenderPasses 里把 Pass 加到 Renderer 中

```c#
public override void Create()
{
	_lensFlarePass = new LensFlarePass(settings.mat, settings.mesh);
}

public override void AddRenderPasses(ScriptableRenderer renderer, ref RenderingData renderingData)
{
    if(renderingData.cameraData.cameraType == CameraType.Game)
    	renderer.EnqueuePass(_lensFlarePass);
}
```

