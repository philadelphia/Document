# DialogFragment 使用

DialogFragment 有2种创建的方法

1：自己提供视图，使用系统创建的Dialog。

```
   @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        getDialog().getWindow().setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
        View view = inflater.inflate(R.layout.fragment_, container, false);
        unbinder = ButterKnife.bind(this, view);
        return view;
    }
```

2：自己创建Dialog

```java
@NonNull
public Dialog onCreateDialog(Bundle savedInstanceState) {
    return new Dialog(getActivity(), getTheme());
}

@Override
public LayoutInflater onGetLayoutInflater(Bundle savedInstanceState) {
    if (!mShowsDialog) {
        return super.onGetLayoutInflater(savedInstanceState);
    }
    mDialog = onCreateDialog(savedInstanceState);//创建Dialog
    if (mDialog != null) {
       setupDialog(mDialog, mStyle);
       return (LayoutInflater) mDialog.getContext()
             .getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    }
    return (LayoutInflater) mHost.getContext()
           .getSystemService(Context.LAYOUT_INFLATER_SERVICE);
} 
```

通过复写onCreateDialog方法，我们自己创建Dialog,推荐使用AlertDialog。这里我们自己创建，管理视图，

通过LayoutInfalter 来生成视图，然后将该View设置给dialog。



1和2的区别就是创建View的方式不同，如果我们两种方式都是想了，以第一种方式为主：

因为

```java
@Override
public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);

    if (!mShowsDialog) {
        return;
    }

    View view = getView();
    if (view != null) {
        if (view.getParent() != null) {
            throw new IllegalStateException(
                    "DialogFragment can not be attached to a container view");
        }
        mDialog.setContentView(view);
    }
    final Activity activity = getActivity();
    if (activity != null) {
        mDialog.setOwnerActivity(activity);
    }
    mDialog.setCancelable(mCancelable);
    mDialog.setOnCancelListener(this);
    mDialog.setOnDismissListener(this);
    if (savedInstanceState != null) {
        Bundle dialogState = savedInstanceState.getBundle(SAVED_DIALOG_STATE_TAG);
        if (dialogState != null) {
            mDialog.onRestoreInstanceState(dialogState);
        }
    }
}
```

在该方法中系统最终调用getView，如果我们实现了onCreateView方法自己创建了view，该方法取得该view 后设置给Dialog

修改窗口大小：

```
 @Override
    public void onStart() {
        super.onStart();
        WindowManager.LayoutParams attributes = getDialog().getWindow().getAttributes();
        int widthPixels = DisplayUtil.getDisplayMetrics().widthPixels;
        attributes.width = (int) (widthPixels * 0.75);//设为当前屏幕尺寸的0.75
        getDialog().getWindow().setAttributes(attributes);
    }
```



reference:

https://blog.csdn.net/zh_nan811/article/details/83015076)

[](https://blog.csdn.net/zh_nan811/article/details/83015076)

