---
title: Android中使用Picasso将图片直接转换为圆形
date: 2016-05-04 23:46:20
categories: Android
tags: [Android, Picasso]
---

## 前言
圆形头像现在很流行了，Github上也有很多开源的库，最经典的是直接使用一个自定义的圆形ImageViwe，比较有代表性的有这个项目：[hdodenhof/CircleImageView][2]。但是，如果你的项目中正好使用[Picasso][1]作为图片异步加载的话，可以直接使用Picasso原生的`Transformation`机制，它允许你在显示图片前做一些转化。

为了避免重复造轮子，先Google大法一下，会发现2个比较多人start的gist：
[aprock/RoundedTransformation.java][3]
[julianshen/CircleTransform.java][4]

## Gist 1
第一个`RoundedTransformation`主要是绘制圆角图形，并且没有处理长方形图片的情况：
```java
@Override
public Bitmap transform(final Bitmap source) {
    final Paint paint = new Paint();
    paint.setAntiAlias(true);
    paint.setShader(new BitmapShader(source, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP));

    Bitmap output = Bitmap.createBitmap(source.getWidth(), source.getHeight(), Config.ARGB_8888);
    Canvas canvas = new Canvas(output);
    canvas.drawRoundRect(new RectF(margin, margin, source.getWidth() - margin, source.getHeight() - margin), radius, radius, paint);

    if (source != output) {
        source.recycle();
    }

    return output;
}
```

## Gist 2
我们来看看第二个`CircleTransform`的代码，是绘制圆形图片，并且有处理长方形图片截取中间部分的逻辑。
```java
@Override
public Bitmap transform(Bitmap source) {
    int size = Math.min(source.getWidth(), source.getHeight());

    int x = (source.getWidth() - size) / 2;
    int y = (source.getHeight() - size) / 2;

    Bitmap squaredBitmap = Bitmap.createBitmap(source, x, y, size, size);
    if (squaredBitmap != source) {
        source.recycle();
    }

    Bitmap bitmap = Bitmap.createBitmap(size, size, source.getConfig());

    Canvas canvas = new Canvas(bitmap);
    Paint paint = new Paint();
    BitmapShader shader = new BitmapShader(squaredBitmap, BitmapShader.TileMode.CLAMP, BitmapShader.TileMode.CLAMP);
    paint.setShader(shader);
    paint.setAntiAlias(true);

    float r = size/2f;
    canvas.drawCircle(r, r, r, paint);

    squaredBitmap.recycle();
    return bitmap;
}
```

## 分析
可以看到上面的两个gist比较核心的一点都是使用`BitmapShader`来绘制圆形图片，其中`TileMode.CLAMP`表示对绘制区域进行着色的模式，它会在超出图片边界的时候拉伸图片最后的那一行（列）像素。

其实第二个gist基本能满足需求了。但是还有一点美中不足，中间创建多了一次square bitmap，他的作用主要是先把长方形的图像转为正方形（居中截取），然后再以它为基础绘制圆形图片。（为啥？就是BitmapShader是从你的画布的左上角开始绘制的，如果直接用长方形的bitmap，绘制出来的圆形图像只能是左/上的那部分，囧...）

那么？该如何进一步优化？能不能把中间那部square bitmap的创建去掉了，这应该能在图片大量加载的时候省下不少开辟内存空间的开销（虽然最后GC还是会回收那部分中间内存，但总觉得不够优雅）。

## 优化
经过一番查询， `BitmapShader`是可以通过`setLocalMatrix()`方法设置一个`Matrix`来进行矩阵变换的，也就是其实是可以不从原图的左上角开始绘制。说到这里，大家应该都知道如何进行优化了，下面贴出代码，省下了中间的square bitmap的创建，棒棒的。代码已上传gist：[codezjx/CircleImageTransformation.java][5]

```java
public class CircleImageTransformation implements Transformation {

    /**
     * A unique key for the transformation, used for caching purposes.
     */
    private static final String KEY = "circleImageTransformation";

    @Override
    public Bitmap transform(Bitmap source) {
        
        int minEdge = Math.min(source.getWidth(), source.getHeight());
        int dx = (source.getWidth() - minEdge) / 2;
        int dy = (source.getHeight() - minEdge) / 2;

        // Init shader
        Shader shader = new BitmapShader(source, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
        Matrix matrix = new Matrix();
        matrix.setTranslate(-dx, -dy);   // Move the target area to center of the source bitmap
        shader.setLocalMatrix(matrix);
        
        // Init paint
        Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
        paint.setShader(shader);

        // Create and draw circle bitmap
        Bitmap output = Bitmap.createBitmap(minEdge, minEdge, source.getConfig());
        Canvas canvas = new Canvas(output);
        canvas.drawOval(new RectF(0, 0, minEdge, minEdge), paint);

        // Recycle the source bitmap, because we already generate a new one
        source.recycle();

        return output;
    }

    @Override
    public String key() {
        return KEY;
    }
}

```

最后需要注意一点：重载的`key()`函数返回的字符串代表着某个配置参数下的Transformation。什么意思呢？也就是说，如果你自定义的`Transformation` 的构造函数中有配置参数的话，如第一个`RoundedTransformation`例子中那样，key()返回的字符串必须包含这些参数，这个key用来做cache的时候会用到。如：
```java
@Override
public String key() {
    return "rounded(radius=" + radius + ", margin=" + margin + ")";
}
```


[1]: https://github.com/square/picasso
[2]: https://github.com/hdodenhof/CircleImageView
[3]: https://gist.github.com/aprock/6213395
[4]: https://gist.github.com/julianshen/5829333
[5]: https://gist.github.com/codezjx/b8a99374385a0210edb9192bced516a3