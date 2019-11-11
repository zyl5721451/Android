### 一、强制黑暗(Force Dark)模式简介
黑暗模式不仅可能减少耗电量，在光线暗的环境中对眼睛也有保护的作用。`AndroidQ`提供了`Force Dark`功能，在浅色主题(例如:`Theme.Material.Light`)上面设置`android:forceDarkAllowed="true"`就可以快速的实现黑色主题背景，而不需要为黑色主题背景提供单独的主题资源。具体使用方法可以参考官方文档：[https://developer.android.google.cn/guide/topics/ui/look-and-feel/darktheme](https://developer.android.google.cn/guide/topics/ui/look-and-feel/darktheme)
### 二、适配过程中遇到的问题
##### 1、图片圆角被加亮
如果图片圆角是通过自定义`ImageView`，在`onDraw()`方法中通过`canvas.drawPath`来实现的。则在使用`ForceDark`功能时绘制的圆角颜色会被加亮。如果圆角颜色是白色，此时圆角在黑色背景下面就会显示为白色，与我们所要的效果刚好相反。
圆角绘制的代码如下所示：
```
   private void initPaint() {
        paint = new Paint();
        paint.setAntiAlias(true);           //设置画笔为无锯齿
        paint.setStyle(Paint.Style.FILL);   //设置实心
        paint.setColor(Color.WHITE);
    }
    public void initRightDownPath() {
        rightDownPath = new Path();
        rightDownPath.moveTo(width, height);
        rightDownPath.lineTo(width, height - radius);
        RectF rectF = new RectF(width - 2 * radius, height - 2 * radius, width, height);
        rightDownPath.arcTo(rectF, 0, 90, true);
        rightDownPath.lineTo(width, height);
        rightDownPath.close();
    }
   /**
      绘制白色圆角在forcedark模式下会被加亮
   **/
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        canvas.drawPath(leftDownPath, paint);
        canvas.drawPath(leftUpPath, paint);
        canvas.drawPath(rightUpPath, paint);
        canvas.drawPath(rightDownPath, paint);
    }

```

##### 2、文字绘制被加亮
如果自定义`ImageView`中，除了绘制圆角外还绘制了文字，则使用 `ForceDark`功能时圆角和文字颜色都会被加亮。

```
   /**
      绘制白色圆角和白色文字在forcedark模式下会被加亮
   **/
@Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        //绘制圆角
        canvas.drawPath(leftDownPath, paint);
        canvas.drawPath(leftUpPath, paint);
        canvas.drawPath(rightUpPath, paint);
        canvas.drawPath(rightDownPath, paint);
        //绘制文字
        mTextPaint.setTextSize(typeIconTextSize);
        Paint.FontMetrics fontMetrics = mTextPaint.getFontMetrics();
        float baseline = getHeight() - typeIconMargin - (typeIconHeight + fontMetrics.bottom + fontMetrics.top) / 2;
        float textWidth = mTextPaint.measureText("长图");
        canvas.drawText("长图", getWidth() - typeIconMargin - typeIconWidth / 2 - textWidth / 2, baseline, mTextPaint);
    }

```
##### 3、RecycleView ItemDecoration被加亮
`RecycleView`的`ItemDecoration`可以用来在`RecycleView`上面绘制一些装饰元素，例如：分割线、圆点标题、字母排序标题等。`ItemDecoration`有两个方法：`onDraw`和`onDrawOver`，这两个方法分别表示在`RecycleView`绘制之前和绘制完成后绘制，字母排序标题就是在`onDrawOver`方法中绘制出来的。当开启`ForceDark`功能时字母以及字母的背景会被处理成加亮，在加暗的`ItemView`上面显得特别突兀。
字母排序的绘制方法如下所示：
```
    /**
     * {@inheritDoc}
     绘制的字母和字母背景forcedark模式下会被加亮
     */
    @Override
    public void onDrawOver(Canvas c, RecyclerView parent, RecyclerView.State state) {
        final int count = parent.getChildCount();
        for (int layoutPos = 0; layoutPos < count; layoutPos++) {
            final View child = parent.getChildAt(layoutPos);
            View header = getHeader(parent, adapterPos).itemView;
            final int left = child.getLeft();
            int top = getHeaderTop(parent, child, header, adapterPos, layoutPos);
            c.save();
            c.translate(left, top);
            header.draw(c);
            c.restore();
        }
    }

```
其中`header`的布局文件如下所示：
```
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <TextView
        android:id="@+id/tv_header"
        style="@style/Text_B4_T7"
        android:layout_width="match_parent"
        android:layout_height="20dp"
        android:background="@color/Blk_8"
        android:gravity="center_vertical"
        android:paddingStart="14dp"
        android:paddingEnd="14dp"
        android:visibility="visible"
        tools:text="A" />
</FrameLayout>

```
需要注意的是显示字母的`TextView`设置了一个背景`android:background="@color/Blk_8"`。
为了解决以上适配问题，需要我们了解一下`View`硬件绘制的相关概念以及`ForceDark`模式的源码
### 三、RenderNode和DisplayList
##### 1、RenderNode介绍
绘制节点`RenderNode`是用来构建`View`的硬件加速绘制结构树的，`java`层`RenderNode`的使用方法如下所示：
```
   public void drawRenderNode(Canvas canvas){
        RenderNode mRenderNode = new RenderNode("name");
        RecordingCanvas rendCanvas = mRenderNode.beginRecording();
        try {
            rendCanvas.drawRect(rect,mPaint);
        }finally {
            mRenderNode.endRecording();
        }
        mRenderNode.setPosition(0,0,100,100);
        if(canvas.isHardwareAccelerated()&&mRenderNode.hasDisplayList()){
            canvas.drawRenderNode(mRenderNode);
        }
    }

```
其中`beginRecording()`方法创建了一个`RecordingCanvas`，用来记录当前`RenderNode`的绘制操作，例如：`drawCircle`、`drawBitmap`、`drawRect`、`drawRenderNode`等。`endRecording()`方法会返回`Native`层中  `RecordingCanvas`中`DisplayList`的地址，并将`DisplayList`保存到`Native`层中的`RenderNode`的`mDisplayList`中。例如：上图所示的`mRenderNode`节点的`DisplayList`中记录了`drawRect`操作。
##### 2、DisplayList介绍
在`AndroidQ`系统源码中，`DisplayList`功能主要是在`SkiaDisplayList`类中实现，`SkiaDisplayList`包含了`RenderNode`集合和当前`View`需要绘制的内容。
```
//位置frameworks/base/libs/hwui/pipeline/skia/SkiaDisplayList.h
  std::deque<RenderNodeDrawable> mChildNodes;//子RenderNode
  DisplayListData mDisplayList;//绘制操作列表
```
`RenderNode`中保存了`DisplayList`数据
```
//位置：frameworks/base/libs/hwui/RenderNode.h
 DisplayList* mDisplayList;
```
##### 3、一个例子
每个`View`至少有一个`RenderNode`。在`View`的`draw(Canvas canvas, ViewGroup parent, long drawingTime) `方法中记录绘制的一系列操作。其中包含了绘制背景时创建的背景`mBackgroundRenderNode`以及子`View`的`RenderNode`(如果包含子`View`的话)。背景`mBackgroundRenderNode`创建时默认`UsageHint`会被设置为`USAGE_BACKGROUND`，`UsageHint`是什么在下面源码的分析中会说明，这里只需要知道，背景节点会被设置为默认加黑。这样如果一个`ViewGroup`设置了背景，并且有一个子`View`，则这个`ViewGroup`除了普通的绘制外还会有两个`drawRenderNode`操作，相应的`DisplayList`中记录了两个`RenderNode`。
如下图所示布局，是一个`FrameLayout`嵌套一个`TextView`。
```
 <?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:background="@color/black"
    android:layout_height="wrap_content">

    <TextView
        android:id="@+id/tv_header"
        style="@style/Text_B4_T7"
        android:layout_width="match_parent"
        android:layout_height="20dp"
        android:background="@color/Blk_8"
        android:gravity="center_vertical"
        android:paddingStart="14dp"
        android:paddingEnd="14dp"
        android:visibility="visible"
        tools:text="A" />
</FrameLayout>
```
![image.png](https://upload-images.jianshu.io/upload_images/3413002-072b38dc4c79a7fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上面这张图展示了上面布局中`RendrNode`和`DisplayList`关系。

### 四、强制黑暗(Force Dark)源码分析
##### 1、黑暗模式的判断
在`Native`层中的`RenderNode`类中实现了黑暗模式的处理，最终是通过`handleFroceDark`方法来实现的，`handlerForceDark`的调用关系是：`RenderNode:prepareTreeImpl-->pushStagingDisplayListChanges-->syncDisplayList-->handleForceDark`。
```
//位置：frameworks/base/libs/hwui/RenderNode.cpp
void RenderNode::prepareTreeImpl(TreeObserver& observer, TreeInfo& info, bool functorsNeedLayer) {
    if (info.mode == TreeInfo::MODE_FULL) {
        pushStagingDisplayListChanges(observer, info);//构造mDisplayList，应用黑暗模式
    }
    if (mDisplayList) {
        info.out.hasFunctors |= mDisplayList->hasFunctor();
        bool isDirty = mDisplayList->prepareListAndChildren(
                observer, info, childFunctorsNeedLayer,
                [](RenderNode* child, TreeObserver& observer, TreeInfo& info,
                   bool functorsNeedLayer) {//递归调用处理子节点
                    child->prepareTreeImpl(observer, info, functorsNeedLayer);
                });
        if (isDirty) {
            damageSelf(info);
        }
    }
}
```
`handleForceDark`方法实现了颜色是加暗还是加亮的判断，判断的方法如下所示：
```
/**

1、位置：frameworks/base/libs/hwui/RenderNode.cpp
2、如果当前RenderNode中有文字需要展示，则是定义为前景，需要颜色变亮
3、如果当前RenderNode的UsageHin是Unknow，且有多于一个子节点，则定义为背景，需要颜色变暗
4、如果当前RenderNode的UsageHin是Unknow，且有一个子节点，并且子节点不是背景，则定义为背景，需要颜色变暗
5、如果有大于一个子节点，遍历子节点，如果当前节点边界包含子节点，则定义为背景，需要颜色变暗
6、最后根据定义的背景，调用applyColorTransform进行颜色变换

**/
void RenderNode::handleForceDark(android::uirenderer::TreeInfo *info) {
    if (CC_LIKELY(!info || info->disableForceDark)) {
        return;
    }
    auto usage = usageHint();
    const auto& children = mDisplayList->mChildNodes;//需要展示的所有节点
    if (mDisplayList->hasText()) {//有文字则是前景
        usage = UsageHint::Foreground;//前景，light
    }
    if (usage == UsageHint::Unknown) {
        if (children.size() > 1) {//有子节点，则是背景
            usage = UsageHint::Background;
        } else if (children.size() == 1 &&//有一个子节点，并且子节点的usageHint不是背景，则是背景
                children.front().getRenderNode()->usageHint() !=
                        UsageHint::Background) {
            usage = UsageHint::Background;
        }
    }
    if (children.size() > 1) {//有子节点
        // Crude overlap check
        SkRect drawn = SkRect::MakeEmpty();
        for (auto iter = children.rbegin(); iter != children.rend(); ++iter) {
            const auto& child = iter->getRenderNode();//遍历到的子节点
            // We use stagingProperties here because we haven't yet sync'd the children
            SkRect bounds = SkRect::MakeXYWH(child->stagingProperties().getX(), child->stagingProperties().getY(),
                    child->stagingProperties().getWidth(), child->stagingProperties().getHeight());//子节点的边界
            if (bounds.contains(drawn)) {//子节点边界包含其它子节点，则是背景
                // This contains everything drawn after it, so make it a background
                child->setUsageHint(UsageHint::Background);
            }
            drawn.join(bounds);
        }
    }
    mDisplayList->mDisplayList.applyColorTransform(//应用颜色变换，如果是背景则把颜色变深，如果是前景则把颜色浅
            usage == UsageHint::Background ? ColorTransform::Dark : ColorTransform::Light);
}
```
##### 2、黑暗模式的执行
在`handleForceDark`方法中的`applyColorTransform`会执行以下调用顺序：`DisplayList:applyColorTransform-->RecordingCanvas:colorTransformForOp-->CanvasTransform:transformPaint`
最终会调用到`CanvasTransform`中的`transfromPaint`方法。`CanvasTransform`的`transformPaint`方法有两个，一个是执行普通绘制颜色变换，一个是执行图片绘制的的颜色变换。
- 普通绘制颜色变换
颜色的变化是通过把颜色转换成Lab颜色模式，然后对颜色的亮度做反转，这样原本暗色颜色会被加亮，原来亮色颜色会被加暗，其它细节可以参考源码。
```
//位置：frameworks/base/libs/hwui/CanvasTransform.cpp
/**
1、Lab颜色模式，包括亮度L和a,b两个颜色通道
2、对颜色的亮度进行反转
**/
static SkColor makeLight(SkColor color) {
    Lab lab = sRGBToLab(color);
    float invertedL = std::min(110 - lab.L, 100.0f);//反转亮度
    if (invertedL > lab.L) {//超过原来的亮度，则以新的为准，比原来还暗，则不进行变换
        lab.L = invertedL;
        return LabToSRGB(lab, SkColorGetA(color));
    } else {
        return color;
    }
}
/**
1、Lab颜色模式，包括亮度L和a,b两个颜色通道
2、对颜色的亮度进行反转
**/
static SkColor makeDark(SkColor color) {
    Lab lab = sRGBToLab(color);
    float invertedL = std::min(110 - lab.L, 100.0f);
    if (invertedL < lab.L) {//没有原来的亮，则以新的为准，比原来还亮，则还用原来的。
        lab.L = invertedL;
        return LabToSRGB(lab, SkColorGetA(color));
    } else {
        return color;
    }
}

```
- 图片绘制颜色变换
图片颜色的变换同样是对图片的明暗度进行反转处理，和普通绘制不同的是，这里的明暗转换是通过Skia图片绘制引擎实现的。具体实现原理可能参考Skia相关资料。
```
/**
1、根据Palette和明暗的判断决定是否要对图片进行处理
2、为图片设置一个colorfilter
**/

bool transformPaint(ColorTransform transform, SkPaint* paint, BitmapPalette palette) {
    palette = filterPalette(paint, palette);//计算图片的palette是明还是暗
    bool shouldInvert = false;
    if (palette == BitmapPalette::Light && transform == ColorTransform::Dark) {//图片是明亮的，效果是要变暗
        shouldInvert = true;
    }
    if (palette == BitmapPalette::Dark && transform == ColorTransform::Light) {//图片是暗的，效果是变亮
        shouldInvert = true;
    }
    if (shouldInvert) {
        SkHighContrastConfig config;
        config.fInvertStyle = SkHighContrastConfig::InvertStyle::kInvertLightness;
        paint->setColorFilter(SkHighContrastFilter::Make(config)->makeComposed(paint->refColorFilter()));//设置一个colorfilter
    }
    return shouldInvert;
}

```
通过以上介绍，我们已经了解了黑暗模式是怎么实现的了，回到第二节中适配中出现的问题，我们可以分析一下问题的解决方法。

### 五、问题的解决
##### 1、图片圆角被加亮
如果`ImageView`没有设置背景，则当前`ImageView`的`DisplayList`中没有子`RenderNode`，根据第三节`Native`中`RenderNode`源码可知`ImageView`的`RenderNode`的`UsageHint`是默认的`Unknow`，因此在`onDraw`中绘制的圆角会被加亮。解决方法是在`onDraw`中增加一个空的`RenderNode`，这样`DisplayList`中就有一个默认`UsageHint`是`Unknow`的`RenderNode`，因此`ImageView`整体会被加暗。
```
   @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        
        drawRoundCornor(canvas);
        
        if (Build.VERSION.SDK_INT > Build.VERSION_CODES.P) {
            if(mRenderNode == null){
                mRenderNode = new RenderNode("");
            }
            mRenderNode.beginRecording(1, 1);
            mRenderNode.endRecording();
            if (canvas.isHardwareAccelerated()&&mRenderNode.hasDisplayList()) {
                canvas.drawRenderNode(mRenderNode);
            }
        }
    }
```
##### 2、文字绘制被加亮
如果以上圆角`ImageView`在`onDraw`中通过`drawText`画了文字，则根据以上分析，`ImageView`中所有的操作会被作加亮处理，解决方式是把`drawText`方法封装到一个`RenderNode`中，这样`ImageView`中只有一个`RenderNode`，`ImageView`中的圆角会被作加暗处理，而文字依然是加亮处理。
```
   @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        drawRoundCornor(canvas);

        if (Build.VERSION.SDK_INT > Build.VERSION_CODES.P) {
            if (mRenderNode == null) {
                mRenderNode = new RenderNode("name");
            }
            Canvas rendCanvas = mRenderNode.beginRecording();
            rendCanvas.translate(-left, -top);//设置位移
            try {
                rendCanvas.drawText("长图", 100,200, mTextPaint);//绘制文字
            } finally {
                mRenderNode.endRecording();
            }
            mRenderNode.setPosition(left,top,right,bottom);//设置renderCanvas的位置
            if (canvas.isHardwareAccelerated() && mRenderNode.hasDisplayList()) {
                canvas.drawRenderNode(mRenderNode);
            }
        } else {
            canvas.drawText("长图", 100,200, mTextPaint);
        }
    }
```
##### 3、RecycleView ItemDecoration被加亮
`RecycleView`的`ItemDecoration`是绘制在`RecycleView`上面的，方法  `onDrawOver`是在绘制完 `RecyelView `之后开始执行，因此在这个方法中的绘制操作会被记录在`RecycleView`的`mDisplayList`中。在`drawOver()`方法中直接调用`header.draw(canvas)`，此时`header`的`mAttachInfo`为空，所以不会走硬件加速绘制。`header`中所有`View`的绘制操作包含`TextView`都会保存在`RecycleView`的`mDisplayList`中。由于`header`中包含`TextView`所以`RecycleView`整体会被加亮，解决方法是增加一个放在`RenderNode`中的绘制背景的操作，这样即保证了文字加亮又有一个加暗的背景。
```
  @Override
    public void onDrawOver(Canvas c, RecyclerView parent, RecyclerView.State state) {
        final int count = parent.getChildCount();
        for (int layoutPos = 0; layoutPos < count; layoutPos++) {
            final View child = parent.getChildAt(layoutPos);
            View header = getHeader(parent, adapterPos).itemView;
            final int left = child.getLeft();
            int top = getHeaderTop(parent, child, header, adapterPos, layoutPos);
            c.save();
            c.translate(left, top);
            if (android.os.Build.VERSION.SDK_INT > android.os.Build.VERSION_CODES.P) {
                Rect rect = new Rect();
                header.getDrawingRect(rect);
                header.findViewById(R.id.tv_header).setBackground(null);
                Canvas rendCanvas = mRenderNode.beginRecording();
                rendCanvas.translate(-rect.left, -rect.top);
                try {
                    rendCanvas.drawRect(rect, mPaint);//绘制文字背景
                } finally {
                    mRenderNode.endRecording();
                }
                mRenderNode.setPosition(rect);
                if (c.isHardwareAccelerated() && mRenderNode.hasDisplayList()) {
                    c.drawRenderNode(mRenderNode);
                }
               
            }
            header.draw(c);
            c.restore();
        }
    }
```
### 六、总结
本文从`AndroidQ`强制黑暗(`ForceDark`)模式的在适配中遇到的问题开始，分析了`AndroidQ`系统黑暗模式的实现原理，并通过解决在适配中遇到的问题，给出了适配黑暗模式的一种方法，希望本文对黑暗模式的适配有一定的帮助。

参考：
1、[https://developer.android.google.cn/preview/features/darktheme](https://developer.android.google.cn/preview/features/darktheme)

2、[https://www.androidos.net.cn/android/10.0.0_r6/xref](https://www.androidos.net.cn/android/10.0.0_r6/xref)

3、[https://blog.csdn.net/shensky711/article/details/102677056](https://blog.csdn.net/shensky711/article/details/102677056)

4、[https://blog.csdn.net/luoshengyang/article/details/45943255](https://blog.csdn.net/luoshengyang/article/details/45943255)

5、[https://blog.csdn.net/luoshengyang/article/details/46281499](https://blog.csdn.net/luoshengyang/article/details/46281499)

