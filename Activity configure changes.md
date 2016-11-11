Android 系统规定当系统的Configuration 改变时将会重启Activity。这是一种不太友好的体验，我们可以通过编辑Manifest文件来避免Activity的重启
>要声明由 Activity 处理配置变更，请在清单文件中编辑相应的 <activity> 元素，以包含 android:configChanges 属性以及代表要处理的配置的值。android:configChanges 属性的文档中列出了该属性的可能值（最常用的值包括 "orientation" 和 "keyboardHidden"，分别用于避免因屏幕方向和可用键盘改变而导致重启）。您可以在该属性中声明多个配置值，方法是用管道 | 字符分隔这些配置值。

例如，以下清单文件代码声明的 Activity 可同时处理屏幕方向变更和键盘可用性变更：

	<activity android:name=".MyActivity"
	 android:configChanges="orientation|keyboardHidden"	
	 android:label="@string/app_name">

现在，当其中一个配置发生变化时，MyActivity 不会重启。相反，MyActivity 会收到对 onConfigurationChanged() 的调用。向此方法传递 Configuration 对象指定新设备配置。您可以通过读取 Configuration 中的字段，确定新配置，然后通过更新界面中使用的资源进行适当的更改。调用此方法时，Activity 的 Resources 对象会相应地进行更新，以根据新配置返回资源，这样，您就能够在系统不重启 Activity 的情况下轻松重置 UI 的元素。

>注意：从 `Android 3.2（API 级别 13）`开始，当设备在纵向和横向之间切换时，“屏幕尺寸”也会发生变化。因此，在开发针对 API 级别 13 或更高版本（正如 minSdkVersion 和 targetSdkVersion 属性中所声明）的应用时，若要避免由于设备方向改变而导致运行时重启，则除了 "orientation" 值以外，您还必须添加 "screenSize" 值。 也就是说，您必须声明 android:configChanges="orientation|screenSize"。但是，如果您的应用面向 API 级别 12 或更低版本，则 Activity 始终会自行处理此配置变更（即便是在 Android 3.2 或更高版本的设备上运行，此配置变更也不会重启 Activity）。

例如，以下 onConfigurationChanged() 实现检查当前设备方向：

	@Override
	public void onConfigurationChanged(Configuration newConfig) {
  		  super.onConfigurationChanged(newConfig);
	
    	// Checks the orientation of the screen
    	if (newConfig.orientation == Configuration.ORIENTATION_LANDSCAPE) {
    	    Toast.makeText(this, "landscape", Toast.LENGTH_SHORT).show();
    	} else if (newConfig.orientation == Configuration.ORIENTATION_PORTRAIT){
    	    Toast.makeText(this, "portrait", Toast.LENGTH_SHORT).show();
    	}
	}

Configuration 对象代表所有当前配置，而不仅仅是已经变更的配置。大多数时候，您并不在意配置具体发生了哪些变更，而且您可以轻松地重新分配所有资源，为您正在处理的配置提供备用资源。 例如，由于 Resources 对象现已更新，因此您可以通过 setImageResource() 重置任何 ImageView，并且使用适合于新配置的资源（如提供资源中所述）。

请注意，Configuration 字段中的值是与 Configuration 类中的特定常量匹配的整型数。有关要对每个字段使用哪些常量的文档，请参阅 [Configuration](https://developer.android.com/guide/topics/manifest/activity-element.html#config) 参考文档中的相应字段。

请谨记：在声明由 Activity 处理配置变更时，您有责任重置要为其提供备用资源的所有元素。 如果您声明由 Activity 处理方向变更，而且有些图像应该在横向和纵向之间切换，则必须在 onConfigurationChanged() 期间将每个资源重新分配给每个元素。

如果无需基于这些配置变更更新应用，则可不用实现 onConfigurationChanged()。在这种情况下，仍将使用在配置变更之前用到的所有资源，只是您无需重启 Activity。 但是，应用应该始终能够在保持之前状态完好的情况下关闭和重启，因此您不得试图通过此方法来逃避在正常 Activity 生命周期期间保持您的应用状态。 这不仅仅是因为还存在其他一些无法禁止重启应用的配置变更，还因为有些事件必须由您处理，例如用户离开应用，而在用户返回应用之前该应用已被销毁。

如需了解有关您可以在 Activity 中处理哪些配置变更的详细信息，请参阅 `android:configChanges` 文档和 `Configuration` 类。

如果想要Activity不在系统语言发生改变时重试，需要给Activity设置`locale`属性
但是这在`Android 4.2` 之前的版本有效，`4.2`之后的版本就不行了。因为`Android 4.2`增加了一个layoutDirection属性，当改变语言设置后，该属性也会成newConfig中的一个mask位。所以ActivityManagerService(实际在ActivityStack)在决定是否重启Activity的时候总是判断为重启。
需要在android:configChanges 中同时添加locale和layoutDirection。


    android:configChanges="locale|layoutDirection"

所以一下设置应该会满足大部分情况下的Activity重启问题

    `android:configChanges="orientation|screenSize|locale|layoutDirection"`

>`orientation|screenSize`---方向改变
>
>`locale|layoutDirection`---系统语言改变


参考
   
1. Android Configuration changes:[https://developer.android.com/guide/topics/resources/runtime-changes.html](https://developer.android.com/guide/topics/resources/runtime-changes.html "Android Configuration changes")