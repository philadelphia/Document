LayoutInflater 使用说明

LayoutInflater主要是用于加载布局。
加载布局的任务通常都是在Activity中调用setContentView()方法来完成的。其实setContentView()方法的内部也是使用LayoutInflater来加载布局的，只不过这部分源码是internal的，不太容易查看到。

先来看一下LayoutInflater的基本用法吧，它的用法非常简单，首先需要获取到LayoutInflater的实例，有两种方法可以获取到，第一种写法如下：
    
    LayoutInflater layoutInflater = LayoutInflater.from(context);  
当然，还有另外一种写法也可以完成同样的效果：

	LayoutInflater layoutInflater = (LayoutInflater) context  
        .getSystemService(Context.LAYOUT_INFLATER_SERVICE);  
其实第一种就是第二种的简单写法，只是Android给我们做了一下封装而已。代码如下：
    
     public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }

得到了LayoutInflater的实例之后就可以调用它的inflate()方法来加载布局了，如下所示：
		
	layoutInflater.inflate(resourceId, root);  





inflate()方法一般接收两个参数，第一个参数就是要加载的布局资源文件id，第二个参数是指给该布局的外部再嵌套一层父布局，如果不需要就直接传null。这样就成功成功创建了一个布局的实例，之后再将它添加到指定的位置就可以显示出来了。
	
其实它是调用一个三个参数的重载方法来实现的。
代码实现：
	
	public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
    }


该三个参数的方法的实现为：
	 


public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
   return inflate(resource, root, root != null);
 }



也就是说不管你是使用的哪个inflate()方法的重载，最终都会辗转调用到LayoutInflater的如下代码中

三个参数的方法的实现为：
	
	public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();

                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }

			//如果开始IDE标签名是Merge，则会将布局文件中的view依次加入到root视图中。
                if (TAG_MERGE.equals(name)) {
					//如果开始IDE标签名是Merge,如果attachToRoot 为false，会报错
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

				//遍历parser。将view依次加入到root中、
                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
					//如果开始标签不是merge。则temp就是该布局的根视图。
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                final InflateException ie = new InflateException(e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } catch (Exception e) {
                final InflateException ie = new InflateException(parser.getPositionDescription()
                        + ": " + e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;

                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }

            return result;
        }
    }


```

	 第二个参数和第三个参数有四种组合情况
        if(root == null && ！attachToRoot){
            返回id里的view本身
        }

        if(root != null && attachToRoot){
            返回root本身，此时root已经将id里的控件添加了。
        此时如果在使用root.addview(view) 就会报错。view此时已经有parent、因为view本身就是root自己。root自己addview自身会报错
        }else if (root != null && !attachToRoot){
            返回返回id里的view本身，此时id里的控件已经设置了布局参数
        }

	1. 如果root为null，attachToRoot将失去作用，设置任何值都没有意义。
	2. 如果root不为null，attachToRoot设为true，则会给加载的布局文件的指定一个父布局，即root。
	3. 如果root不为null，attachToRoot设为false，则会将布局文件最外层的所有layout属性进行设置，当该view被添加到父view当中时，这些layout属性会自动生效。
	4. 在不设置attachToRoot参数的情况下，如果root不为null，attachToRoot参数默认为true。

```



Reference:	[http://blog.csdn.net/guolin_blog/article/details/12921889](http://blog.csdn.net/guolin_blog/article/details/12921889)
