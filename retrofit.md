# 	Retrofit 2.0 #

## 1.定义 ##

官方的Retrofit主页是这样描述它的
    
>A type-safe HTTP client for Android and Java

>用于Android和Java的一个类型安全(type-safe)的REST客户端

你可以使用注解去描述HTTP请求，同时Retrofit默认集成URL参数替换和查询参数.除此之外它还支持 Multipart请求和文件上传。

## 2.使用 ##

注意这个任务是网络任务，不要忘记给程序加入网络权限


    <uses-permission android:name="android.permission.INTERNET" />


build.gradle
 	
	dependencies { 
     // Retrofit &amp; OkHttp
    compile 'com.squareup.retrofit:retrofit:2.0.0-beta2'
 	}
     


retfofit 使用注解的方式定义API 

方法类型有：
> GET, POST, PUT, DELETE, HEAD

1	声明API

    public interface GitHubService {
      @GET("users/{user}/repos")
      Call<List<Repo>> listRepos(@Path("user") String user);
    }
2	初始化Retrofit

    Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();
    
    GitHubService service = retrofit.create(GitHubService.class);
    Call<List<Repo>> repos = service.listRepos("octocat");    
3	发起请求：

//同步请求

	try {
	  Response<List<Repo>> bodyResponse = call.execute();
	  String body = bodyResponse.body().string();//获取返回体的字符串
	  Log.i("wxl", "body=" + body);
	  } catch (IOException e) {
	  e.printStackTrace();
	  }


//异步请求


	call.enqueue(new Callback<List<Repo>>() {
	            @Override
	            public void onResponse(Response<List<Repo>> response) {
	                try {
	                    Log.i("wxl", "response=" + response.body().string());
	                } catch (IOException e) {
	                    e.printStackTrace();
	                }
	            }
	            @Override
	            public void onFailure(Throwable t) {
	                Log.i("wxl", "onFailure=" + t.getMessage());
	            }
	        });
4 取消请求	
	
	call.cancel();



所有的请求方法都是按照上述类型执行。
在方法声明时注解@GET，@POST,@PUT 分别对应方法的类型。

接口参数
>Path

ApiStores

	 /**
	 * Call<T> get();必须是这种形式,这是2.0之后的新形式
	 * 如果不需要转换成Json数据,可以用了ResponseBody;
	 * 你也可以使用Call<GsonBean> get();这样的话,需要添加Gson转换器
	 */
	public interface ApiStores {
	    @GET("adat/sk/{cityId}.html")
	    Call<ResponseBody> getWeather(@Path("cityId") String cityId);
	}

>Query

如果链接是http://ip.taobao.com/service/getIpInfo.php?ip=202.202.33.33

ApiStores

	public interface ApiStores {
		@GET("http://ip.taobao.com/service/getIpInfo.php")
		Call<ResponseBody> getWeather(@Query("ip") String ip);

	
复杂的查询参数也可以使用Map。


	@GET("group/{id}/users")
	Call<List<User>> groupList(@Path("id") int groupId, @QueryMap Map<String, String> options);

>Headers

你可以通过@Headers参数来设置静态headers

	@Headers("Cache-Control: max-age=640000")
	@GET("widget/list")
	Call<List<Widget>> widgetList();
	
	@Headers({
	    "Accept: application/vnd.github.v3.full+json",
	    "User-Agent: Retrofit-Sample-App"
	})
	@GET("users/{username}")
	Call<User> getUser(@Path("username") String username);

`Headers`不复写其他的`Headers`值。所有同名的的`Headers`参数都将包含在该请求中。
Request的Header可以通过使用`@Header`注解动态更新。必须给`@Header`提供一个相应的参数。如果该值为空，该`Header`就被忽略了。不然就会调用value的`toString()`方法获得value的值。


	@GET("user")
	Call<User> getUser(@Header("Authorization") String authorization)
每一个request都有的header可以通过设置`OkHttp interceptor`来统一添加。
	


>Body

这是针对POST方式，如果参数是json格式，如：

	{		
	    "apiInfo": {		
	        "apiName": "WuXiaolong",		
	        "apiKey": "666"		
	    }		
	}
ApiStores

	public interface ApiStores {
        @POST("client/shipper/getCarType")
        Call<ResponseBody> getCarType(@Body ApiInfo apiInfo);
	   }

这个`Object`将被`Retrofit`对象所指定的`Converter`所转换，如果没有指定转换器，只能使用`RequestBody`来定义参数。

建立Bean

	public class ApiInfo {
	       private ApiInfoBean apiInfo;
	       public ApiInfoBean getApiInfo() {
	           return apiInfo;
	       }
	       public void setApiInfo(ApiInfoBean apiInfo) {
	           this.apiInfo = apiInfo;
	       }
	       public class ApiInfoBean {
	           private String apiName;
	           private String apiKey;
	           //省略get和set方法
	       }
	   }
代码调用

	private void getCarType() {
       mRetrofit = new Retrofit.Builder()
               .baseUrl("http://WuXiaolong.me/")
               .addConverterFactory(GsonConverterFactory.create())
               .build();
       ApiStores apiStores = mRetrofit.create(ApiStores.class);
       ApiInfo apiInfo = new ApiInfo();
       ApiInfo.ApiInfoBean apiInfoBean = apiInfo.new ApiInfoBean();
       apiInfoBean.setApiKey("666");
       apiInfoBean.setApiName("WuXiaolong");
       apiInfo.setApiInfo(apiInfoBean);
       Call<ResponseBody> call = apiStores.getCarType(apiInfo);
       call.enqueue(new Callback<ResponseBody>() {
           @Override
           public void onResponse(Response<ResponseBody> response) {
               String body = null;//获取返回体的字符串
               try {
                   body = response.body().string();
               } catch (IOException e) {
                   e.printStackTrace();
               }
               Log.i("wxl", "get=" + body);
           }
           @Override
           public void onFailure( Throwable t) {
           }
       });
	   }


>Url encode 

POST or PUT Url encode 過的表單資料，用@FormUrlEncoded，使用@Field個別指定
	
	@POST("/articles/{article_id}/comments")
	@FormUrlEncoded
	Comment create(
    @Path("article_id") int articleId,
    @Field("rating") int rating,
    @Field("content") String content
	);
	
POST or PUT Url encode 過的表單資料，用@FormUrlEncoded，參數也可用@Body傳

	@POST("/articles/{article_id}/comments")
	@FormUrlEncoded
	Comment create(
	    @Path("article_id") int articleId,
	    @Body Comment comment
	);



>POST or PUT 用@Multipart來上傳檔案
	
	@PUT("/me")
	@Multipart
	Me update(
	    @Part("display_name") String displayName,
	    @Part("avatar") TypedFile avatar
	);


更详细的信息，请点击：[http://www.chenkaihua.com/2016/04/02/retrofit2-upload-multipart-files/](http://www.chenkaihua.com/2016/04/02/retrofit2-upload-multipart-files/)

更多更有效的 Converters
	
>Retrofit 1 里有一个 converter 的问题。多数人可能没遇到过，是库内部的一个问题。在 Retrofit 2 里，已经解决了这个问题，同时开始支持多种 Converter 并存。

>在之前，如果你遇到这种情况：一个 API 请求返回的结果需要通过 JSON 反序列化，另一个 API 请求需要通过 proto 反序列化，唯一的解决方>案就是将两个接口分离开声明。

	interface SomeProtoService {
	  @GET("/some/proto/endpoint")
	  Call<SomeProtoResponse> someProtoEndpoint();
	}
	
	interface SomeJsonService {
	  @GET("/some/json/endpoint")
	  Call<SomeJsonResponse> someJsonEndpoint();

>之所以搞得这么麻烦是因为一个 REST adapter 只能绑定一个 Converter 对象。我们费工夫去解决这个是因为：接口的声明是要语意化的。API 接口应该通过功能实现分组，比如： account 的接口，user 的接口，或者 Twitter 相关的接口。返回格式的差异不应该成为你分组时候的阻碍。

>现在，你可以把他们都放在一起了：

	interface SomeService {
	  @GET("/some/proto/endpoint")
	  Call<SomeProtoResponse> someProtoEndpoint();
	
	  @GET("/some/json/endpoint")
	  Call<SomeJsonResponse> someJsonEndpoint();
	}


>SomeProtoResponse —> Proto? Yes!

>原理很简单，其实就是对着每一个 converter 询问他们是否能够处理某种类型。我们问 proto 的 converter： “Hi, 你能处理 SomeProtoResponse 吗？”，然后它尽可能的去判断它是否可以处理这种类型。我们都知道：Protobuff 都是从一个名叫 message 或者 message lite 的类继承而来。所以，判断方法通常就是检查这个类是否继承自 message。
在面对 JSON 类型的时候，首先问 proto converter，proto converter 会发现这个不是继承子 Message 的，然后回复 no。紧接着移到下一个 JSON converter 上。JSON Converter 会回复说我可以！

>SomeJsonResponse —> Proto? No! —> JSON? Yes!

>因为 JSON 并没有什么继承上的约束。所以我们无法通过什么确切的条件来判断一个对象是否是 JSON 对象。以至于 JSON 的 converters 会对任何数据都回复说：我可以处理！这个一定要记住， JSON converter 一定要放在最后，不然会和你的预期不符。
>
>另一个要注意的是，现在已经不提供默认的 converter 了。如果不显性的声明一个可用的 Converter 的话，Retrofit 是会报错的：提醒你没有可用的 Converter。因为核心代码已经不依赖序列化相关的第三方库了，我们依然提供对 Converter 的支持，不过你需要自己引入这些依赖，同时显性的声明 Retrofit 需要用的 Converter 有哪些。
添加 converter 的顺序很重要。按照这个顺序，我们将依次询问每一个 converter 能否处理一个类型。我上面写的其实是错的。如果我们试图反序列化一个 proto 格式，它其实会被当做 JSON 来对待。这显然不是我们想要的。我们需要调整下顺序，因为我们先要检查 proto buffer 格式，然后才是 JSON。
>

	Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com")
    .addConverterFactory(ProtoConverterFactory.create())
    .addConverterFactory(GsonConverterFactory.create())
    .build();


>由于现在Retrofit开始依赖 OkHttp， 并没有 Http Client 层的抽象。现在是可以传递一个配置好的 OkHttp 实例的。比如：配置 interceptors, 或者一个 SSL socket 工厂类， 或者 timeouts 的具体数值。 （OkHttp 有默认的超时机制，如果你不需要自定义，实际上不>必进行任何设置，但是如果你想要去设置它们，下面是一个例子告诉你来怎么操作。）

	 OkHttpClient client = new OkHttpClient.Builder()
                .retryOnConnectionFailure(true)			//失败后是否重试
                .connectTimeout(15, TimeUnit.SECONDS)	//设置连接超时时间
                .addInterceptor(getHttpInterceptor())	//设置应用拦截器
                .build();
	
	Retrofit retrofit = new Retrofit.Builder()
	    .baseUrl("https://api.github.com")	//设置BaseURL
	    .client(client)						//设置client
	    .build();

## 3 Interceptor ##
	
>在使用Android retrofit+rxjava时，想获知网络请求的一些参数，方便调试，比如：请求地址、请求响应时间、请求响应消息体等内容，虽然部分可以通过每个接口进行获知，但是这样极其不方便，可以使用拦截器来做统一的操作。

添加拦截器：
可以针对OkHttpClient.Builder 添加拦截器
	
	HttpLoggingInterceptor interceptor = new HttpLoggingInterceptor();
	interceptor.setLevel(HttpLoggingInterceptor.Level.BODY);	//设置日志级别
 	
	OkHttpClient client = new OkHttpClient.Builder()
                .retryOnConnectionFailure(true)			//失败后是否重试
                .connectTimeout(15, TimeUnit.SECONDS)	//设置连接超时时间
                .addInterceptor(interceptor)	//设置应用拦截器
                .build();

## 4.Cache ##

>开启OKHttp缓存： 

先获取系统外部存储的路径，”xxx”可以自己命名，缓存文件就存在 Android/data/<包名>/cache/resposes。

	File httpCacheDirectory = new File(UIUtils.getContext().getExternalCacheDir(), "xxx"); 
	client.setCache(new Cache(httpCacheDirectory,10 * 1024 * 1024));


## 5:References ##



1. Retrofit github主页：[https://square.github.io/retrofit/](https://square.github.io/retrofit/ "retrofit github 主页")

2. Interceptor wiki:	[https://github.com/square/okhttp/wiki/Interceptors](https://github.com/square/okhttp/wiki/Interceptors "retrofit interceptor")


3. 用 Retrofit 2 简化 HTTP 请求	[ https://realm.io/cn/news/droidcon-jake-wharton-simple-http-retrofit-2/]( https://realm.io/cn/news/droidcon-jake-wharton-simple-http-retrofit-2/)

4. Http Caching:		[https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn "Http Cache")

5. 四种常见的 POST 提交数据方式		[https://imququ.com/post/four-ways-to-post-data-in-http.html](https://imququ.com/post/four-ways-to-post-data-in-http.html "四种常见的 POST 提交数据方式") 


6. Blog				 [ http://wuxiaolong.me/2016/01/15/retrofit/]( http://wuxiaolong.me/2016/01/15/retrofit/ "Blog")

6. URL wikipedia: 		[https://zh.wikipedia.org/wiki/%E7%BB%9F%E4%B8%80%E8%B5%84%E6%BA%90%E5%AE%9A%E4%BD%8D%E7%AC%A6](https://zh.wikipedia.org/wiki/%E7%BB%9F%E4%B8%80%E8%B5%84%E6%BA%90%E5%AE%9A%E4%BD%8D%E7%AC%A6 "URL wikipedia: ")