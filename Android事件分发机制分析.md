# Android 系统的事件分发机制分析 #


Android中可以相应事件的控件有View和ViewGroup。其实ViewGroup也是一种View。只不过大部分情况下ViewGroup作为一种View容器来使用了。
View中有三个关键的方法
    
    1. public boolean dispatchTouchEvent(MotionEvent event)		//事件分发的方法
    2. public boolean onTouchEvent(MotionEvent event)			//事件处理的方法


ViewGroup中多了以下方法

	public boolean onInterceptTouchEvent(MotionEvent ev)		//事件拦截的方法

Android中touch事件的传递，绝对是先传递到ViewGroup，再传递到View的

当你点击了某个控件(touch事件发生)，首先会去调用该控件所在布局(ViewGroup)的dispatchTouchEvent方法，然后在布局的

dispatchTouchEvent方法中找到被点击的相应控件，再去调用该控件的dispatchTouchEvent方法。

    
	public boolean dispatchTouchEvent(MotionEvent ev) {  
    final int action = ev.getAction();  
    final float xf = ev.getX();  
    final float yf = ev.getY();  
    final float scrolledXFloat = xf + mScrollX;  
    final float scrolledYFloat = yf + mScrollY;  
    final Rect frame = mTempRect;  
    boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;  
    if (action == MotionEvent.ACTION_DOWN) {  
        if (mMotionTarget != null) {  
            mMotionTarget = null;  
        }  
        if (disallowIntercept || !onInterceptTouchEvent(ev)) {  
            ev.setAction(MotionEvent.ACTION_DOWN);  
            final int scrolledXInt = (int) scrolledXFloat;  
            final int scrolledYInt = (int) scrolledYFloat;  
            final View[] children = mChildren;  
            final int count = mChildrenCount;  
            for (int i = count - 1; i >= 0; i--) {  
                final View child = children[i];  
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE  
                        || child.getAnimation() != null) {  
                    child.getHitRect(frame);  
                    if (frame.contains(scrolledXInt, scrolledYInt)) {  
                        final float xc = scrolledXFloat - child.mLeft;  
                        final float yc = scrolledYFloat - child.mTop;  
                        ev.setLocation(xc, yc);  
                        child.mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;  
                        if (child.dispatchTouchEvent(ev))  {  
                            mMotionTarget = child;  
                            return true;  
                        }  
                    }  
                }  
            }  
        }  
    }  
    boolean isUpOrCancel = (action == MotionEvent.ACTION_UP) ||  
            (action == MotionEvent.ACTION_CANCEL);  
    if (isUpOrCancel) {  
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;  
    }  
    final View target = mMotionTarget;  
    if (target == null) {  
        ev.setLocation(xf, yf);  
        if ((mPrivateFlags & CANCEL_NEXT_UP_EVENT) != 0) {  
            ev.setAction(MotionEvent.ACTION_CANCEL);  
            mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;  
        }  
        return super.dispatchTouchEvent(ev);  
    }  
    if (!disallowIntercept && onInterceptTouchEvent(ev)) {  
        final float xc = scrolledXFloat - (float) target.mLeft;  
        final float yc = scrolledYFloat - (float) target.mTop;  
        mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;  
        ev.setAction(MotionEvent.ACTION_CANCEL);  
        ev.setLocation(xc, yc);  
        if (!target.dispatchTouchEvent(ev)) {  
        }  
        mMotionTarget = null;  
        return true;  
    }  
    if (isUpOrCancel) {  
        mMotionTarget = null;  
    }  
    final float xc = scrolledXFloat - (float) target.mLeft;  
    final float yc = scrolledYFloat - (float) target.mTop;  
    ev.setLocation(xc, yc);  
    if ((target.mPrivateFlags & CANCEL_NEXT_UP_EVENT) != 0) {  
        ev.setAction(MotionEvent.ACTION_CANCEL);  
        target.mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;  
        mMotionTarget = null;  
    }  
    return target.dispatchTouchEvent(ev);  
	}  

    
从上面的代码可以看出。如果viewgroup的disallowIntercept属性为false(默认是false，也可以通过requestDisallowInterceptTouchEvent方法对这个值进行修改)，或者onInterceptTouchEvent(ev)返回了true，那ViewGroup的
dispatchTouchEvent()方法也会返回false，事件就不会往下分发了，如果ViewGroup禁用了拦截机制(disallowIntercept=true)或者没有拦截事件(返回false)，事件就会分发给ViewGroup中的 子view了。然后子view开始处理事件了。

![](http://img.blog.csdn.net/20130629200236578?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc2lueXU4OTA4MDc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

从图中可以看出当ViewGroup分发事件的时候先调用自己的拦截方法，如果拦截方法返回了true，那么事件就不在向下传递给view了。然后ViewGroup就开始自己处理事件了。如果ViewGroup拦截方法返回了false，那么事件就会向下传递给view，然后view开始调用自己的分发方法，然后调用自己的事件处理方法。

## 总结 ##
如果View设置了`setOnTouchListener()`方法和`setOnLongClick()`和 `setOnClickListener()`方法，那默认情况下onTouch方法会先于
`onClick()`, `onLongClick()`方法调用。
  
	  button.setOnTouchListener(new View.OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                Log.i(TAG, "button------onTouch: " + event.getAction());
                return false;
            }
        });

	
	button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Log.i(TAG, "button --- onClick: ");
            }
        });


    09-15 14:08:15.184 27948-27948/com.example.drawerlayoutdemo I/MainActivity: button------onTouch: 0(Down)
	09-15 14:08:15.254 27948-27948/com.example.drawerlayoutdemo I/MainActivity: button------onTouch: 1(Up)
	09-15 14:08:15.273 27948-27948/com.example.drawerlayoutdemo I/MainActivity: button --- onClick:(onClick)

----
 view的`dispatchTouchEvent`方法（精简后代码）。

		public boolean dispatchTouchEvent(MotionEvent event) {
	    ...
	    boolean result = false;	// result 为返回值，主要作用是告诉调用者事件是否已经被消费。
	    if (onFilterTouchEventForSecurity(event)) {
	        ListenerInfo li = mListenerInfo;
	        /** 
	         * 如果设置了OnTouchListener，并且当前 View 可点击，就调用监听器的 onTouch 方法，
	         * 如果 onTouch 方法返回值为 true，就设置 result 为 true。
	         */
	        if (li != null && li.mOnTouchListener != null
	                && (mViewFlags & ENABLED_MASK) == ENABLED
	                && li.mOnTouchListener.onTouch(this, event)) {
	            result = true;
	        }
	      
	        /** 
	         * 如果 result 为 false，则调用自身的 onTouchEvent。
	         * 如果 onTouchEvent 返回值为 true，则设置 result 为 true。
	         */
	        if (!result && onTouchEvent(event)) {
	            result = true;
	        }
	    }
	    ...
	    return result;
	}	

可以看出如果设置了`onTouchListener`的话。优先调用`onTouchListener`的`onTouch()`方法。
但是如果在OnTouch方法里面返回true的话，就代表消费了事件，该事件不在往下分发，所以View的 `onClick`，`onLongClick` 方法就不会被调用了。因为`onClick()`and `onLongClick（）`是在view的`onTouchevent()`方法中调用的。
如果`onTouchListener`的`onTouch()`返回false的话。才会调用view的`onTouchEvent(event)`方法。

		public boolean onTouchEvent(MotionEvent event) {
	    ...
	    final int action = event.getAction();
	  	// 检查各种 clickable
	    if (((viewFlags & CLICKABLE) == CLICKABLE ||
	            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
	            (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
	        switch (action) {
	            case MotionEvent.ACTION_UP:
	                ...
	                removeLongPressCallback();  // 移除长按
	                ...
	                performClick();             // 检查单击
	                ...
	                break;
	            case MotionEvent.ACTION_DOWN:
	                ...
	                checkForLongClick(0);       // 检测长按
	                ...
	                break;
	            ...
	        }
	        return true;                        // ◀︎表示事件被消费
	    }
	    return false;
	}


从View的`onTouchevent（）`方法中可以看出。
如果view设置了`clickable`,`longClickable`，`contextClickable`属性中的任何一个。
或者设置了 `onClickListener`，`onLongClickListener`，`onContextClickListener`中的任何一个
那个该view就会消费事件。不然不消费事件。，

从`dispatchTouchEvent` 和`onTouchEvent()`方法可以看出。

1	事件的调度顺序为：
onTouchListener > onTouchEvent > onLongClickListener > onClickListener

2: 如果View为Disabled。则：`OntouchListener`里面的方法不会执行。但是会执行`OnTouchEvent(event)`方法，也就是说即使view是disabled的，依然可以相应onClick事件。

3： onTouchEvent方法中的ACTION_UP分支中触发onClick方法。并在这里移除长按监听。因为长按不需要处理ACTION_UP事件。