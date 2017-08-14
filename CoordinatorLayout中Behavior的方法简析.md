# CoordinatorLayout中Behavior的方法简析 #
8/14/2017 3:51:46 PM 

`Behavior` 是 `CoordinatorLayout` 里面的一个内部类，通过它我们可以与 `CoordinatorLayout` 的一个或者多个子View进行交互，包括 drag，swipes, flings等手势动作。
我们在继承Behavior类的时候需要复写其中几个方法来实现效果。Behavior中的方法主要分为两组，现将这几个方法介绍一下:

1. 第一组
	
	1. 决定child 依赖于哪一个 dependency
	    
			boolean	layoutDependsOn(CoordinatorLayout parent, V child, View dependency)
	
	2. 当 dependency View 改变的时候 child 要做出怎样的响应
		
			boolean	onDependentViewChanged(CoordinatorLayout parent, V child, View dependency)

2. 第二组

	调用顺序如下

	1.   当CoordinatorLayout的一个子节点试图开始嵌套滚动时调用
	     `Called when a descendant of the CoordinatorLayout attempts to initiate a nested scroll`
	
			public boolean onStartNestedScroll(CoordinatorLayout coordinatorLayout, 
												V child, View directTargetChild, View target, int nestedScrollAxes）
	
	
	2.	当CoordinatorLayout 开始接受嵌套滚动时调用 
		`Called when a nested scroll has been accepted by the CoordinatorLayout	`

		    public void onNestedScrollAccepted(CoordinatorLayout coordinatorLayout,
												 V child, View 	directTargetChild, 		View target, int nestedScrollAxes)
	


	3. 当一个嵌套滚动正在进行时，target消费任何的滚动距离之前调用，
	`Called when a nested scroll in progress is about to 		update, before the target has consumed any of the scrolled distance.`， 
	     	
			public void onNestedPreScroll(CoordinatorLayout coordinatorLayout, V child, View target,int dx,
						 int dy, 	int[] consumed)
	
	4. 当target试图滚动或者已经滚动时调用，
		Called when a nested scroll in progress has updated and the target has scrolled or attempted to scroll.  
			
			 public void onNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target,
               							 int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed)
	
	5. 开始依靠惯性滑动
		    
				public boolean onNestedPreFling(CoordinatorLayout coordinatorLayout, V child, View target,
                float velocityX, float velocityY)	
	
	6. 开始依靠惯性滑动
	  
			 public boolean onNestedFling(CoordinatorLayout coordinatorLayout, V child, View target,
            							    float velocityX, float velocityY, boolean consumed)
	
	7. 滑动停止   
		   
			public void onStopNestedScroll(CoordinatorLayout coordinatorLayout, V child, View target) 
   
                
	
	 
 


——————————————————————————————————————————————————————————————————————
	
## Reference： ##

1. [https://gdutxiaoxu.github.io/2016/12/08/%E8%87%AA%E5%AE%9A%E4%B9%89Behavior-%E2%80%94%E2%80%94-%E4%BB%BF%E7%9F%A5%E4%B9%8E%EF%BC%8CFloatActionButton%E9%9A%90%E8%97%8F%E4%B8%8E%E5%B1%95%E7%A4%BA/](https://gdutxiaoxu.github.io/2016/12/08/%E8%87%AA%E5%AE%9A%E4%B9%89Behavior-%E2%80%94%E2%80%94-%E4%BB%BF%E7%9F%A5%E4%B9%8E%EF%BC%8CFloatActionButton%E9%9A%90%E8%97%8F%E4%B8%8E%E5%B1%95%E7%A4%BA/)

 






