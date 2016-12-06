# Android 中内部存储和外部存储分析 #

Android系统存储相关有这样几个概念，内存，内部存储，外部存储
	内存，
	英文中称作memory，

>内部存储 `InternalStorage`
>
>外部存储
 `	ExternalStorage`


Android 系统自身自带有存储，另外也可以通过 SD 卡来扩充存储空间。 前者空间较小，后者空间大，但后者不一定可用。 开发应用，处理本地数据存取时，可能会遇到这些问题：

>1. 需要判断 SD 卡是否可用: 占用过多机身内部存储，容易招致用户反感，优先将数据存放于 SD 卡;

>2. 应用数据存放路径，同其他应用应该保持一致，应用卸载时，清除数据:
 
>3. 标新立异在 SD 卡根目录建一个目录，招致用户反感

>4. 用户卸载应用后，残留目录或者数据在用户机器上，招致用户反感

>5. 需要判断两者的可用空间: SD 卡存在时，可用空间反而小于机身内部存储，这时应该选用机身存储;
 
>6. 数据安全性，本应用数据不愿意被其他应用读写;
 
>7. 图片缓存等，不应该被扫描加入到用户相册等媒体库中去。


那么究竟什么是内部存储什么是外部存储呢？
首先我们打开DDMS，有一个File Explorer，如下：
![](http://img.blog.csdn.net/20151211224232123?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
这里有三个文件夹需要我们重视，一个是data，一个是mnt，一个是storage，我们下面就详细说说这三个文件夹。
## 1.内部存储 ##
data文件夹就是我们常说的内部存储，当我们打开data文件夹之后（没有root的手机不能打开该文件夹），里边有两个文件夹值得我们关注，如下：
![](http://img.blog.csdn.net/20151211224609474?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

一个文件夹是app文件夹，还有一个文件夹就是data文件夹，app文件夹里存放着我们所有安装的app的apk文件，其实，当我们调试一个app的时候，可以看到控制台输出的内容，有一项是uploading .....就是上传我们的apk到这个文件夹，上传成功之后才开始安装。另一个重要的文件夹就是data文件夹了，这个文件夹里边都是一些包名，打开这些包名之后我们会看到这样的一些文件：

    1.data/data/包名/shared_prefs
    2.data/data/包名/databases
    3.data/data/包名/files
    4.data/data/包名/cache
如果打开过data文件，应该都知道这些文件夹是干什么用的，我们在使用sharedPreferenced的时候，将数据持久化存储于本地，其实就是存在这个文件中的xml文件里，我们App里边的数据库文件就存储于databases文件夹中，还有我们的普通数据存储在files中，缓存文件存储在cache文件夹中，存储在这里的文件我们都称之为内部存储。
## 2.外部存储 ##

外部存储才是我们平时操作最多的，外部存储一般就是我们上面看到的storage文件夹，当然也有可能是mnt文件夹，这个不同厂家有可能不一样。
一般来说，在storage文件夹中有一个sdcard文件夹，这个文件夹中的文件又分为两类，一类是公有目录，还有一类是私有目录，其中的公有目录有九大类，比如`DCIM`、`MUSIC`、`Ringtons`、`Pictures`、`Alarms`,`Movies`、`Documents`、`Notifications`等这种系统为我们创建的文件夹，私有目录就是`Android`这个文件夹，这个文件夹打开之后里边有两个文件夹：`data`，`media`，打开这个data文件夹，里边有许多包名组成的文件夹。每个包文件夹下又包含两个文件夹：`cache` 和`files`

## 3.操作存储空间 ##
首先，经过上面的分析，大家已经明白了，什么是内部存储，什么是外部存储，以及这两种存储方式分别存储在什么位置，一般来说，我们不会自己去操作内部存储空间，没有root权限的话，我们也没法操作内部存储空间，事实上内部存储主要是由系统来维护的。不过在代码中我们是可以访问到这个文件夹的。由于内部存储空间有限，在开发中我们一般都是操作外部存储空间，Google官方建议我们App的数据应该存储在外部存储的私有目录中该App的包名下，这样当用户卸载掉App之后，相关的数据会一并删除，如果你直接在/storage/sdcard目录下创建了一个应用的文件夹，那么当你删除应用的时候，这个文件夹就不会被删除。
经过以上的介绍，我们可以总结出下面一个表格：
![](http://img.blog.csdn.net/20151212091841385?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

一目了然，什么是内部存储，什么是外部存储。
如果按照路径的特征，我们又可以将文件存储的路径分为两大类，一类是路径中含有包名的，一类是路径中不含有包名的，含有包名的路径，因为和某个App有关，所以对这些文件夹的访问都是调用Context里边的方法，而不含有包名的路径，和某一个App无关，我们可以通过Environment中的方法来访问。如下图：
![](http://img.blog.csdn.net/20151212094211922?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

大家看到，有包名的路径我们都是调用Context中的方法来获得，没有包名的路径，我们直接调用Environment中的方法获得，那么其中有两个方法需要传入一个String类型的参数，这个参数我们使用了Environment中的常量，参数的意思是我们要访问这个路径下的哪个文件夹，比如getExternalFilesDir方法，我们看看它的源码：
	
	/** 
	 * 
	 * @param type The type of files directory to return.  May be null for 
	 * the root of the files directory or one of 
	 * the following Environment constants for a subdirectory: 
	 * {@link android.os.Environment#DIRECTORY_MUSIC}, 
	 * {@link android.os.Environment#DIRECTORY_PODCASTS}, 
	 * {@link android.os.Environment#DIRECTORY_RINGTONES}, 
	 * {@link android.os.Environment#DIRECTORY_ALARMS}, 
	 * {@link android.os.Environment#DIRECTORY_NOTIFICATIONS}, 
	 * {@link android.os.Environment#DIRECTORY_PICTURES}, or 
	 * {@link android.os.Environment#DIRECTORY_MOVIES}. 
	 * 
	 * @return The path of the directory holding application files 
	 * on external storage.  Returns null if external storage is not currently 
	 * mounted so it could not ensure the path exists; you will need to call 
	 * this method again when it is available. 
	 * 
	 * @see #getFilesDir 
	 * @see android.os.Environment#getExternalStoragePublicDirectory 
	 */  
	@Nullable  
	public abstract File getExternalFilesDir(@Nullable String type);  

它的注释非常多，我这里只列出其中一部分，我们看到，我们可以访问files文件夹下的Music文件夹、Movies文件夹等等好几种。
说到这里，我想大家对内部存储、外部存储该有了一个清晰的认识了吧。我们在开发中，**不建议往内部存储中写太多的数据，毕竟空间有限。外部存储在使用的时候最好能够将文件存放在私有目录下，这样有利于系统维护，也避免用户的反感**。
现在我们再来看看我们一开始提出的问题，当我们点击清除数据的时候清除的是哪里的数据呢？毫无疑问，当然是内部存储目录中相应的files和cache文件夹中的文件和外部存储中相应的files和cache文件夹中的文件，至于这些文件夹的路径我想你应该已经明白了。
好了，最后再送给大家一个文件操作工具类：
	
	public class SDCardHelper {  
  
	    // 判断SD卡是否被挂载  
	    public static boolean isSDCardMounted() {  
	        // return Environment.getExternalStorageState().equals("mounted");  
	        return Environment.getExternalStorageState().equals(  
	                Environment.MEDIA_MOUNTED);  
    }  
  
	    // 获取SD卡的根目录  
	    public static String getSDCardBaseDir() {  
	        if (isSDCardMounted()) {  
	            return Environment.getExternalStorageDirectory().getAbsolutePath();  
	        }  
	        return null;  
	    }  
	  
	    // 获取SD卡的完整空间大小，返回MB  
	    public static long getSDCardSize() {  
	        if (isSDCardMounted()) {  
	            StatFs fs = new StatFs(getSDCardBaseDir());  
	            long count = fs.getBlockCountLong();  
	            long size = fs.getBlockSizeLong();  
	            return count * size / 1024 / 1024;  
	        }  
	        return 0;  
	    }  
	  
	    // 获取SD卡的剩余空间大小  
	    public static long getSDCardFreeSize() {  
	        if (isSDCardMounted()) {  
	            StatFs fs = new StatFs(getSDCardBaseDir());  
	            long count = fs.getFreeBlocksLong();  
	            long size = fs.getBlockSizeLong();  
	            return count * size / 1024 / 1024;  
	        }  
	        return 0;  
	    }  
	  
	    // 获取SD卡的可用空间大小  
	    public static long getSDCardAvailableSize() {  
	        if (isSDCardMounted()) {  
	            StatFs fs = new StatFs(getSDCardBaseDir());  
	            long count = fs.getAvailableBlocksLong();  
	            long size = fs.getBlockSizeLong();  
	            return count * size / 1024 / 1024;  
	        }  
	        return 0;  
	    }  
	  
	    // 往SD卡的公有目录下保存文件  
	    public static boolean saveFileToSDCardPublicDir(byte[] data, String type,  
	            String fileName) {  
	        BufferedOutputStream bos = null;  
	        if (isSDCardMounted()) {  
	            File file = Environment.getExternalStoragePublicDirectory(type);  
	            try {  
	                bos = new BufferedOutputStream(new FileOutputStream(new File(  
	                        file, fileName)));  
	                bos.write(data);  
	                bos.flush();  
	                return true;  
	            } catch (Exception e) {  
	                e.printStackTrace();  
	            } finally {  
	                try {  
	                    bos.close();  
	                } catch (IOException e) {  
	                    // TODO Auto-generated catch block  
	                    e.printStackTrace();  
	                }  
	            }  
	        }  
	        return false;  
	    }  
	  
	    // 往SD卡的自定义目录下保存文件  
	    public static boolean saveFileToSDCardCustomDir(byte[] data, String dir,  
	            String fileName) {  
	        BufferedOutputStream bos = null;  
	        if (isSDCardMounted()) {  
	            File file = new File(getSDCardBaseDir() + File.separator + dir);  
	            if (!file.exists()) {  
	                file.mkdirs();// 递归创建自定义目录  
	            }  
	            try {  
	                bos = new BufferedOutputStream(new FileOutputStream(new File(  
	                        file, fileName)));  
	                bos.write(data);  
	                bos.flush();  
	                return true;  
	            } catch (Exception e) {  
	                e.printStackTrace();  
	            } finally {  
	                try {  
	                    bos.close();  
	                } catch (IOException e) {  
	                    // TODO Auto-generated catch block  
	                    e.printStackTrace();  
	                }  
	            }  
	        }  
	        return false;  
	    }  
	  
	    // 往SD卡的私有Files目录下保存文件  
	    public static boolean saveFileToSDCardPrivateFilesDir(byte[] data,  
	            String type, String fileName, Context context) {  
	        BufferedOutputStream bos = null;  
	        if (isSDCardMounted()) {  
	            File file = context.getExternalFilesDir(type);  
	            try {  
	                bos = new BufferedOutputStream(new FileOutputStream(new File(  
	                        file, fileName)));  
	                bos.write(data);  
	                bos.flush();  
	                return true;  
	            } catch (Exception e) {  
	                e.printStackTrace();  
	            } finally {  
	                try {  
	                    bos.close();  
	                } catch (IOException e) {  
	                    // TODO Auto-generated catch block  
	                    e.printStackTrace();  
	                }  
	            }  
	        }  
        return false;  
	    }  
	  
	    // 往SD卡的私有Cache目录下保存文件  
	    public static boolean saveFileToSDCardPrivateCacheDir(byte[] data,  
	            String fileName, Context context) {  
	        BufferedOutputStream bos = null;  
	        if (isSDCardMounted()) {  
	            File file = context.getExternalCacheDir();  
	            try {  
	                bos = new BufferedOutputStream(new FileOutputStream(new File(  
	                        file, fileName)));  
	                bos.write(data);  
	                bos.flush();  
	                return true;  
	            } catch (Exception e) {  
	                e.printStackTrace();  
	            } finally {  
	                try {  
	                    bos.close();  
	                } catch (IOException e) {  
	                    // TODO Auto-generated catch block  
	                    e.printStackTrace();  
	                }  
	            }  
	        }  
	        return false;  
	    }  
	  
	    // 保存bitmap图片到SDCard的私有Cache目录  
	    public static boolean saveBitmapToSDCardPrivateCacheDir(Bitmap bitmap,  
	            String fileName, Context context) {  
	        if (isSDCardMounted()) {  
	            BufferedOutputStream bos = null;  
	            // 获取私有的Cache缓存目录  
	            File file = context.getExternalCacheDir();  
	  
	            try {  
	                bos = new BufferedOutputStream(new FileOutputStream(new File(  
	                        file, fileName)));  
	                if (fileName != null  
	                        && (fileName.contains(".png") || fileName  
	                                .contains(".PNG"))) {  
	                    bitmap.compress(Bitmap.CompressFormat.PNG, 100, bos);  
	                } else {  
	                    bitmap.compress(Bitmap.CompressFormat.JPEG, 100, bos);  
	                }  
	                bos.flush();  
	            } catch (Exception e) {  
	                e.printStackTrace();  
	            } finally {  
	                if (bos != null) {  
	                    try {  
	                        bos.close();  
	                    } catch (IOException e) {  
	                        e.printStackTrace();  
	                    }  
	                }  
	            }  
	            return true;  
	        } else {  
	            return false;  
	        }  
	    }  
	  
	    // 从SD卡获取文件  
	    public static byte[] loadFileFromSDCard(String fileDir) {  
	        BufferedInputStream bis = null;  
	        ByteArrayOutputStream baos = new ByteArrayOutputStream();  
	  
	        try {  
	            bis = new BufferedInputStream(  
	                    new FileInputStream(new File(fileDir)));  
	            byte[] buffer = new byte[8 * 1024];  
	            int c = 0;  
	            while ((c = bis.read(buffer)) != -1) {  
	                baos.write(buffer, 0, c);  
	                baos.flush();  
	            }  
	            return baos.toByteArray();  
	        } catch (Exception e) {  
	            e.printStackTrace();  
	        } finally {  
	            try {  
	                baos.close();  
	                bis.close();  
	            } catch (IOException e) {  
	                e.printStackTrace();  
	            }  
	        }  
	        return null;  
	    }  
	  
	    // 从SDCard中寻找指定目录下的文件，返回Bitmap  
	    public Bitmap loadBitmapFromSDCard(String filePath) {  
	        byte[] data = loadFileFromSDCard(filePath);  
	        if (data != null) {  
	            Bitmap bm = BitmapFactory.decodeByteArray(data, 0, data.length);  
	            if (bm != null) {  
	                return bm;  
	            }  
	        }  
	        return null;  
	    }  
	  
	    // 获取SD卡公有目录的路径  
	    public static String getSDCardPublicDir(String type) {  
	        return Environment.getExternalStoragePublicDirectory(type).toString();  
	    }  
	  
	    // 获取SD卡私有Cache目录的路径  
	    public static String getSDCardPrivateCacheDir(Context context) {  
	        return context.getExternalCacheDir().getAbsolutePath();  
	    }  
	  
	    // 获取SD卡私有Files目录的路径  
	    public static String getSDCardPrivateFilesDir(Context context, String type) {  
	        return context.getExternalFilesDir(type).getAbsolutePath();  
	    }  
	  
	    public static boolean isFileExist(String filePath) {  
	        File file = new File(filePath);  
	        return file.isFile();  
	    }  
	  
	    // 从sdcard中删除文件  
	    public static boolean removeFileFromSDCard(String filePath) {  
	        File file = new File(filePath);  
	        if (file.exists()) {  
	            try {  
	                file.delete();  
	                return true;  
	            } catch (Exception e) {  
	                return false;  
	            }  
	        } else {  
	            return false;  
	        }  
	    }  
	}  



copy自：

[http://blog.csdn.net/u012702547/article/details/50269639](http://blog.csdn.net/u012702547/article/details/50269639)

[https://www.liaohuqiu.net/cn/posts/storage-in-android/](https://www.liaohuqiu.net/cn/posts/storage-in-android/)

Ths