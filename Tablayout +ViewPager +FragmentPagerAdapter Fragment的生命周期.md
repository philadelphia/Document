Tablayout +ViewPager +FragmentPagerAdapter Fragment的生命周期

实际的业务开发中经常遇到一个Activity嵌套多个Fragment的例子

实现上都是使用TabLayou + ViewPager + FragmentPagerAdapter来实现

现有Fragment AndroidBaseFragment 嵌套四个子Fragment，

依次调用AndroidBaseFragment的onAttach,onCreate方法

```
2020-07-08 11:56:55.788 20434-20434/com.example.myapplication I/AndroidBaseFragment: onAttach: 
2020-07-08 11:56:55.789 20434-20434/com.example.myapplication I/AndroidBaseFragment: onCreate:
2020-07-08 11:56:55.814 20434-20434/com.example.myapplication I/AndroidBaseFragment: onCreateView: 
2020-07-08 11:56:55.820 20434-20434/com.example.myapplication I/AndroidBaseFragment: onViewCreated:
```



然后回调子Fragment的声明周期函数

```
2020-07-08 11:56:55.865 20434-20434/com.example.myapplication I/ActivityFragment: onAttach: 
2020-07-08 11:56:55.865 20434-20434/com.example.myapplication I/ActivityFragment: onCreate: 
2020-07-08 11:56:55.866 20434-20434/com.example.myapplication I/ServiceFragment: onAttach: 
2020-07-08 11:56:55.866 20434-20434/com.example.myapplication I/ServiceFragment: onCreate: 
2020-07-08 11:56:55.867 20434-20434/com.example.myapplication I/BroadCastReceiverFragment: onAttach: 
2020-07-08 11:56:55.873 20434-20434/com.example.myapplication I/BroadCastReceiverFragment: onCreate: 
2020-07-08 11:56:55.873 20434-20434/com.example.myapplication I/ContentProviderFragment: onAttach: 
2020-07-08 11:56:55.873 20434-20434/com.example.myapplication I/ContentProviderFragment: onCreate: 

```

然后回调父Fragment的onActivityCreated方法

```
2020-07-08 11:56:55.874 20434-20434/com.example.myapplication I/AndroidBaseFragment: onActivityCreated:
```



最后回调子Fragment的onCreateView(). onViewCreated(),onActivityCreated()方法。

```
2020-07-08 11:56:55.874 20434-20434/com.example.myapplication I/ActivityFragment: onCreateView: 
2020-07-08 11:56:56.063 20434-20434/com.example.myapplication I/ActivityFragment: onViewCreated:  
2020-07-08 11:56:56.117 20434-20434/com.example.myapplication I/ActivityFragment: onActivityCreated: 
2020-07-08 11:56:56.117 20434-20434/com.example.myapplication I/ServiceFragment: onCreateView: 
2020-07-08 11:56:56.148 20434-20434/com.example.myapplication I/ServiceFragment: onViewCreated: 
2020-07-08 11:56:56.148 20434-20434/com.example.myapplication I/ServiceFragment: onActivityCreated: 
2020-07-08 11:56:56.149 20434-20434/com.example.myapplication I/BroadCastReceiverFragment: onCreateView: 
2020-07-08 11:56:56.176 20434-20434/com.example.myapplication I/BroadCastReceiverFragment: onViewCreated: 
2020-07-08 11:56:56.177 20434-20434/com.example.myapplication I/BroadCastReceiverFragment: onActivityCreated: 
2020-07-08 11:56:56.177 20434-20434/com.example.myapplication I/ContentProviderFragment: onCreateView: 
2020-07-08 11:56:56.187 20434-20434/com.example.myapplication I/ContentProviderFragment: onViewCreated: 
2020-07-08 11:56:56.187 20434-20434/com.example.myapplication I/ContentProviderFragment: onActivityCreated: 
```

接着回调父Fragment的onStart()方法和子Fragment的onStart()方法。

```
2020-07-08 11:56:56.187 20434-20434/com.example.myapplication I/AndroidBaseFragment: onStart: 
2020-07-08 11:56:56.187 20434-20434/com.example.myapplication I/ActivityFragment: onStart: 
2020-07-08 11:56:56.187 20434-20434/com.example.myapplication I/ServiceFragment: onStart: 
2020-07-08 11:56:56.187 20434-20434/com.example.myapplication I/BroadCastReceiverFragment: onStart: 
2020-07-08 11:56:56.188 20434-20434/com.example.myapplication I/ContentProviderFragment: onStart: 
```

接着回调父Fragment的onResume()方法和子Fragment的onResume()方法。

```
2020-07-08 11:56:56.188 20434-20434/com.example.myapplication I/AndroidBaseFragment: onResume: 
2020-07-08 11:56:56.188 20434-20434/com.example.myapplication I/ActivityFragment: onResume: 
2020-07-08 11:56:56.188 20434-20434/com.example.myapplication I/ServiceFragment: onResume: 
2020-07-08 11:56:56.188 20434-20434/com.example.myapplication I/BroadCastReceiverFragment: onResume: 
2020-07-08 11:56:56.190 20434-20434/com.example.myapplication I/ContentProviderFragment: onResume: 
```

当切换到第二个Fragment时会回调上一个Fragment和下一个Fragment的setUserVisibleHint方法。

```
2020-07-08 13:43:08.028 20434-20434/com.example.myapplication I/ActivityFragment: setUserVisibleHint: false
2020-07-08 13:43:08.029 20434-20434/com.example.myapplication I/ServiceFragment: setUserVisibleHint: true
```

