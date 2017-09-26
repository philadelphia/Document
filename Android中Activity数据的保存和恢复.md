# Android中Activity数据的保存和恢复 #

在我们的APP使用的过程中，总有可能出现各种手滑、被压在后台、甚至突然被杀死的情况。所以对APP中一些临时数据或关键持久型数据，就需要我们使用正确的方式进行保存或恢复。

突发情况都有哪些？

因为本文讨论的是当一些突发情况的出现时，对数据的保存和恢复。所以现在总结一下突发情况应该都有哪些？

	1. 点击back键
	2. 点击锁屏键
	3. 点击home键
	4. 其他APP进入前台
	5. 启动了另一个Activity
	6. 屏幕方向旋转
	7. APP被Kill

##  ##


现在我们创建一个Activity并复写其中的关键方法：
	    
		package com.example.activitytest;
	
		import android.content.Intent;
		import android.os.Bundle;
		import android.support.v7.app.AppCompatActivity;
		import android.util.Log;
		import android.widget.Button;
		
		import butterknife.BindView;
		import butterknife.ButterKnife;
		import butterknife.OnClick;
		
		public class MainActivity extends AppCompatActivity {
		
	
	    @BindView(R.id.btn_secondActivity)
	    Button btnSecondActivity;
	    private static final String TAG = "MainActivity";
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        Log.i(TAG, "onCreate: ");
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	        ButterKnife.bind(this);
	    }
	
	    @Override
	    protected void onRestart() {
	        super.onRestart();
	        Log.i(TAG, "onRestart: ");
	    }
	
	    @Override
	    protected void onStart() {
	        super.onStart();
	        Log.i(TAG, "onStart: ");
	    }
	
	    @Override
	    protected void onResume() {
	        Log.i(TAG, "onResume: ");
	        super.onResume();
	    }
	
	    @Override
	    protected void onPause() {
	        super.onPause();
	        Log.i(TAG, "onPause: ");
	    }
	
	    @Override
	    protected void onStop() {
	        super.onStop();
	        Log.i(TAG, "onStop: ");
	    }
	
	    @Override
	    protected void onDestroy() {
	        super.onDestroy();
	        Log.i(TAG, "onDestroy: ");
	    }
	
	    @OnClick(R.id.btn_secondActivity)
	    public void onViewClicked() {
	        startActivity(new Intent(MainActivity.this, SecondActivity.class));
	    }
	
	
	    @Override
	    protected void onSaveInstanceState(Bundle outState) {
	        Log.i(TAG, "onSaveInstanceState: ");
	        super.onSaveInstanceState(outState);
	    }
	
	    @Override
	    protected void onRestoreInstanceState(Bundle savedInstanceState) {
	        super.onRestoreInstanceState(savedInstanceState);
	        Log.i(TAG, "onRestoreInstanceState: ");
	    }
	
	}

来看看各种情况下，Activity会调用哪些方法。


1.  点击back键

  
	    09-26 16:00:02.124 21636-21636/com.example.activitytest I/MainActivity: onPause: 
    	09-26 16:00:02.575 21636-21636/com.example.activitytest I/MainActivity: onStop: 
    	09-26 16:00:02.575 21636-21636/com.example.activitytest I/MainActivity: onDestroy: 

2. 点击锁屏键
		
		09-26 16:02:14.453 21636-21636/com.example.activitytest I/MainActivity: onPause: 
		09-26 16:02:14.497 21636-21636/com.example.activitytest I/MainActivity: onSaveInstanceState: 
		09-26 16:02:14.498 21636-21636/com.example.activitytest I/MainActivity: onStop:		

3. 点击home键
		
		09-26 16:02:54.239 21636-21636/com.example.activitytest I/MainActivity: onPause: 
		09-26 16:02:54.306 21636-21636/com.example.activitytest I/MainActivity: onSaveInstanceState: 
    	09-26 16:02:54.355 21636-21636/com.example.activitytest I/MainActivity: onStop: 
    
4. 其他APP进入前台
		
		09-26 16:04:02.658 21636-21636/com.example.activitytest I/MainActivity: onPause: 
		09-26 16:04:03.159 21636-21636/com.example.activitytest I/MainActivity: onSaveInstanceState: 
		09-26 16:04:03.159 21636-21636/com.example.activitytest I/MainActivity: onStop: 

5. 启动了另一个Activity

		09-26 16:04:02.658 21636-21636/com.example.activitytest I/MainActivity: onPause: 
		09-26 16:04:03.159 21636-21636/com.example.activitytest I/MainActivity: onSaveInstanceState: 
		09-26 16:04:03.159 21636-21636/com.example.activitytest I/MainActivity: onStop: 
6. 屏幕方向旋转

		09-26 16:04:51.199 21636-21636/com.example.activitytest I/MainActivity: onPause: 
		09-26 16:04:51.199 21636-21636/com.example.activitytest I/MainActivity: onSaveInstanceState: 
		09-26 16:04:51.201 21636-21636/com.example.activitytest I/MainActivity: onStop: 
		09-26 16:04:51.201 21636-21636/com.example.activitytest I/MainActivity: onDestroy: 
		09-26 16:04:51.220 21636-21636/com.example.activitytest I/MainActivity: onCreate: 
		09-26 16:04:51.250 21636-21636/com.example.activitytest I/MainActivity: onStart: 
		09-26 16:04:51.250 21636-21636/com.example.activitytest I/MainActivity: onRestoreInstanceState: 
		09-26 16:04:51.252 21636-21636/com.example.activitytest I/MainActivity: onResume: 

7. APP被Kill 
		
		09-26 16:05:17.544 21636-21636/com.example.activitytest I/MainActivity: onPause: 
		09-26 16:05:17.601 21636-21636/com.example.activitytest I/MainActivity: onSaveInstanceState: 
		09-26 16:05:17.611 21636-21636/com.example.activitytest I/MainActivity: onStop: 

当APP处在前台，能与用户交互的情况下，出现上述的突发事件时，只有点击back键，onSaveInstanceState方法不会调用。其余的情况下， 该方法一律都会调用，并且onPause方法是必然会调用的.
我们可以看一下onSaveInstanceState()方法的注释
	
	/**
     * Called to retrieve per-instance state from an activity before being killed
     * so that the state can be restored in {@link #onCreate} or
     * {@link #onRestoreInstanceState} (the {@link Bundle} populated by this method
     * will be passed to both).
     *
     * <p>This method is called before an activity may be killed so that when it
     * comes back some time in the future it can restore its state.  For example,
     * if activity B is launched in front of activity A, and at some point activity
     * A is killed to reclaim resources, activity A will have a chance to save the
     * current state of its user interface via this method so that when the user
     * returns to activity A, the state of the user interface can be restored
     * via {@link #onCreate} or {@link #onRestoreInstanceState}.
     *
     * <p>Do not confuse this method with activity lifecycle callbacks such as
     * {@link #onPause}, which is always called when an activity is being placed
     * in the background or on its way to destruction, or {@link #onStop} which
     * is called before destruction.  One example of when {@link #onPause} and
     * {@link #onStop} is called and not this method is when a user navigates back
     * from activity B to activity A: there is no need to call {@link #onSaveInstanceState}
     * on B because that particular instance will never be restored, so the
     * system avoids calling it.  An example when {@link #onPause} is called and
     * not {@link #onSaveInstanceState} is when activity B is launched in front of activity A:
     * the system may avoid calling {@link #onSaveInstanceState} on activity A if it isn't
     * killed during the lifetime of B since the state of the user interface of
     * A will stay intact.
     *
     * <p>The default implementation takes care of most of the UI per-instance
     * state for you by calling {@link android.view.View#onSaveInstanceState()} on each
     * view in the hierarchy that has an id, and by saving the id of the currently
     * focused view (all of which is restored by the default implementation of
     * {@link #onRestoreInstanceState}).  If you override this method to save additional
     * information not captured by each individual view, you will likely want to
     * call through to the default implementation, otherwise be prepared to save
     * all of the state of each view yourself.
     *
     * <p>If called, this method will occur before {@link #onStop}.  There are
     * no guarantees about whether it will occur before or after {@link #onPause}.
     *
     * @param outState Bundle in which to place your saved state.
     * 这个方法在Activity被Killed之前调用去保存Activity的每个实体的状态，所以状态可以在`onCreat
     * 或者`onRestoreInstanceState`被恢复。通过这个方法传递过去的`Bundle`
     * 这个方法在一个Activity被killed之前调用，所以当它将来恢复时可以恢复它的状态。比如：如果Activity B 
     * 在Acitivity A之前入站。如果在某一时刻A被销毁去释放资源，那么A将有机会调用该方法去保存它的用户UI的状态。
     * 因此，当用户返回A时。用户状态可以被恢复。
     * 不要将这个方法和Activity的生命周期回掉比如`onPause`，`onPause`在Activity被取代或者销毁时调用。`onStop`在被销毁之前
     * 调用。一个`onPause`和`onStop`被调用但是`onSaveInstanceState`没被调用的情况是当用户从B回退到A时。不需要调用B的
     * `onSaveInstanceState`
     * 一个`onPause`被调用但是`onSaveInstanceState`没被Blaunch 在A的上面时。系统避免调用A的 `onSaveInstanceState`
     * 如果A在B的生命周期内没有被销毁。所以A的用户状态将保持不变。
     * 可以通过给View设置ID。可以自动保存每个View的状态。
     * 如果这个方法被调用了。这个方法将在`onStop`之前调用。但是不保证和`onPause`的调用顺序。
     *

google工程师们对onSaveInstanceState如此设计就是让其完成对一些临时的、非永久数据存储并进行恢复。但是一些永久性质的数据，比如插入数据库的操作，我们应该在什么方法中对其进行保存呢？

	onPause
	onPuase的解释是：onPause(), onStop(), onDestroy() are "killable after" lifecycle methods. This indicates whether or not the system can kill the process hosting the activity at any time after the method returns, without executing another line of the activity's code. Because onPause() is the first of the three, once the activity is created, onPause() is the last method that's guaranteed to be called before the process can be killed—if the system must recover memory in an emergency, then onStop() and onDestroy() might not be called. Therefore, you should use onPause() to write crucial persistent data (such as user edits) to storage. However, you should be selective about what information must be retained during onPause(), because any blocking procedures in this method block the transition to the next activity and slow the user experience.

翻译过来就是：无论出现怎样的情况，比如程序突然死亡了，能保证的就是onPause方法是一定会调用的，而onStop和onDestory方法并不一定，所以这个特性使得onPause是持久化相关数据的最后的可靠时机。当然onPause方法不能做大量的操作，这会影响下一个Activity入栈

最后	我们来一句话总结一下：

##     临时数据使用onSaveInstanceState保存恢复，永久性数据使用onPause方法保存。 ##


参考：

1.[http://www.jianshu.com/p/6622434511f7](http://www.jianshu.com/p/6622434511f7 "http://www.jianshu.com/p/6622434511f7")

2.[http://blog.csdn.net/lonelyroamer/article/details/8927940](http://blog.csdn.net/lonelyroamer/article/details/8927940 "http://blog.csdn.net/lonelyroamer/article/details/8927940")
