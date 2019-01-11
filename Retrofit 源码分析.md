## 

# 简介

retrofit是square出品的一个优秀的网络框架，注意，不是一个网络引擎。它的定位和Volley是一样的。

它完成了封装请求，线程切换，数据装换等一系列工作，如果自己有能力也可以封装一个这种框架，本质上是没有区别的。

retrofit使用的网络引擎是OkHttp.

而OKHttp和HTTPClient，HttpUrlConnection是一个级别的。

#使用

```java
//1 创建网络请求接口类
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}

//2 创建Retrofit实例对象
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
     .addConverterFactory(GsonConverterFactory.create())
     .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
    .build();

//3 通过动态代理创建网络接口代理对象
GitHubService service = retrofit.create(GitHubService.class);

//4 获取Call对象
Call<List<Repo>> repos = service.listRepos("octocat");

//5	执行同步请求或异步请求
repos.execute();
repos.enqueue(callback)
```

# Retrofit

Retrofit也是使用Build模式创建的。

![屏幕快照 2019-01-11 下午3.12.35](/Users/meiliwu/Desktop/屏幕快照 2019-01-11 下午3.12.35.png)

builder类有这些方法。从图表可以看出，我们可以调用client方法传入一个我们自定义的OkhttpClient,

调用baseUrl方法传入Host,最后调动build方法生成一个Retrofit 对象

```java
public Retrofit build() {
    //baseUrl是必须的
  if (baseUrl == null) {
    throw new IllegalStateException("Base URL required.");
  }

   //如果没有设置callFactory对象，系统自动生成一个OkhttpClient对象.因为OKHttpclient实现了			Call.Factory接口
   // public class OkHttpClient implements Cloneable, Call.Factory, WebSocket.Factory
  okhttp3.Call.Factory callFactory = this.callFactory;
  if (callFactory == null) {
    callFactory = new OkHttpClient();
  }

  //如果没有设置callbackExecutor，系统自动生成一个，platform.defaultCallbackExecutor，这个platform是无参构造方法里调用Platform.get()方法得到的。
    /** 
    public Builder() {
      this(Platform.get());
    }**/
    
  Executor callbackExecutor = this.callbackExecutor;
  if (callbackExecutor == null) {
    callbackExecutor = platform.defaultCallbackExecutor();
  }

  // Make a defensive copy of the adapters and add the default Call adapter.
  List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
  adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

  // Make a defensive copy of the converters.
  List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

  return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
      callbackExecutor, validateEagerly);
}
```

# Platform

```java
class Platform {
  private static final Platform PLATFORM = findPlatform();

  static Platform get() {
    return PLATFORM;
  }

  private static Platform findPlatform() {
    try {
      Class.forName("android.os.Build");
      if (Build.VERSION.SDK_INT != 0) {
        return new Android();
      }
    } catch (ClassNotFoundException ignored) {
    }
    try {
      Class.forName("java.util.Optional");
      return new Java8();
    } catch (ClassNotFoundException ignored) {
    }
    try {
        //怎么还有IOS代码呢？
      Class.forName("org.robovm.apple.foundation.NSObject");
      return new IOS();
    } catch (ClassNotFoundException ignored) {
    }
    return new Platform();
  }

  Executor defaultCallbackExecutor() {
    return null;
  }

  CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
    if (callbackExecutor != null) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }
    return DefaultCallAdapterFactory.INSTANCE;
  }

  boolean isDefaultMethod(Method method) {
    return false;
  }

  Object invokeDefaultMethod(Method method, Class<?> declaringClass, Object object, Object... args)
      throws Throwable {
    throw new UnsupportedOperationException();
  }

  static class Android extends Platform {
    @Override public Executor defaultCallbackExecutor() {
      return new MainThreadExecutor();
    }

    @Override CallAdapter.Factory defaultCallAdapterFactory(Executor callbackExecutor) {
      return new ExecutorCallAdapterFactory(callbackExecutor);
    }

    static class MainThreadExecutor implements Executor {
      private final Handler handler = new Handler(Looper.getMainLooper());

      @Override public void execute(Runnable r) {
        handler.post(r);
      }
    }
  }
}
```



Retrofit 要求必须将请求API写到一个interface接口文件里，这是动态代理特性要求的。

从接口文件里我们可以看到，我们将每个请求用这种形式表达

```Java
public interface GitHubService {
	@GET("users/{user}/repos")
	Call<List<Repo>> listRepos(@Path("user") String user);
}		

```
从接口文件我们可以看出，一个请求接口被各种注解所表示。

我们知道一个方法有一下关键字段组成

首先一个方法必须有描述符，返回值，方法名，参数类型，参数构成。

那我们用一个方法表示一个http请求需要哪些东西呢？

Http请求，首先我们得知道是GET请求还是POST请求，

然后就是请求头信息，请求路径，查询参数等等。

POST请求还需要Body。

Retrofit 已经提供了足够的注解来表示一个方法。

Retrofit的核心思想AOP，面向切面变成，通过动态代理的反射，将接口文件里的每个方法记性处理，也就是分析该方法的注解生成一个ServiceMethod类。

Retrofit 里有个关键的类，ServiceMethod

```java
@SuppressWarnings("unchecked") // Single-interface proxy creation guarded by parameter safety.
public <T> T create(final Class<T> service) {
  Utils.validateServiceInterface(service);
  if (validateEagerly) {
    eagerlyValidateMethods(service);
  }
  return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
      new InvocationHandler() {
        private final Platform platform = Platform.get();

        @Override public Object invoke(Object proxy, Method method, Object... args)
            throws Throwable {
          // If the method is a method from Object then defer to normal invocation.
          if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
          }
          if (platform.isDefaultMethod(method)) {
            return platform.invokeDefaultMethod(method, service, proxy, args);
          }
            //创建ServiceMethod对象
          ServiceMethod serviceMethod = loadServiceMethod(method);
          OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
          return serviceMethod.callAdapter.adapt(okHttpCall);
        }
      });
}
```

从第3步我们可以看出create方法的实现就是使用了动态代理，在运行时生成了GitHubService对象。

 //创建ServiceMethod对象
 ServiceMethod serviceMethod = loadServiceMethod(method);

```java
ServiceMethod loadServiceMethod(Method method) {
  ServiceMethod result;
  synchronized (serviceMethodCache) {
  //先从换从中取改方法对应的ServiceMethod对象，如果为null就构建一个ServiceMethod对象并存入到map中，如果不为null直接返回
    result = serviceMethodCache.get(method);
    if (result == null) {
      result = new ServiceMethod.Builder(this, method).build();
      serviceMethodCache.put(method, result);
    }
  }
  return result;
}
```

我们可以看到loadServiceMethod(Method method)方法返回了一个ServiceMethod对象
 这个serviceMethodCache对象是Retrofit的一个字段，是一个Map集合。

`private final Map<Method, ServiceMethod> serviceMethodCache = new LinkedHashMap<>();`

将接口文件里每个方法转换为一个ServiceMethod对象后放入改map中作为缓存，下次调用该方法后就不用再次解析改方法对象了，直接从改map里去以方法为key去取对应的ServiceMethod就行了。666

接下来看一下ServiceMethod对象的构造

# ServiceMethod

```java
final class ServiceMethod<T> {
  // Upper and lower characters, digits, underscores, and hyphens, starting with a character.
  static final String PARAM = "[a-zA-Z][a-zA-Z0-9_-]*";
  static final Pattern PARAM_URL_REGEX = Pattern.compile("\\{(" + PARAM + ")\\}");
  static final Pattern PARAM_NAME_REGEX = Pattern.compile(PARAM);

  final okhttp3.Call.Factory callFactory;
  final CallAdapter<?> callAdapter;

  private final HttpUrl baseUrl; 主机地址
  private final Converter<ResponseBody, T> responseConverter;
  private final String httpMethod; 
  private final String relativeUrl; 相对路径
  private final Headers headers;    请求头部信息
  private final MediaType contentType; 请求参数类型
  private final boolean hasBody;  是否有请求体
  private final boolean isFormEncoded; 是否是格式化的表单
  private final boolean isMultipart; 是不是分块
  private final ParameterHandler<?>[] parameterHandlers;

  ServiceMethod(Builder<T> builder) {
    this.callFactory = builder.retrofit.callFactory();
    this.callAdapter = builder.callAdapter;
    this.baseUrl = builder.retrofit.baseUrl();
    this.responseConverter = builder.responseConverter;
    this.httpMethod = builder.httpMethod;
    this.relativeUrl = builder.relativeUrl;
    this.headers = builder.headers;
    this.contentType = builder.contentType;
    this.hasBody = builder.hasBody;
    this.isFormEncoded = builder.isFormEncoded;
    this.isMultipart = builder.isMultipart;
    this.parameterHandlers = builder.parameterHandlers;
  }
}
```

ServiceMethod是采用Builder模式创建的。

```java
static final class Builder<T> {
  final Retrofit retrofit;
  final Method method; 		//接口里生命的方法
  final Annotation[] methodAnnotations;  //方法的注解，get/post/header之类的
  final Annotation[][] parameterAnnotationsArray; //方法的参数注解数组，二维数组
  final Type[] parameterTypes;  //方法的参数数组

  Type responseType;
  boolean gotField;
  boolean gotPart;
  boolean gotBody;
  boolean gotPath;
  boolean gotQuery;
  boolean gotUrl;
  String httpMethod;
  boolean hasBody;
  boolean isFormEncoded;
  boolean isMultipart;
  String relativeUrl;
  Headers headers;
  MediaType contentType;
  Set<String> relativeUrlParamNames;
  ParameterHandler<?>[] parameterHandlers;
  Converter<ResponseBody, T> responseConverter;
  CallAdapter<?> callAdapter;

  public Builder(Retrofit retrofit, Method method) {
    this.retrofit = retrofit;
    this.method = method;
    this.methodAnnotations = method.getAnnotations(); //获取方法的注解
    this.parameterTypes = method.getGenericParameterTypes(); //获取被注解修饰的方法，一个数组
    this.parameterAnnotationsArray = method.getParameterAnnotations(); //获取方法的参数注解信息，是一个二维数组
  }
```

Builder的构造参数需要一个Retrofit对象和一个Method对象。

首先解析方法对象，将其注解和参数注解放到对应的数组里。

首先在构造方法里获取该方法的注解,方法的参数，以及每个参数的注解。

关键就在build方法，在build方法里对方法做了一个彻底的分解

```java
public ServiceMethod build() {
  //1 处理返回结果，做一定的转换
  callAdapter = createCallAdapter();
  responseType = callAdapter.responseType();
  if (responseType == Response.class || responseType == okhttp3.Response.class) {
    throw methodError("'"
        + Utils.getRawType(responseType).getName()
        + "' is not a valid response body type. Did you mean ResponseBody?");
  }
  responseConverter = createResponseConverter();

    //2提取方法的注解
  for (Annotation annotation : methodAnnotations) {
    parseMethodAnnotation(annotation);
  }
	//如果httpMethod为null，即没有使用方法类型注解修饰，抛出异常进行提示
  if (httpMethod == null) {
    throw methodError("HTTP method annotation is required (e.g., @GET, @POST, etc.).");
  }
//如果没有请求体，即使用了GET，HEAD，DELETE，OPTIONS等所修饰，即不涉及到表单的提交，但是同时使用了Multipart，或者FormUrlEncoded所修饰，就报错
  if (!hasBody) {
    if (isMultipart) {
      throw methodError(
          "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
    }
    if (isFormEncoded) {
      throw methodError("FormUrlEncoded can only be specified on HTTP methods with "
          + "request body (e.g., @POST).");
    }
  }

  //3提取方法的参数
  int parameterCount = parameterAnnotationsArray.length;
  parameterHandlers = new ParameterHandler<?>[parameterCount];
  for (int p = 0; p < parameterCount; p++) {
    Type parameterType = parameterTypes[p];
    if (Utils.hasUnresolvableType(parameterType)) {
      throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
          parameterType);
    }

    Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
    if (parameterAnnotations == null) {
      throw parameterError(p, "No Retrofit annotation found.");
    }

    parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
  }

  //相对路径为null且gotURL为false的话，抛出异常，因为没有相对路径无法请求。
  if (relativeUrl == null && !gotUrl) {
    throw methodError("Missing either @%s URL or @Url parameter.", httpMethod);
  }
  //没有使用@FormUrlEncoded，@Multipart主机并且hasBody为false，但是gotBody为true，抛出异常，提示
    Non-Body类型的HTTP method 不能参数不能使用@Body注解
  if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
    throw methodError("Non-body HTTP method cannot contain @Body.");
  }
  //使用@FormUrlEncoded修饰的方法中的参数至少有一个参数被@Field注解修饰
  if (isFormEncoded && !gotField) {
    throw methodError("Form-encoded method must contain at least one @Field.");
  }
    
  //使用@Multipart修饰的方法中的参数至少有一个参数被@Part注解修饰
  if (isMultipart && !gotPart) {
    throw methodError("Multipart method must contain at least one @Part.");
  }

 //4 当前Builder对象初始化完毕，可以用来够着ServiceMethod对象。
  return new ServiceMethod<>(this);
}
```

## 处理返回结果

```java
private CallAdapter<?> createCallAdapter() {
  //获取方法的返回结果,如果有不能解析的类型则抛出异常，也就是说接口中定义的方法的返回值不能使用泛型
  Type returnType = method.getGenericReturnType();
  if (Utils.hasUnresolvableType(returnType)) {
    throw methodError(
        "Method return type must not include a type variable or wildcard: %s", returnType);
  }
   //接口里的方法不能返回void
  if (returnType == void.class) {
    throw methodError("Service methods cannot return void.");
  }
  Annotation[] annotations = method.getAnnotations();
  try {
    return retrofit.callAdapter(returnType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    //用户自定义的Adapter可能不能正确的处理返回结果，这时候抛出异常
    throw methodError(e, "Unable to create call adapter for %s", returnType);
  }
}
```















## 解析方法注解

1处处理方法的注解，就是先处理GET/POST/Header等注解信息

```
private void parseMethodAnnotation(Annotation annotation) {
  if (annotation instanceof DELETE) {
    parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
  } else if (annotation instanceof GET) {
    parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
  } else if (annotation instanceof HEAD) {
    parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
    if (!Void.class.equals(responseType)) {
      throw methodError("HEAD method must use Void as response type.");
    }
  } else if (annotation instanceof PATCH) {
    parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
  } else if (annotation instanceof POST) {
    parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
  } else if (annotation instanceof PUT) {
    parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
  } else if (annotation instanceof OPTIONS) {
    parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
  } else if (annotation instanceof HTTP) {
    HTTP http = (HTTP) annotation;
    parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
  } else if (annotation instanceof retrofit2.http.Headers) {
  headers注解
    String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
    if (headersToParse.length == 0) {
      throw methodError("@Headers annotation is empty.");
    }
    headers = parseHeaders(headersToParse);
  } else if (annotation instanceof Multipart) {//如果是Multipart注解
    if (isFormEncoded) {
    //如果同时使用了FormUrlEncoded注解报错
      throw methodError("Only one encoding annotation is allowed.");
    }
    isMultipart = true;
  } else if (annotation instanceof FormUrlEncoded) {
    if (isMultipart) {
    //如果同时使用了Multipart注解报错,从这我们可以看出一个方法不能同时被Multipart和FormUrlEncoded所修饰
      throw methodError("Only one encoding annotation is allowed.");
    }
    isFormEncoded = true;
  }
}
```

然后根据具体的注解类型，在做进一步的处理，这里主要分析GET/POST/HEADER/ 等注解

### @GET

```java
else if (annotation instanceof GET) {
  parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
} 
```

get类型的请求，没有请求体

```java
private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
 //如果该Builder已经有HTTPMethod了就不能改变了，直接抛异常
    if (this.httpMethod != null) {
    throw methodError("Only one HTTP method is allowed. Found: %s and %s.",
        this.httpMethod, httpMethod);
  }
  //将HTTPMethod赋值给httpMethod对象，Get、Post、Delete等
  this.httpMethod = httpMethod;
  this.hasBody = hasBody;//是否有请求体

    //如果value为null，返回，因为value参数的值其实就是relativeURL。所以不能为null
  if (value.isEmpty()) {
    return;
  }

  // Get the relative URL path and existing query string, if present.
  int question = value.indexOf('?');
  if (question != -1 && question < value.length() - 1) {
    // Ensure the query string does not have any named parameters.
    String queryParams = value.substring(question + 1);
      //获取查询参数
    Matcher queryParamMatcher = PARAM_URL_REGEX.matcher(queryParams);
    if (queryParamMatcher.find()) {
        //如果在value里面找到里查询参数的话，抛出异常。因为查询参数可以使用@Query注解来动态配置。
      throw methodError("URL query string \"%s\" must not have replace block. "
          + "For dynamic query parameters use @Query.", queryParams);
    }
  }

  this.relativeUrl = value; //将value赋值给relativeUrl
  this.relativeUrlParamNames = parsePathParameters(value); //获取value里面的path占位符，如果有的话
}
```

再来看下解析value里的path占位符的方法。

```
/**
获取已知URI里面的路径集合，如果一个参数被使用了两次，它只会在set中出现一次，好拗口啊，使用LinkedHashSet来保存path参数集合，保证了路径参数的顺序。
 * Gets the set of unique path parameters used in the given URI. If a parameter is used twice
 * in the URI, it will only show up once in the set.
 */
static Set<String> parsePathParameters(String path) {
  Matcher m = PARAM_URL_REGEX.matcher(path);
  Set<String> patterns = new LinkedHashSet<>();
  while (m.find()) {
    patterns.add(m.group(1));
  }
  return patterns;
}
```

至此，GET方法的相关的注解分析完毕

### @POST

```Java
else if (annotation instanceof POST) {
  parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
} 
```

POST类型的请求，没有请求体。所以hasBody参数为true。

parseHttpMethodAndPath()方法已将在GET方法里面分析过了，这里面都一样。

其他的请求类型也是大同小异。

然后接着分析方法的Header注解

### @Headers

```java
else if (annotation instanceof retrofit2.http.Headers) {
 //   首先获取Headers注解的值，是一个字符串数组。
  String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
    如果header注解长度为0，抛出异常，所以使用了header注解必须设置值，不能存在空的header
  if (headersToParse.length == 0) {
    throw methodError("@Headers annotation is empty.");
  }
    处理header信息，我猜肯定是一个map
  headers = parseHeaders(headersToParse);
```

啊，居然不是，666.因为header不是KV结构的数据类型，而是一个key可以对应多个值。理论上可以使用Map<String,Set<String>>表示。

```
private Headers parseHeaders(String[] headers) {
  Headers.Builder builder = new Headers.Builder();
  for (String header : headers) {
  // header以“:"分割，前面是key,后面是value
    int colon = header.indexOf(':');
    if (colon == -1 || colon == 0 || colon == header.length() - 1) {
    //header必须是key:value格式表示，不然报错
      throw methodError(
          "@Headers value must be in the form \"Name: Value\". Found: \"%s\"", header);
    }
    String headerName = header.substring(0, colon); //key值
    String headerValue = header.substring(colon + 1).trim(); //value值，必须是一个数组，艹，又猜错了。
    if ("Content-Type".equalsIgnoreCase(headerName)) {
    //遇到"Content-Type"字段。还需要获得具体的MediaType。
      MediaType type = MediaType.parse(headerValue);
      if (type == null) {
      //如果mediaType为null。抛出一个type畸形的错误。
        throw methodError("Malformed content type: %s", headerValue);
      }
      contentType = type;
    } else {
    将header的key和value加入到Builder里面。
      builder.add(headerName, headerValue);
    }
  }
  最后调用build方法生成一个Header对爱。
  return builder.build();
}
```

```java
/**
 * Add a header with the specified name and value. Does validation of header names and values.
 */
public Builder add(String name, String value) {
  checkNameAndValue(name, value);
  return addLenient(name, value);
}
```

```Java
Builder addLenient(String name, String value) {
  namesAndValues.add(name);
  namesAndValues.add(value.trim());
  return this;
}
```

```java
final List<String> namesAndValues = new ArrayList<>(20);
```

namesAndValues是Header.Builder类的一种子段。可见在Builder内部header信息是按照key/value异常放到一个String集合里面的。为什么不放到一个Map里面呢，不懂。

总之，最后就是讲方法的Headers注解信息提取完毕。



## 处理方法参数

```java
int parameterCount = parameterAnnotationsArray.length; //求得数组的长度
parameterHandlers = new ParameterHandler<?>[parameterCount];
for (int p = 0; p < parameterCount; p++) {
  Type parameterType = parameterTypes[p]; //便利参数，依次处理参数
    //如果参数不能解析，抛出异常
  if (Utils.hasUnresolvableType(parameterType)) {
    throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
        parameterType);
  }
//获取第p个参数的注解数组，如果没有注解抛出异常，可见，使用了Retrofit，接口方法中每个参数都必须使用注解进行修饰。
  Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
  if (parameterAnnotations == null) {
    throw parameterError(p, "No Retrofit annotation found.");
  }

   //解析方法中的参数，存入parameterHandlers[]数组中。
  parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
}
```

## 参数校验

Utils.hasUnresolvableType(parameterType)，这个方法是对参数的类型做个校验。

```
static boolean hasUnresolvableType(Type type) {
  //如果参数是引用数据类型，返回false，可见，接口定义中方法的参数只能是基本数据类型
  if (type instanceof Class<?>) {
    return false;
  }
  //如果参数是泛型
  if (type instanceof ParameterizedType) {
    ParameterizedType parameterizedType = (ParameterizedType) type;
    //去除泛型类中的实际类型，遍历
    for (Type typeArgument : parameterizedType.getActualTypeArguments()) {
    //如果有一个泛型参数是基本数据类型，返回true，都不是返回false
      if (hasUnresolvableType(typeArgument)) {
        return true;
      }
    }
    return false;
  }
  //如果参数是泛型数组类型
  if (type instanceof GenericArrayType) {
    return hasUnresolvableType(((GenericArrayType) type).getGenericComponentType());
  }
  if (type instanceof TypeVariable) {
    return true;
  }
  if (type instanceof WildcardType) {
    return true;
  }
  String className = type == null ? "null" : type.getClass().getName();
  throw new IllegalArgumentException("Expected a Class, ParameterizedType, or "
      + "GenericArrayType, but <" + type + "> is of type " + className);
}

```

## 解析参数

```java
parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
private ParameterHandler<?> parseParameter(
        int p, Type parameterType, Annotation[] annotations) {
      ParameterHandler<?> result = null;
    //遍历参数的注解数组，调用parseParameterAnnotation()
      for (Annotation annotation : annotations) {
        ParameterHandler<?> annotationAction = parseParameterAnnotation(
            p, parameterType, annotations, annotation);
		//如果该注解没有返回，则解析下一个注解
        if (annotationAction == null) {
          continue;
        }

        if (result != null) {
          throw parameterError(p, "Multiple Retrofit annotations found, only one allowed.");
        }
		
        result = annotationAction; //将解析的结果赋值给Result
      }

    //如果注解为null，抛出异常。这个地方永远不会调用，因为在获取注解数组之前就做过判断了，如果注解数组为null，直接抛异常，Line197-Line200 in ServiceMethod.Builder中
      if (result == null) {
        throw parameterError(p, "No Retrofit annotation found.");
      }

      return result;
    }
```

## 获取参数注解信息

再来看看parseParameterAnnotation()方法,内容略多

```Java
private ParameterHandler<?> parseParameterAnnotation(
    int p, Type type, Annotation[] annotations, Annotation annotation) {
  if (annotation instanceof Url) {
      //如果使用了Url注解，
    if (gotUrl) {
        //如果gotUrl为true，因为gotURL默认为false，说明之前处理过Url注解了,抛出多个@Url注解异常
      throw parameterError(p, "Multiple @Url method annotations found.");
    }
    if (gotPath) {
        //如果gotPath为true，抛出异常，说明@Path注解不能和@Url注解一起使用
      throw parameterError(p, "@Path parameters may not be used with @Url.");
    }
    if (gotQuery) {
        //如果gotQuery为true，抛出异常，说明@Url注解不能用在@Query注解后面
      throw parameterError(p, "A @Url parameter must not come after a @Query");
    }
    if (relativeUrl != null) {
        //如果relativeUrl不为null,抛出异常，说明使用了@Url注解，relativeUrl必须为null
      throw parameterError(p, "@Url cannot be used with @%s URL", httpMethod);
    }

    gotUrl = true;
      
----------------------------------------------------------------------------------------    
      //如果参数类型是HttpURL，String，URI或者参数类型是“android.net.Uri",返回ParameterHandler.RelativeUrl()，实际是交由这个类处理
    if (type == HttpUrl.class
        || type == String.class
        || type == URI.class
        || (type instanceof Class && "android.net.Uri".equals(((Class<?>) type).getName()))) {
      return new ParameterHandler.RelativeUrl();
    } else {
        //不然就抛出异常，也就是说@Url注解必须使用在okhttp3.HttpUrl, String, java.net.URI, or android.net.Uri 这几种类型的参数上。
      throw parameterError(p,
          "@Url must be okhttp3.HttpUrl, String, java.net.URI, or android.net.Uri type.");
    }
------------------------------------------------------------------------------------------
  } else if (annotation instanceof Path) { //@Path注解
     //如果gotQuery为true。抛出异常，因为@Path修饰的参数是路径的占位符。不是查询参数，不能使用@Query注解修饰
    if (gotQuery) {
      throw parameterError(p, "A @Path parameter must not come after a @Query.");
    }
    if (gotUrl) {
      throw parameterError(p, "@Path parameters may not be used with @Url.");
    }
      //如果相对路径为null，那@path注解也就无意义了。
    if (relativeUrl == null) {
      throw parameterError(p, "@Path can only be used with relative url on @%s", httpMethod);
    }
    gotPath = true;

    Path path = (Path) annotation;
    String name = path.value(); //获取@Path注解的值
    validatePathName(p, name); //对改值进行校验，1该value必须是合法字符，2:该相对路径必须包含相应的占位符
 
      //然后将改参数的所有注解进行处理，最终调用ParameterHandler.Path进行处理。
    Converter<?, String> converter = retrofit.stringConverter(type, annotations);
    return new ParameterHandler.Path<>(name, converter, path.encoded());

  } else if (annotation instanceof Query) { //Query注解，看不太懂，最后也是调用ParameterHandler.Query进行处理
    Query query = (Query) annotation;
    String name = query.value();
    boolean encoded = query.encoded();

    Class<?> rawParameterType = Utils.getRawType(type);
    gotQuery = true;
    if (Iterable.class.isAssignableFrom(rawParameterType)) {
      if (!(type instanceof ParameterizedType)) {
        throw parameterError(p, rawParameterType.getSimpleName()
            + " must include generic type (e.g., "
            + rawParameterType.getSimpleName()
            + "<String>)");
      }
      ParameterizedType parameterizedType = (ParameterizedType) type;
      Type iterableType = Utils.getParameterUpperBound(0, parameterizedType);
      Converter<?, String> converter =
          retrofit.stringConverter(iterableType, annotations);
      return new ParameterHandler.Query<>(name, converter, encoded).iterable();
    } else if (rawParameterType.isArray()) {
      Class<?> arrayComponentType = boxIfPrimitive(rawParameterType.getComponentType());
      Converter<?, String> converter =
          retrofit.stringConverter(arrayComponentType, annotations);
      return new ParameterHandler.Query<>(name, converter, encoded).array();
    } else {
      Converter<?, String> converter =
          retrofit.stringConverter(type, annotations);
      return new ParameterHandler.Query<>(name, converter, encoded);
    }

  } else if (annotation instanceof QueryMap) {
    Class<?> rawParameterType = Utils.getRawType(type);
    if (!Map.class.isAssignableFrom(rawParameterType)) {
      throw parameterError(p, "@QueryMap parameter type must be Map.");
    }
    Type mapType = Utils.getSupertype(type, rawParameterType, Map.class);
    if (!(mapType instanceof ParameterizedType)) {
      throw parameterError(p, "Map must include generic types (e.g., Map<String, String>)");
    }
    ParameterizedType parameterizedType = (ParameterizedType) mapType;
    Type keyType = Utils.getParameterUpperBound(0, parameterizedType);
    if (String.class != keyType) {
      throw parameterError(p, "@QueryMap keys must be of type String: " + keyType);
    }
    Type valueType = Utils.getParameterUpperBound(1, parameterizedType);
    Converter<?, String> valueConverter =
        retrofit.stringConverter(valueType, annotations);

    return new ParameterHandler.QueryMap<>(valueConverter, ((QueryMap) annotation).encoded());

  } else if (annotation instanceof Header) {
    Header header = (Header) annotation;
    String name = header.value();

    Class<?> rawParameterType = Utils.getRawType(type);
    if (Iterable.class.isAssignableFrom(rawParameterType)) {
      if (!(type instanceof ParameterizedType)) {
        throw parameterError(p, rawParameterType.getSimpleName()
            + " must include generic type (e.g., "
            + rawParameterType.getSimpleName()
            + "<String>)");
      }
      ParameterizedType parameterizedType = (ParameterizedType) type;
      Type iterableType = Utils.getParameterUpperBound(0, parameterizedType);
      Converter<?, String> converter =
          retrofit.stringConverter(iterableType, annotations);
      return new ParameterHandler.Header<>(name, converter).iterable();
    } else if (rawParameterType.isArray()) {
      Class<?> arrayComponentType = boxIfPrimitive(rawParameterType.getComponentType());
      Converter<?, String> converter =
          retrofit.stringConverter(arrayComponentType, annotations);
      return new ParameterHandler.Header<>(name, converter).array();
    } else {
      Converter<?, String> converter =
          retrofit.stringConverter(type, annotations);
      return new ParameterHandler.Header<>(name, converter);
    }

  } else if (annotation instanceof HeaderMap) {
    Class<?> rawParameterType = Utils.getRawType(type);
    if (!Map.class.isAssignableFrom(rawParameterType)) {
      throw parameterError(p, "@HeaderMap parameter type must be Map.");
    }
    Type mapType = Utils.getSupertype(type, rawParameterType, Map.class);
    if (!(mapType instanceof ParameterizedType)) {
      throw parameterError(p, "Map must include generic types (e.g., Map<String, String>)");
    }
    ParameterizedType parameterizedType = (ParameterizedType) mapType;
    Type keyType = Utils.getParameterUpperBound(0, parameterizedType);
    if (String.class != keyType) {
      throw parameterError(p, "@HeaderMap keys must be of type String: " + keyType);
    }
    Type valueType = Utils.getParameterUpperBound(1, parameterizedType);
    Converter<?, String> valueConverter =
        retrofit.stringConverter(valueType, annotations);

    return new ParameterHandler.HeaderMap<>(valueConverter);

  } else if (annotation instanceof Field) {
    if (!isFormEncoded) {
      throw parameterError(p, "@Field parameters can only be used with form encoding.");
    }
    Field field = (Field) annotation;
    String name = field.value();
    boolean encoded = field.encoded();

    gotField = true;

    Class<?> rawParameterType = Utils.getRawType(type);
    if (Iterable.class.isAssignableFrom(rawParameterType)) {
      if (!(type instanceof ParameterizedType)) {
        throw parameterError(p, rawParameterType.getSimpleName()
            + " must include generic type (e.g., "
            + rawParameterType.getSimpleName()
            + "<String>)");
      }
      ParameterizedType parameterizedType = (ParameterizedType) type;
      Type iterableType = Utils.getParameterUpperBound(0, parameterizedType);
      Converter<?, String> converter =
          retrofit.stringConverter(iterableType, annotations);
      return new ParameterHandler.Field<>(name, converter, encoded).iterable();
    } else if (rawParameterType.isArray()) {
      Class<?> arrayComponentType = boxIfPrimitive(rawParameterType.getComponentType());
      Converter<?, String> converter =
          retrofit.stringConverter(arrayComponentType, annotations);
      return new ParameterHandler.Field<>(name, converter, encoded).array();
    } else {
      Converter<?, String> converter =
          retrofit.stringConverter(type, annotations);
      return new ParameterHandler.Field<>(name, converter, encoded);
    }

  } else if (annotation instanceof FieldMap) {
    if (!isFormEncoded) {
      throw parameterError(p, "@FieldMap parameters can only be used with form encoding.");
    }
    Class<?> rawParameterType = Utils.getRawType(type);
    if (!Map.class.isAssignableFrom(rawParameterType)) {
      throw parameterError(p, "@FieldMap parameter type must be Map.");
    }
    Type mapType = Utils.getSupertype(type, rawParameterType, Map.class);
    if (!(mapType instanceof ParameterizedType)) {
      throw parameterError(p,
          "Map must include generic types (e.g., Map<String, String>)");
    }
    ParameterizedType parameterizedType = (ParameterizedType) mapType;
    Type keyType = Utils.getParameterUpperBound(0, parameterizedType);
    if (String.class != keyType) {
      throw parameterError(p, "@FieldMap keys must be of type String: " + keyType);
    }
    Type valueType = Utils.getParameterUpperBound(1, parameterizedType);
    Converter<?, String> valueConverter =
        retrofit.stringConverter(valueType, annotations);

    gotField = true;
    return new ParameterHandler.FieldMap<>(valueConverter, ((FieldMap) annotation).encoded());

  } else if (annotation instanceof Part) {
    if (!isMultipart) {
      throw parameterError(p, "@Part parameters can only be used with multipart encoding.");
    }
    Part part = (Part) annotation;
    gotPart = true;

    String partName = part.value();
    Class<?> rawParameterType = Utils.getRawType(type);
    if (partName.isEmpty()) {
      if (Iterable.class.isAssignableFrom(rawParameterType)) {
        if (!(type instanceof ParameterizedType)) {
          throw parameterError(p, rawParameterType.getSimpleName()
              + " must include generic type (e.g., "
              + rawParameterType.getSimpleName()
              + "<String>)");
        }
        ParameterizedType parameterizedType = (ParameterizedType) type;
        Type iterableType = Utils.getParameterUpperBound(0, parameterizedType);
        if (!MultipartBody.Part.class.isAssignableFrom(Utils.getRawType(iterableType))) {
          throw parameterError(p,
              "@Part annotation must supply a name or use MultipartBody.Part parameter type.");
        }
        return ParameterHandler.RawPart.INSTANCE.iterable();
      } else if (rawParameterType.isArray()) {
        Class<?> arrayComponentType = rawParameterType.getComponentType();
        if (!MultipartBody.Part.class.isAssignableFrom(arrayComponentType)) {
          throw parameterError(p,
              "@Part annotation must supply a name or use MultipartBody.Part parameter type.");
        }
        return ParameterHandler.RawPart.INSTANCE.array();
      } else if (MultipartBody.Part.class.isAssignableFrom(rawParameterType)) {
        return ParameterHandler.RawPart.INSTANCE;
      } else {
        throw parameterError(p,
            "@Part annotation must supply a name or use MultipartBody.Part parameter type.");
      }
    } else {
      Headers headers =
          Headers.of("Content-Disposition", "form-data; name=\"" + partName + "\"",
              "Content-Transfer-Encoding", part.encoding());

      if (Iterable.class.isAssignableFrom(rawParameterType)) {
        if (!(type instanceof ParameterizedType)) {
          throw parameterError(p, rawParameterType.getSimpleName()
              + " must include generic type (e.g., "
              + rawParameterType.getSimpleName()
              + "<String>)");
        }
        ParameterizedType parameterizedType = (ParameterizedType) type;
        Type iterableType = Utils.getParameterUpperBound(0, parameterizedType);
        if (MultipartBody.Part.class.isAssignableFrom(Utils.getRawType(iterableType))) {
          throw parameterError(p, "@Part parameters using the MultipartBody.Part must not "
              + "include a part name in the annotation.");
        }
        Converter<?, RequestBody> converter =
            retrofit.requestBodyConverter(iterableType, annotations, methodAnnotations);
        return new ParameterHandler.Part<>(headers, converter).iterable();
      } else if (rawParameterType.isArray()) {
        Class<?> arrayComponentType = boxIfPrimitive(rawParameterType.getComponentType());
        if (MultipartBody.Part.class.isAssignableFrom(arrayComponentType)) {
          throw parameterError(p, "@Part parameters using the MultipartBody.Part must not "
              + "include a part name in the annotation.");
        }
        Converter<?, RequestBody> converter =
            retrofit.requestBodyConverter(arrayComponentType, annotations, methodAnnotations);
        return new ParameterHandler.Part<>(headers, converter).array();
      } else if (MultipartBody.Part.class.isAssignableFrom(rawParameterType)) {
        throw parameterError(p, "@Part parameters using the MultipartBody.Part must not "
            + "include a part name in the annotation.");
      } else {
        Converter<?, RequestBody> converter =
            retrofit.requestBodyConverter(type, annotations, methodAnnotations);
        return new ParameterHandler.Part<>(headers, converter);
      }
    }

  } else if (annotation instanceof PartMap) {
    if (!isMultipart) {
      throw parameterError(p, "@PartMap parameters can only be used with multipart encoding.");
    }
    gotPart = true;
    Class<?> rawParameterType = Utils.getRawType(type);
    if (!Map.class.isAssignableFrom(rawParameterType)) {
      throw parameterError(p, "@PartMap parameter type must be Map.");
    }
    Type mapType = Utils.getSupertype(type, rawParameterType, Map.class);
    if (!(mapType instanceof ParameterizedType)) {
      throw parameterError(p, "Map must include generic types (e.g., Map<String, String>)");
    }
    ParameterizedType parameterizedType = (ParameterizedType) mapType;

    Type keyType = Utils.getParameterUpperBound(0, parameterizedType);
    if (String.class != keyType) {
      throw parameterError(p, "@PartMap keys must be of type String: " + keyType);
    }

    Type valueType = Utils.getParameterUpperBound(1, parameterizedType);
    if (MultipartBody.Part.class.isAssignableFrom(Utils.getRawType(valueType))) {
      throw parameterError(p, "@PartMap values cannot be MultipartBody.Part. "
          + "Use @Part List<Part> or a different value type instead.");
    }

    Converter<?, RequestBody> valueConverter =
        retrofit.requestBodyConverter(valueType, annotations, methodAnnotations);

    PartMap partMap = (PartMap) annotation;
    return new ParameterHandler.PartMap<>(valueConverter, partMap.encoding());

  } else if (annotation instanceof Body) {
    if (isFormEncoded || isMultipart) {
      throw parameterError(p,
          "@Body parameters cannot be used with form or multi-part encoding.");
    }
    if (gotBody) {
      throw parameterError(p, "Multiple @Body method annotations found.");
    }

    Converter<?, RequestBody> converter;
    try {
      converter = retrofit.requestBodyConverter(type, annotations, methodAnnotations);
    } catch (RuntimeException e) {
      // Wide exception range because factories are user code.
      throw parameterError(e, p, "Unable to create @Body converter for %s", type);
    }
    gotBody = true;
    return new ParameterHandler.Body<>(converter);
  }

  return null; // Not a Retrofit annotation.找不到该注解
}
```



从上面可以看出，改立参数注解的套路就是：先判断该注解的类型，然后使用策略模式分别调用ParameterHandler里对应的子类来处理

写到这里我已经晕了。晕晕乎乎好舒服



### @Header

#### 使用场景

有时候我们需要动态的设置请求header中的某个请求头的值，这个时候就可以使用@Header来修饰个参数。

最终都是讲header里的信息提取到Request里面

```java
static final class Header<T> extends ParameterHandler<T> {
  private final String name;
  private final Converter<T, String> valueConverter;

  Header(String name, Converter<T, String> valueConverter) {
    this.name = checkNotNull(name, "name == null");
    this.valueConverter = valueConverter;
  }

  @Override void apply(RequestBuilder builder, T value) throws IOException {
    if (value == null) return; // Skip null values.
    builder.addHeader(name, valueConverter.convert(value));
  }
}
```

```java
void addHeader(String name, String value) {
  if ("Content-Type".equalsIgnoreCase(name)) {
    MediaType type = MediaType.parse(value);
    if (type == null) {
      throw new IllegalArgumentException("Malformed content type: " + value);
    }
    contentType = type;
  } else {
    requestBuilder.addHeader(name, value);
  }
}
```

调用requestBuilder.addHeader()方法。

这个requestBuilder是OKHttp中Request的内部静态类Builder类的一个对象。

```
private final Request.Builder requestBuilder;
```

从中我们可以看出最后将@Header注释的参数的值解析后添加到Request对象中的Header信息里。

### @Path

#### 使用场景

有时候请求路径是不定的，即请求路径里的某个segment是变化的，也就是需要我们使用参数来动态的改变，这个时候我们就需要使用@Path 来修饰这个参数

```java
static final class Path<T> extends ParameterHandler<T> {
  private final String name; //参数名，占位符
  private final Converter<T, String> valueConverter;
  private final boolean encoded; //是否编码

  Path(String name, Converter<T, String> valueConverter, boolean encoded) {
    this.name = checkNotNull(name, "name == null");
    this.valueConverter = valueConverter;
    this.encoded = encoded;
  }

  @Override void apply(RequestBuilder builder, T value) throws IOException {
    if (value == null) {
      throw new IllegalArgumentException(
          "Path parameter \"" + name + "\" value must not be null.");
    }
    builder.addPathParam(name, valueConverter.convert(value), encoded);
  }
}
```



```java
void addPathParam(String name, String value, boolean encoded) {
  if (relativeUrl == null) {
    // The relative URL is cleared when the first query parameter is set.
    throw new AssertionError();
  }
   //将占位符”{name}”使用value替换
  relativeUrl = relativeUrl.replace("{" + name + "}", canonicalizeForPath(value, encoded));
}
```



### @Query

#### 使用场景

@Query用来修饰接口方法中的查询字段

```java
static final class Query<T> extends ParameterHandler<T> {
  private final String name;
  private final Converter<T, String> valueConverter;
  private final boolean encoded;

  Query(String name, Converter<T, String> valueConverter, boolean encoded) {
    this.name = checkNotNull(name, "name == null");
    this.valueConverter = valueConverter;
    this.encoded = encoded;
  }

  @Override void apply(RequestBuilder builder, T value) throws IOException {
    if (value == null) return; // Skip null values.
    builder.addQueryParam(name, valueConverter.convert(value), encoded);
  }
}
```



```Java
//将查询参数组合到相对路径上。
void addQueryParam(String name, String value, boolean encoded) {
  if (relativeUrl != null) {
    // Do a one-time combination of the built relative URL and the base URL.
    urlBuilder = baseUrl.newBuilder(relativeUrl);
    if (urlBuilder == null) {
      throw new IllegalArgumentException(
          "Malformed URL. Base: " + baseUrl + ", Relative: " + relativeUrl);
    }
    relativeUrl = null;
  }

  if (encoded) {
    urlBuilder.addEncodedQueryParameter(name, value);
  } else {
    urlBuilder.addQueryParameter(name, value);
  }
}
```



### @QueryMap

#### 使用场景

当接口中的一个 方法有比较多的查询字段时，全部定义到方法中时比较麻烦且容易出错，这个使用我们完全可以将所有的查询参数放到一个Map里面。

可想而知，其内部实现必定是遍历map ，然后像处理@Query参数一样调用addQueryParam()处理每个查询参数。

```
static final class FieldMap<T> extends ParameterHandler<Map<String, T>> {
  private final Converter<T, String> valueConverter;
  private final boolean encoded;

  FieldMap(Converter<T, String> valueConverter, boolean encoded) {
    this.valueConverter = valueConverter;
    this.encoded = encoded;
  }

  @Override void apply(RequestBuilder builder, Map<String, T> value) throws IOException {
    if (value == null) {
      throw new IllegalArgumentException("Field map was null.");
    }

    for (Map.Entry<String, T> entry : value.entrySet()) {
      String entryKey = entry.getKey();
      if (entryKey == null) {
        throw new IllegalArgumentException("Field map contained null key.");
      }
      T entryValue = entry.getValue();
      if (entryValue == null) {
        throw new IllegalArgumentException(
            "Field map contained null value for key '" + entryKey + "'.");
      }
      //果然不假
      builder.addFormField(entryKey, valueConverter.convert(entryValue), encoded);
    }
  }
}
```





### @Field

#### 使用场景

@Field注解一般用在表单参数的提交上

```java
static final class Field<T> extends ParameterHandler<T> {
  private final String name; //参数名字
  private final Converter<T, String> valueConverter; //参数值转换器
  private final boolean encoded; //是否编码

  Field(String name, Converter<T, String> valueConverter, boolean encoded) {
    this.name = checkNotNull(name, "name == null");
    this.valueConverter = valueConverter;
    this.encoded = encoded;
  }

  @Override void apply(RequestBuilder builder, T value) throws IOException {
    if (value == null) return; // Skip null values. 所以使用@Field修饰的字段，是不会上传到服务器的。
    //调用ResuestBuilder对象的具体想法来处理@Field修饰的表单字段
    builder.addFormField(name, valueConverter.convert(value), encoded);
  }
}
```

```
void addFormField(String name, String value, boolean encoded) {
//根据参数值是否被编码，调用不同的方法。formBuilder是OKHttp中的一个类。也是使用Builder模式创建的。
  if (encoded) {
    formBuilder.addEncoded(name, value);
  } else {
    formBuilder.add(name, value);
  }
}
```

### @FieldMap

@FieldMap 

#### 使用场景

假如表单参数有很多个，我们可以使用一个Map<String,String>来表示，然后使用@FieldMap注解来修饰该参数就行了。可想而知，如同@QueryMap一样，其内部实现肯定是遍历Map，然后像处理@Field参数一样调用

builder.addFormField(name, valueConverter.convert(value), encoded);

### @Body

#### 使用场景

在以下需要提交表单的请求里，我们可以使用@Field,@FieldMap,我们还可以使用@Body来修饰我们提交的表单数据，这个时候我们需要定义一个Bean类，Bean类的各个Field必须和表单字段的key一样

```java
static final class Body<T> extends ParameterHandler<T> {
  private final Converter<T, RequestBody> converter;

  Body(Converter<T, RequestBody> converter) {
    this.converter = converter;
  }

  @Override void apply(RequestBuilder builder, T value) {
    if (value == null) {
      throw new IllegalArgumentException("Body parameter value must not be null.");
    }
    RequestBody body;
    try {
      body = converter.convert(value);
    } catch (IOException e) {
      throw new RuntimeException("Unable to convert " + value + " to RequestBody", e);
    }
    builder.setBody(body);
  }
}
```

这里Retrofit并没有像@Field一样处理表单参数。仔细想想也对，因为凡是提交的表单数据都需要放到请求体里面，即使使用@Field，@FieldMap提交的数据，最终还是需要放到请求体里面。

### @Part

### @RawPart

### @RawPart

以上三个注解都是使用修饰上传文件的参数的，

### 结论

从对上面的分析可以知道，我们在提取使用注解修饰的参数后将值存放到RequestBuilder对象里。

这里又引入了RequestBuilder类

### RequestBuilder



```java
final class RequestBuilder {
  private static final char[] HEX_DIGITS =
      { '0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'A', 'B', 'C', 'D', 'E', 'F' };
  private static final String PATH_SEGMENT_ALWAYS_ENCODE_SET = " \"<>^`{}|\\?#";

  private final String method; //方法类型

  private final HttpUrl baseUrl;
  private String relativeUrl;
  private HttpUrl.Builder urlBuilder;

  private final Request.Builder requestBuilder;
  private MediaType contentType;

  private final boolean hasBody;
  private MultipartBody.Builder multipartBuilder;
  private FormBody.Builder formBuilder;
  private RequestBody body;

  RequestBuilder(String method, HttpUrl baseUrl, String relativeUrl, Headers headers,
      MediaType contentType, boolean hasBody, boolean isFormEncoded, boolean isMultipart) {
    this.method = method;
    this.baseUrl = baseUrl;
    this.relativeUrl = relativeUrl;
    this.requestBuilder = new Request.Builder();
    this.contentType = contentType;
    this.hasBody = hasBody;

    if (headers != null) {
      requestBuilder.headers(headers);
    }

    if (isFormEncoded) {
      // Will be set to 'body' in 'build'.
      formBuilder = new FormBody.Builder();
    } else if (isMultipart) {
      // Will be set to 'body' in 'build'.
      multipartBuilder = new MultipartBody.Builder();
      multipartBuilder.setType(MultipartBody.FORM);
    }
  }

  void setRelativeUrl(Object relativeUrl) {
    if (relativeUrl == null) throw new NullPointerException("@Url parameter is null.");
    this.relativeUrl = relativeUrl.toString();
  }

  void addHeader(String name, String value) {
    if ("Content-Type".equalsIgnoreCase(name)) {
      MediaType type = MediaType.parse(value);
      if (type == null) {
        throw new IllegalArgumentException("Malformed content type: " + value);
      }
      contentType = type;
    } else {
      requestBuilder.addHeader(name, value);
    }
  }

  void addPathParam(String name, String value, boolean encoded) {
    if (relativeUrl == null) {
      // The relative URL is cleared when the first query parameter is set.
      throw new AssertionError();
    }
    relativeUrl = relativeUrl.replace("{" + name + "}", canonicalizeForPath(value, encoded));
  }

  private static String canonicalizeForPath(String input, boolean alreadyEncoded) {
    int codePoint;
    for (int i = 0, limit = input.length(); i < limit; i += Character.charCount(codePoint)) {
      codePoint = input.codePointAt(i);
      if (codePoint < 0x20 || codePoint >= 0x7f
          || PATH_SEGMENT_ALWAYS_ENCODE_SET.indexOf(codePoint) != -1
          || (!alreadyEncoded && (codePoint == '/' || codePoint == '%'))) {
        // Slow path: the character at i requires encoding!
        Buffer out = new Buffer();
        out.writeUtf8(input, 0, i);
        canonicalizeForPath(out, input, i, limit, alreadyEncoded);
        return out.readUtf8();
      }
    }

    // Fast path: no characters required encoding.
    return input;
  }

  private static void canonicalizeForPath(Buffer out, String input, int pos, int limit,
      boolean alreadyEncoded) {
    Buffer utf8Buffer = null; // Lazily allocated.
    int codePoint;
    for (int i = pos; i < limit; i += Character.charCount(codePoint)) {
      codePoint = input.codePointAt(i);
      if (alreadyEncoded
          && (codePoint == '\t' || codePoint == '\n' || codePoint == '\f' || codePoint == '\r')) {
        // Skip this character.
      } else if (codePoint < 0x20 || codePoint >= 0x7f
          || PATH_SEGMENT_ALWAYS_ENCODE_SET.indexOf(codePoint) != -1
          || (!alreadyEncoded && (codePoint == '/' || codePoint == '%'))) {
        // Percent encode this character.
        if (utf8Buffer == null) {
          utf8Buffer = new Buffer();
        }
        utf8Buffer.writeUtf8CodePoint(codePoint);
        while (!utf8Buffer.exhausted()) {
          int b = utf8Buffer.readByte() & 0xff;
          out.writeByte('%');
          out.writeByte(HEX_DIGITS[(b >> 4) & 0xf]);
          out.writeByte(HEX_DIGITS[b & 0xf]);
        }
      } else {
        // This character doesn't need encoding. Just copy it over.
        out.writeUtf8CodePoint(codePoint);
      }
    }
  }

  void addQueryParam(String name, String value, boolean encoded) {
    if (relativeUrl != null) {
      // Do a one-time combination of the built relative URL and the base URL.
      urlBuilder = baseUrl.newBuilder(relativeUrl);
      if (urlBuilder == null) {
        throw new IllegalArgumentException(
            "Malformed URL. Base: " + baseUrl + ", Relative: " + relativeUrl);
      }
      relativeUrl = null;
    }

    if (encoded) {
      urlBuilder.addEncodedQueryParameter(name, value);
    } else {
      urlBuilder.addQueryParameter(name, value);
    }
  }

  void addFormField(String name, String value, boolean encoded) {
    if (encoded) {
      formBuilder.addEncoded(name, value);
    } else {
      formBuilder.add(name, value);
    }
  }

  void addPart(Headers headers, RequestBody body) {
    multipartBuilder.addPart(headers, body);
  }

  void addPart(MultipartBody.Part part) {
    multipartBuilder.addPart(part);
  }

  void setBody(RequestBody body) {
    this.body = body;
  }

  Request build() {
    HttpUrl url;
    HttpUrl.Builder urlBuilder = this.urlBuilder;
    if (urlBuilder != null) {
      url = urlBuilder.build();
    } else {
      // No query parameters triggered builder creation, just combine the relative URL and base URL.
      url = baseUrl.resolve(relativeUrl);
      if (url == null) {
        throw new IllegalArgumentException(
            "Malformed URL. Base: " + baseUrl + ", Relative: " + relativeUrl);
      }
    }

    RequestBody body = this.body;
    if (body == null) {
      // Try to pull from one of the builders.
      if (formBuilder != null) {
        body = formBuilder.build();
      } else if (multipartBuilder != null) {
        body = multipartBuilder.build();
      } else if (hasBody) {
        // Body is absent, make an empty body.
        body = RequestBody.create(null, new byte[0]);
      }
    }

    MediaType contentType = this.contentType;
    if (contentType != null) {
      if (body != null) {
        body = new ContentTypeOverridingRequestBody(body, contentType);
      } else {
        requestBuilder.addHeader("Content-Type", contentType.toString());
      }
    }

    return requestBuilder
        .url(url)
        .method(method, body)
        .build();
  }

  private static class ContentTypeOverridingRequestBody extends RequestBody {
    private final RequestBody delegate;
    private final MediaType contentType;

    ContentTypeOverridingRequestBody(RequestBody delegate, MediaType contentType) {
      this.delegate = delegate;
      this.contentType = contentType;
    }

    @Override public MediaType contentType() {
      return contentType;
    }

    @Override public long contentLength() throws IOException {
      return delegate.contentLength();
    }

    @Override public void writeTo(BufferedSink sink) throws IOException {
      delegate.writeTo(sink);
    }
  }
}
```



# OkHttpCall

```java
OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
```

在创建了ServiceMethod对象后，使用该ServiceMethod对象和其参数创建一个OKHttPCall对象

OkHttpCall 实现了Call接口，这个Call接口和OkHttp中的Call接口一样，毕竟一家公司嘛。

其实就是对OkHttpCall 做了一层包装。

最后方法的执行时通过调用

```
return serviceMethod.callAdapter.adapt(okHttpCall);
```

返回接口中方法定义的返回值。



