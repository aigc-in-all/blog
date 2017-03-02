title: Android加载外部SO库
date: 2017-02-26 11:52:20
categories: Android
tags: 
- jni
- Android
comments: false
reward: false
---

如何加载外部的so文件

注意到我们一般加载so的方式是使用`System.loadLibrary(libname)`，同时我们发现`System`类还有一个`load`方法：

```java
/**
 * Loads a code file with the specified filename from the local file
 * system as a dynamic library. The filename
 * argument must be a complete path name.
 */
public static void load(String filename) {
    Runtime.getRuntime().load0(VMStack.getStackClass1(), filename);
}

/**
 * Loads the system library specified by the <code>libname</code>
 * argument. The manner in which a library name is mapped to the
 * actual system library is system dependent.
 */
public static void loadLibrary(String libname) {
    Runtime.getRuntime().loadLibrary0(VMStack.getCallingClassLoader(), libname);
}
```

先看看loadLibrary，这里调用了Runtime的loadLibrary，进去一看又是熟悉的ClassLoader，这也说明so库的使用就是一种动态加载的过程。

<!-- more -->

```java
synchronized void loadLibrary0(ClassLoader loader, String libname) {
    if (libname.indexOf((int)File.separatorChar) != -1) {
        throw new UnsatisfiedLinkError("Directory separator should not appear in library name: " + libname);
    }
    String libraryName = libname;
    if (loader != null) {
        String filename = loader.findLibrary(libraryName);
        if (filename == null) {
            // It's not necessarily true that the ClassLoader used
            // System.mapLibraryName, but the default setup does, and it's
            // misleading to say we didn't find "libMyLibrary.so" when we
            // actually searched for "liblibMyLibrary.so.so".
            throw new UnsatisfiedLinkError(loader + " couldn't find \"" +
                                           System.mapLibraryName(libraryName) + "\"");
        }
        String error = doLoad(filename, loader);
        if (error != null) {
            throw new UnsatisfiedLinkError(error);
        }
        return;
    }
    ...
```

看样子就像是通过库名找到文件，看看loader.findLibrary的实现：

```java
protected String findLibrary(String libname) {
    return null;
}
```

ClassLoader是个抽象类，它的大部分工作都在BaseDexClassLoader类中实现：

```java
public String findLibrary(String name) {
    throw new RuntimeException("Stub!");
}
```

不对呀，为什么这里直接抛了一个RuntimeException？

其实在看源码的时候经常看到这样的实现，这里有一个误区，这里我们看的源码只是Android SDK的源码，是给我们开发者参考的，Google不会把整个Android系统的源码放到这里来，整个项目非常大，ClassLoader类平时我们接触的少，所以它的具体实现的源码并没有打包到SDK里，如果想查看的话，我们需要到官方AOSP项目里面去看。如果本地没有下载源码，可以在线看[BaseDexClassLoader.java](https://android.googlesource.com/platform/libcore-snapshot/+/ics-mr1/dalvik/src/main/java/dalvik/system/BaseDexClassLoader.java)

*BaseDexClassLoader.java*
```java
@Override
public String findLibrary(String name) {
    return pathList.findLibrary(name);
}
```

*DexPathList.java*
```java
public String findLibrary(String libraryName) {
    String fileName = System.mapLibraryName(libraryName);
    for (File directory : nativeLibraryDirectories) {
        File file = new File(directory, fileName);
        if (file.exists() && file.isFile() && file.canRead()) {
            return file.getPath();
        }
    }
    return null;
}
```

根据传进来的libraryName，扫描apk内部的nativeLibrary目录，获取并返回内部so库文件的完整路径filename。再回到Runtime类，获取filename后调用`doLoad`方法：

再调用doLoad方法加载这个文件：

```java
private String doLoad(String name, ClassLoader loader) {
    synchronized (this) {
        return nativeLoad(name, loader, librarySearchPath);
    }
}
```

到这里就彻底清楚了，调用Native方法“nativeLoad”，通过完整的SO库路径filename，把目标SO库加载进来。

由此可以想到，如果使用`loadLibrary`方法，到最后还是要找到目标so库的完整路径，再把so库加载进来，那我们能不能直接加载一个外部的so呢？（`System.load(filename`有点像，来看看吧）

*Runtime.java*
```java
synchronized void load0(Class fromClass, String filename) {
    if (!(new File(filename).isAbsolute())) {
        throw new UnsatisfiedLinkError(
            "Expecting an absolute path of the library: " + filename);
    }
    if (filename == null) {
        throw new NullPointerException("filename == null");
    }
    String error = doLoad(filename, fromClass.getClassLoader());
    if (error != null) {
        throw new UnsatisfiedLinkError(error);
    }
}
```
这里直接就调用了`doLoad方法`，所以到此感觉`System.load`与`System.loadLibrary`的区别只是是否要查找so的文件路径。那试试呗~

这里我们尝试从asset里面手动加载一个so，看看是否能成功。

```java
private void customLoadLibrary() {
    String soName = "test.so";
    File soFile = getFileStreamPath(soName);
    if (soFile.exists()) {
        System.load(soFile.getAbsolutePath());
    } else {
        if (copyFromAssets(soName, soFile)) {
            System.load(soFile.getAbsolutePath());
        } else {
            // 无法拷贝!!!
            throw new IllegalStateException();
        }
    }
}

private boolean copyFromAssets(String name, File dest) {
    int len;
    byte[] buf = new byte[4096];
    InputStream in = null;
    FileOutputStream out = null;
    try {
        in = getAssets().open(name);
        out = new FileOutputStream(dest);
        while ((len = in.read(buf)) != -1) {
            out.write(buf, 0, len);
        }
    } catch (IOException e) {
        return false;
    } finally {
        DemoUtils.closeQuietly(in);
        DemoUtils.closeQuietly(out);
    }
    return true;
}
```

这里测试结果一切正常！

那能不能直接加载SDCard上的so呢，试试吧：

```java
D/dalvikvm: Trying to load lib /storage/emulated/0/libtest.so 0x422490a8
D/dalvikvm: Added shared lib /storage/emulated/0/libtest.so 0x422490a8
```
你没有看错，加载成功了！