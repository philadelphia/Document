最近在做APP的通知模块时，产品要求点击通知后



1. 在用户登录的情况下直接进入到通知消息的详情页
2. 没有登录的情况下先进行登录，然后在跳转到消息的详情页



但是登录页面可能包含了N多个步骤，所以最好的方式是当用户没有登录的情况下使用startActivityForResult进行登录请求，把登录功能当做一个模块，

因为登录支持很多方式登录，用户在登录页面可能使用账密登录或者点击其他登录选项使用其他方式登录，

我们只需要关注LogInActivity就行了，

如果使用其他登录方式进行登录，当登录成功后跳转到LogInActivity

我们可以给跳转intent设置flag    

```
intent.setFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP|Intent.FLAG_ACTIVITY_CLEAR_TOP);
```

只有这两种启动标志组合使用，才会调用LogInActivity的onNewIntent方法。

而如果单独使用

```
intent.setFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP);
```

的话，会将LoginActivity提到栈顶并销毁其他Activity，但是并不会回调onNewIntent方法。

这样的话我们就相当于使用singleTask的方式启动了LogInActivity

这样中间登录步骤的页面就可以全部销毁了。

然后在LogInActivity的onNewIntent()方法中setResult(Activity.Result_OK）来告诉强求页登录成功了，

​	

```
@Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        setResult(RESULT_OK);
        finish();   
    }
```

​    没有登录的情况下复写onBackPressed()方法就行了，因为不论使用何种登录方式，只要进行到一半返回，最终没有完成登录返回的话一定会调用onBackPressed方法。因为只用成功才会自动finish LoginActivity 

```java
   @Override

​    public void onBackPressed() {

​        super.onBackPressed();

​        setResult(RESULT_CANCELED);

​    }
```

​    

其他功能也类似，比如认证相关的功能。



\## TaskAffinity

每个Activity都有taskAffinity属性，这个属性指出了它希望进入的Task。如果一个Activity没有显式的指明该Activity的taskAffinity，那么它的这个属性就等于Application指明的taskAffinity，如果Application也没有指明，那么该taskAffinity的值就等于包名。而TaskRecord也有自己的affinity属性，它的值等于它的根Activity的taskAffinity的值

当我们在manifest 中设置Activity的taskAffinity属性时 android:taskAffinity="com.example.test"

此时启动activity，然后执行

​        

​     adb shell dumpsys activity activities

 

```java
<!-- more -->

​    mResumedActivity=ActivityRecord{61b664c u0 com.example.installer/.ui.SingleTaskActivity t6360}

​            \* TaskRecord{4312a6a #6360 A=com.example.installer U=0 StackId=1 sz=2}

​            userId=0 effectiveUid=u0a1105 mCallingUid=u0a1105 mUserSetupComplete=true mCallingPackage=com.example.installer

​           ***\* affinity=com.example.installer\****

​            intent={act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.installer/.ui.WelcomeActivity}

​            realActivity=com.example.installer/.ui.WelcomeActivity

​            Activities=[ActivityRecord{301164 u0 com.example.installer/.ui.SingleTopActivity t6360}, ActivityRecord{61b664c u0 com.example.installer/.ui.SingleTaskActivity t6360}]

​            \* Hist #1: ActivityRecord{61b664c u0 com.example.installer/.ui.SingleTaskActivity t6360}

​                packageName=com.example.installer processName=com.example.installer

​                launchedFromUid=11105 launchedFromPackage=com.example.installer userId=0

​                app=ProcessRecord{bd78b5b 10653:com.example.installer/u0a1105}

​                Intent { cmp=com.example.installer/.ui.SingleTaskActivity }

​                frontOfTask=false task=TaskRecord{4312a6a #6360 A=com.example.installer U=0 StackId=1 sz=2}


```

<!-- more -->



<!-- more -->



```
                ***\*taskAffinity=com.example.test\****

​                realActivity=com.example.installer/.ui.SingleTaskActivity

​                baseDir=/data/app/com.example.installer-U815kFDGSxMa7fp67bPz2g==/base.apk

​                dataDir=/data/user/0/com.example.installer

​                realComponentName=com.example.installer/.ui.SingleTaskActivity

​            \* Hist #0: ActivityRecord{301164 u0 com.example.installer/.ui.SingleTopActivity t6360}

​                packageName=com.example.installer processName=com.example.installer

​                launchedFromUid=11105 launchedFromPackage=com.example.installer userId=0

​                app=ProcessRecord{bd78b5b 10653:com.example.installer/u0a1105}

​                Intent { cmp=com.example.installer/.ui.SingleTopActivity }

​                frontOfTask=true task=TaskRecord{4312a6a #6360 A=com.example.installer U=0 StackId=1 sz=2}

​                ***\* taskAffinity=com.example.installer \****

​                realActivity=com.example.installer/.ui.SingleTopActivity

​                baseDir=/data/app/com.example.installer-U815kFDGSxMa7fp67bPz2g==/base.apk

​                dataDir=/data/user/0/com.example.installer

​                realComponentName=com.example.installer/.ui.SingleTopActivity

​        TaskRecord{4312a6a #6360 A=com.example.installer U=0 StackId=1 sz=2}

​                Run #1: ActivityRecord{61b664c u0 com.example.installer/.ui.SingleTaskActivity t6360}

​                Run #0: ActivityRecord{301164 u0 com.example.installer/.ui.SingleTopActivity t6360}
```



此时系统只有一个TaskRecord,里边有两个Activity。也就是说即使Activity设置了不同的taskAffinity属性，但是launchMode不是singleTask或者singleInstance,此时系统依然会将activity放置在同一个TaskRecord中。

当对SingleTaskActivity的launchMode设置为singleTask后再次执行



​    adb shell dumpsys activity activities

​    

<!-- more -->

```
 

​    TaskRecord{ab8f7b2 #6363 A=com.example.test U=0 StackId=1 sz=1}

​                Run #1: ActivityRecord{e445992 u0 com.example.installer/.ui.SingleTaskActivity t6363}

​    TaskRecord{15ec280 #6362 A=com.example.installer U=0 StackId=1 sz=1}

​                Run #0: ActivityRecord{4c114aa u0 com.example.installer/.ui.SingleTopActivity t6362}
```



此时系统有2个TaskRecord里，每个TaskRecord 各自有一个Activity。每个Activity有着不同的TaskAffinity。



因为，可以得出一个结论就是：TaskAffinity 决定了Activity的分组，同一个的Task里有不同TaskAffinity的Activity.