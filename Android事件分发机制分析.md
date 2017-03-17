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