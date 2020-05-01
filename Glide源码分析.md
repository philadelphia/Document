# Glide源码分析

## 简介

Glide（https://github.com/bumptech/glide）是Android平台一个优秀的图片加载框架

本次分析主要是对glide 的流程做一个大概的分析，不涉及具体的细节。Glide4.8.0版本分析

## 使用

在我们的module的build.gradle 文件中添加

1、添加依赖：

```
implementation("com.github.bumptech.glide:glide:4.8.0"
annotationProcessor 'com.github.bumptech.glide:compiler:4.8.0'
```

然后Sync一下就行了。

2、添加网络权限

`<uses-permission android:name="android.permission.INTERNET" />`

3、调用

`Glide.with(this).load(imgUrl).into(imageView);`

或者

```
RequestOptions options = new RequestOptions()
        .placeholder(R.drawable.ic_launcher_background)
        .error(R.mipmap.ic_launcher)
        .diskCacheStrategy(DiskCacheStrategy.NONE)
        .override(200, 100);

Glide.with(this)
        .load(imgUrl)
        .apply(options)
        .into(iamgeView);
```



## 分析

### Glide.with();

![image-20200501231050252](/Users/meiliwu/Library/Application Support/typora-user-images/image-20200501231050252.png)



with()方法有几个版本，基本都是接受一个Context对象，返回一个RequestManager对象。

```java
@NonNull
public static RequestManager with(@NonNull Context context) {
  return getRetriever(context).get(context);
}
```



其内部实现就是 getRetriever(context)返回一个RequestManagerRetriever，然后调用RequestManagerRetriever对象的get()方法，返回一个RequestManager对象。

```
@NonNull
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
  // Context could be null for other reasons (ie the user passes in null), but in practice it will
  // only occur due to errors with the Fragment lifecycle.
  Preconditions.checkNotNull(
      context,
      "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
          + "returns null (which usually occurs when getActivity() is called before the Fragment "
          + "is attached or after the Fragment is destroyed).");
  return Glide.get(context).getRequestManagerRetriever();
}
```



首先Glide.get(context)返回一个Glide对象，然后调用Glide对象的getRequestManagerRetriever方法。

```
private static void initializeGlide(@NonNull Context context) {
  initializeGlide(context, new GlideBuilder());
}
```

Glide.get（context）方法会先初始化Glide对象，这时候传递一个GlideBuilder对象。由此可以Glide对象使用了Builder模式。

![image-20200501232300791](/Users/meiliwu/Library/Application Support/typora-user-images/image-20200501232300791.png)

由此我们得到了一个Glide对象，此时一个完整的Glide的对象已经产生了，接下来就是使用了

### Glide.with(this).load(imgUrl)

Glide.with()会返回一个RequestManager对象。

然后调用RequestManager对象的load方法。



![image-20200501232737324](/Users/meiliwu/Library/Application Support/typora-user-images/image-20200501232737324.png)

load()方法有多个重载，

```java
@NonNull
@CheckResult
@Override
public RequestBuilder<Drawable> load(@Nullable String string) {
  return asDrawable().load(string);
}
```

首先调用asDrawable()方法

```
@NonNull
@CheckResult
public RequestBuilder<Drawable> asDrawable() {
  return as(Drawable.class);
}
```

生成一个RequestBuilder<Drawable>对象，然后调用该队象的load方法。

```
@NonNull
@CheckResult
public <ResourceType> RequestBuilder<ResourceType> as(
    @NonNull Class<ResourceType> resourceClass) {
  return new RequestBuilder<>(glide, this, resourceClass, context);
}
```

```java
protected RequestBuilder(Glide glide, RequestManager requestManager,
    Class<TranscodeType> transcodeClass, Context context) {
  this.glide = glide;
  this.requestManager = requestManager;
  this.transcodeClass = transcodeClass;
  this.defaultRequestOptions = requestManager.getDefaultRequestOptions();
  this.context = context;
  this.transitionOptions = requestManager.getDefaultTransitionOptions(transcodeClass);
  this.requestOptions = defaultRequestOptions;
  this.glideContext = glide.getGlideContext();
}
```

最后调用RequestBuilder对象的load方法，该方法返回一个RequestBuilder<Drawable>对象

### apply）

apply方法传递一个RequestOptions对象，该对象不传的话Glide会默认使用GlideBuilder对象默认的RequestOptions对象，改对象其实就是Glide加载图片的一个默认配置。

```
public RequestBuilder<TranscodeType> apply(@NonNull RequestOptions requestOptions) {
  Preconditions.checkNotNull(requestOptions);
  this.requestOptions = getMutableOptions().apply(requestOptions);
  return this;
}
```



```
/**
 * Provides type independent options to customize loads with Glide.
 */
```

从该类的简介也可以看出，改类提供了一个类型独立的选项来自定义Glide加载图片。

主要是一些缓存配置，占位符和出错时的配置。

该方法依旧返回一个RequestBuilder<Drawable>对象

### into(ImageView view)

该方法最后是最后一步，最后将加载好的图片设置给ImageView对象。

```java
// TranscodeType 此时是Drawable类型
@NonNull
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
  Util.assertMainThread(); //首先判断是否在主线程，如果不在主线程则抛出异常。
  Preconditions.checkNotNull(view); //检查view是否为null，为null抛出NPE

  RequestOptions requestOptions = this.requestOptions; //取出RequestBuilder对象的RequestOptions
  if (!requestOptions.isTransformationSet() //转换相关
      && requestOptions.isTransformationAllowed()
      && view.getScaleType() != null) {
    // Clone in this method so that if we use this RequestBuilder to load into a View and then
    // into a different target, we don't retain the transformation applied based on the previous
    // View's scale type.
    switch (view.getScaleType()) { //根据ImageView的ScaleType设置RequestOptions
      case CENTER_CROP:
        requestOptions = requestOptions.clone().optionalCenterCrop();
        break;
      case CENTER_INSIDE:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case FIT_CENTER:
      case FIT_START:
      case FIT_END:
        requestOptions = requestOptions.clone().optionalFitCenter();
        break;
      case FIT_XY:
        requestOptions = requestOptions.clone().optionalCenterInside();
        break;
      case CENTER:
      case MATRIX:
      default:
        // Do nothing.
    }
  }

  return into(
      glideContext.buildImageViewTarget(view, transcodeClass),
      /*targetListener=*/ null,
      requestOptions);
}
```



首先看下

#### glideContext.buildImageViewTarget(view, transcodeClass)



```java
@NonNull
public <X> ViewTarget<ImageView, X> buildImageViewTarget(
    @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
  return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
}

public class ImageViewTargetFactory {
  @NonNull
  @SuppressWarnings("unchecked")
  public <Z> ViewTarget<ImageView, Z> buildTarget(@NonNull ImageView view,
      @NonNull Class<Z> clazz) {
    //由此可知道，Glide最后加载的只有Drawable和Bitamap
    if (Bitmap.class.equals(clazz)) {
      return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);
    } else if (Drawable.class.isAssignableFrom(clazz)) {
      return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);
    } else {
      throw new IllegalArgumentException(
          "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
    }
  }
}
```

最后生成一个ViewTarget<ImageView, Drawabel>对象

#### into()

```
private <Y extends Target<TranscodeType>> Y into(
    @NonNull Y target,
    @Nullable RequestListener<TranscodeType> targetListener,
    @NonNull RequestOptions options) {
  Util.assertMainThread();
  Preconditions.checkNotNull(target);
  if (!isModelSet) {
    throw new IllegalArgumentException("You must call #load() before calling #into()");
  }

  options = options.autoClone();
  Request request = buildRequest(target, targetListener, options);

  Request previous = target.getRequest();
  if (request.isEquivalentTo(previous)
      && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
    request.recycle();
    // If the request is completed, beginning again will ensure the result is re-delivered,
    // triggering RequestListeners and Targets. If the request is failed, beginning again will
    // restart the request, giving it another chance to complete. If the request is already
    // running, we can let it continue running without interruption.
    if (!Preconditions.checkNotNull(previous).isRunning()) {
      // Use the previous request rather than the new one to allow for optimizations like skipping
      // setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
      // that are done in the individual Request.
      previous.begin();
    }
    return target;
  }

  requestManager.clear(target);
  target.setRequest(request);
  requestManager.track(target, request);

  return target;
}
```



### ModelTypes

这里有一个重要的接口ModelTypes，它有两实现类RequestBuilder和RequestManager

```
interface ModelTypes<T> {
  @NonNull
  @CheckResult
  T load(@Nullable Bitmap bitmap);

  @NonNull
  @CheckResult
  T load(@Nullable Drawable drawable);

  @NonNull
  @CheckResult
  T load(@Nullable String string);

  @NonNull
  @CheckResult
  T load(@Nullable Uri uri);

  @NonNull
  @CheckResult
  T load(@Nullable File file);

  @NonNull
  @CheckResult
  T load(@RawRes @DrawableRes @Nullable Integer resourceId);

  @Deprecated
  @CheckResult
  T load(@Nullable URL url);

  @NonNull
  @CheckResult
  T load(@Nullable byte[] model);

  @NonNull
  @CheckResult
  @SuppressWarnings("unchecked")
  T load(@Nullable Object model);
}
```

### 总结

图片加载本质上也是一个网络相关的任务执行框架，本质上与OKHttp没有区别。从代码也可以看出。即首先构建出一个Request对象。

然后加入到任务队列中执行，成功或者失败后执行响应的回调。

