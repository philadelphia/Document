# Retrofit:声明 ApiService接口



我们在使用Retrofit的时候只需要把URL通过注解的形式写到APIService文件中就行了。

比如登录功能：

如果后台的成功返回格式为

```
{	
	code:0;
	Message:"login success"
}
```

失败的返回格式为

```
{	
	code:-1;
	Message:"login failed"
}
```

我们定义个一个Bean类LoginResult.

```
public class LoginResult{
	private int code;
	private String message;
	
	//get/set
}


```

## 不使用Rxjava

在方法的定义中，如果不适用Rxjava的话，

```
@Post
@FormUrlEncoded
Call<LogInResult> login(@Field("userName") String userName, @Field("password")String password);
```

在使用的时候们只需要使用apiService.login("dsf", "dsf"),就好了。该方法会返回一个Call<LoginResult> 对象

我们只需要或者异步执行

```
//同步执行
Response<LoginResult> response = call.execute();
//异步调用
call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {

            }
        });
```

然后我们在返回结果里面操作就行了，如果我们需要Response里面的元数据，比如header里面的信息的话，直接从Response取就好了。

因为Retrofit 使用DefaultCallAdapterFactory默认将OKHttp返回的Call对象(OkHttp对象)转换成了Retrofit的Call<T>对象了

## 使用Rxajva

### Observable<T>

但是我们在使用RxJava后，有时可能会这样声明接口

```
@Post
@FormUrlEncoded
Observable<LogInResult> login(@Field("userName") String userName, @Field("password")String password);
```

这样定义可以很方便的在返回中直接subScribe一个Action对象了

```
Observable<LoginResult> observable = model.login("dfs", "dfs");
observable.subscribe(new Action1<LoginResult>() {
            @Override
            public void call(LoginResult loginResult) {
								if (loginResult.getCode() == 0) {
                    //登录成功   
                } else {
                    //登录失败
                }
					
            }
        });
```

但是这样有一个问题就是会丢失Response的元数据，因为Response对象我们是没法访问的。因为Retrofit已经帮助我们做了转换，直接把我们接口方法里定义的数据类型转换好后返回给我们了。去掉了Response信息。

### Observable<Response<T>>

我们可以这么定义接口

```
@Post
@FormUrlEncoded
Observable<Response<LogInResult>> login(@Field("userName") String userName, @Field("password")String password);
```

这样我们就得到这样的返回结果。

```java
Observable<Response<LoginResult>> observable = model.login("dfs", "dfs");
observable.subscribe(new Action1<Response<LoginResult>>() {
            @Override
            public void call(Response<LoginResult> loginResultResponse) {
                if (loginResultResponse.code() == 200){
                    LoginResult body = loginResultResponse.body();
                }else {
                    //登录失败
                }
            }
        });
```

这样就能拿到Response信息了，然后根据Response code判断是否登录成功了。

但是这样有个问题是我们写接口会特备繁琐，每次都得使用Response<>泛型。一般我们都写作Observable<LoginResult> login();

其实我们一般情况下也不关注Response信息的。但是不排除特殊情况

比如这种情况

https://academy.realm.io/cn/posts/droidcon-jake-wharton-simple-http-retrofit-2/

这篇文章提到的分页加载的情况，其实这种情况也可以将下一个页面的URL放到Response body里返回。

这里只是给出了一种情景。

现实中解决方案很多种，只能折中了，其实我们没必要使用Response的Response code 来判断接口是否调用成功了。因为Retrofit都帮我们做过了。所以我们定义接口的时候只要使用Observable<Response>就好了。

我们在使用Retrofit + Rxjava 的时候一般都这么生成Retrofit client的

```
 Retrofit retrofit = new Retrofit.Builder()
                .client(okHttpClient)
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .baseUrl(baseUrl)
                .build();
```

这里RxJavaCallAdapterFactory生成的CallAdapter对象就帮我们做好了结果的转换

我们看一下RxJavaCallAdapterFactory的声明。

```
/**
 * A {@linkplain CallAdapter.Factory call adapter} which uses RxJava for creating observables.
 * <p>
 * Adding this class to {@link Retrofit} allows you to return {@link Observable} from service
 * methods.
 * <pre><code>
 * interface MyService {
 *   &#64;GET("user/me")
 *   Observable&lt;User&gt; getUser()
 * }
 * </code></pre>
 * There are three configurations supported for the {@code Observable} type parameter:
 有三种配置支持Observable的类型参数
 * <ul>
 * <li>Direct body (e.g., {@code Observable<User>}) calls {@code onNext} with the deserialized body
 * for 2XX responses and calls {@code onError} with {@link HttpException} for non-2XX responses and
 * {@link IOException} for network errors.</li>
 第一种：直接使用Observable<T>声明接口, 将会针对所有的2** Response code会调用onNext(T t)方法。所有的Response code 不是200的都调用OnError(Throwable t)方法抛出一个HttpException 异常，网络错误调用onError()方法，并抛出一个IO异常。
 
 * <li>Response wrapped body (e.g., {@code Observable<Response<User>>}) calls {@code onNext}
 * with a {@link Response} object for all HTTP responses and calls {@code onError} with
 * {@link IOException} for network errors</li>
 //第二种:如果使用Observable<Response<T>>声明接口，那么针对所有的Response将会调用onNext(Respones<T> response)方法，针对网络异常将调用OnError()方法，并抛出一个IO异常。
 * <li>Result wrapped body (e.g., {@code Observable<Result<User>>}) calls {@code onNext} with a
 * {@link Result} object for all HTTP responses and errors.</li>
 //第三种：如果使用Observable<Result<T>声明接口。所有的Response和Error将会调用onNext方法。
 注意：这个Result是Retrofit提供的。
 * </ul>
 */
 
```

## 结论

结论就是，如果我们没有对Response有特殊的需求的话，直接在接口声明处直接声明成Observable<T>是最方便最直观的。

这篇文章本应该和Retofit源码分析一起写的，奈何Retrofit的源码实在是太晦涩难懂了