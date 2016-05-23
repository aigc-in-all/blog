---
layout: post
title: "签名你的应用程序"
date: 2014-05-24 22:23:41 +0800
comments: true
tags: Android
categories: Android
---

签名你的应用程序
=========

Android系统要求所有的应用程序都需要被持有开发者信息的key签名。Android系统使用证书作为识别应用程序的作者和建立应用程序之间的信任关系的一种手段。该证书不能控制应用程序是否可以安装。该证书不需要由证书颁发机构进行签名：它是完全允许和典型的，为Android应用程序使用自签名证书。

要了解有关Android应用程序签名的要点是：

  - 所有应用程序都必须签名。系统不允许安装未签名的应用程序，包括模拟器和真实设备。
  - 测试和调试应用程序，编译工具会使用一个默认的key去签名你的应用程序，这个key是由Android SDK编译工具创建的。
  - 当你准备向你的最终用户发布应用程序时，你必须使用一个适当的私有key去签名你的应用程序。不能发布由默认key签名的应用程序。
  - 你可以使用自签名证书去签名你的应用程序。不需要任何证书颁发机构。
  - 系统只有在安装应用程序的时候检测签名证书的有效期。如果应用程序安装后应用程序的签名证书过期，应用程序将继续正常工作。
  - 你可以使用标准工具（keytool和jarsigner）去生成密钥和签名应用程序的`.apk`文件。
  - 当你签名完后准备发布，我们建议你使用`zipalign`工具来优化最终的APK包。
  
Android系统不会安装或运行未签名成功的应用程序，这适应于所有运行的Android系统，无论是模拟器还是实际设备。出于这个原因，在真机或者模拟器运行或者调试之前，你必须为你的应用程序设置签名。

<!-- more -->

签名过程
---------
----------

应用程序不同的编译模式对于不同的签名过程。有两种编译模式：debug和release，当你开发或者测试的时候使用debug模式，当你想编译一个直接给用户的release版本或者发布到市场的应用程序使用release模式，例如Google Play。

当你在debug模式下的时候，Android SDK编译工具使用Keytool工具（在JDK里面）创建一个debug key.因为这个debug key是SDK编译工具创建的，所以它知道这个debug key的别名(alias)和密码，当你每次在debug模式下编译你的应用程序，这个编译工具使用这个debug key一起的Jarsigner工具（也在JDK里面）去签名你的应用程序的 `.apk` 文件。因为编译工具知道这个别名和密码，所以每次在编译的时候不需要提示你输入这个debug key的别名和密码。

当你在release模式下的时候，你需要使用你自己的私有key去签名你的应用程序。如果你没有一个私有的key，你可以使用Keytool工具创建一个，当你在release模式下编译你的应用程序，编译工具使用这个私有key和Jarsigner去签名你的应用程序的 `.apk` 文件，因为这个证书和私有key是你自己的，你必须向这个keystore和key alias提供密码。

当你使用Eclipse和ADT插件来开发应用程序的话，这个 `debug` 模式下的签名过程自动发生当你运行或者调试你的应用程序的时候。当你使用Ant编译脚本（debug）选项的时候，debug签名也会自动发生。你可以使用Exlipse Exprort Wizard或者修改Ant编译脚本（release）去编译应用程序让 `release` 签名自动发生。


签名策略
---------
----------

应用程序签名的某些方面可能会影响你如何对待你所开发的应用程序，特别是如果你计划推出多个应用程序。

通常，在整个应用程序的预期寿命里面，推荐所有的开发者都使用相同的证书去签名所有的应用程序，有几个原因，你应该这样做：

  - 应用程序升级。当你发布应用更新，如果你想让用户无疑地更新到新版本，你必须使用相同的证书签名更新程序。当系统安装一个更新的应用程序，它会比较新版本的证书和旧版本的证书（如果有的话），如果两个证书完全匹配，包括证书数据和序列，系统允许安装更新。如果你使用一个不匹配的证书签名，你还必须指定一个不同的包名称的应用程序，相当于用户安装一个新的应用程序。
  - 应用程序模块化。Android系统允许应用相同签名的应用程序运行在同一个进程里，如果应用程序请求，从而系统会将其视为一个单一的应用程序。通过这种方式，你可以在模块中布置你的应用程序，用户可以根据需要单独更新某一个模块。
  - 代通过权限实现代码/数据共享。Android系统提供了基于签名的权限执行，这样应用程序可以公开功能到与指定证书签名的另一个应用程序。通过订阅多个应用程序使用相同的证书，并使用基于签名的权限检查，你的应用程序可以在一个安全的方式共享代码和数据。


决定你的签名策略的另一个重要的考虑是怎样设置你将要签名的应用程序的key的有效期。

  - 如果你打算支持升级一个单一的应用程序，你应该确保你的key有超过程序的预期寿命的有效期。建议设置25年的有效期。当你的key的有效期满后，用户将无法无缝地升级到新你的应用程序的新版本。
  - 如果你计划使用相同的key布置多个应用程序，你应该确保你的key的有效期超过所有版本的所有应用程序，包括未来可能被增加到套件中来的应用程序。
  - 如果你计划把你的应用程序发布到Google Play,你所使用的签名key的有效期必须支持到2033年10月22日之后。Google Play强制执行这一要求，确保用户 可以无缝地升级到应用程序的新版本是可用的。

当你设计你的应用程序时，请保持以上几点，确保使用合适的证书签名你的应用程序。


基本设置
---------
----------

在开始之前，确保Keytook工具和Jarsigner工具对于SDK编译工具可用。这两个工具都在JDK中，在大多数情况下，你可以告诉SDK编译工具如何通过设置 `JAVA_HOME` 环境变量来查找这些工具，以便它引用一个合适的JDK。或者你也可以添加JDK版本的Keytool和Jarsigner到你的 `PATH` 环境变量。

如果你是在一个基于Linux的平台上开发，系统原本附带有一个GNU编译器的Java，请确保系统使用的是JDK版本的Keytool，如果Keytool已经在PATH里面，它可能是一个指向 `user/bin/keytool` 的一个符号链接，在这种情况下，确保它指向JDK下面的keytool


Debug模式下签名
---------
----------

Android编译工具提供一个debug签名模式，使得你开发或者调试应用程序更方便。同时还能满足Android系统需要签署过的应用程序的需求。当你使用debug模式编译app，SDK工具调用Keytool来自动创建一个debug的keystore和key，这个debug key将自动签名你的app，因此你不需要用你自己的key去签名你的app。
SDK使用确定的names和password创建debug模式的keystore/key：

  - Keystore name:"debug.keystore"
  - Keystore password:"android"
  - Key alias:"androiddebugkey"
  - Key password:"android"
  - CN:"CN=Android Debug,O=Android,C=US"

如果需要，你可以更改debug keystore/key 的位置或者名称或者提供一个自定义的debug keystore/key来使用。然而，任何自定义的 debug keystore/key都必须使用默认的keystore/key 名字和密码（如上所述）。（例如在Eclipse/ADT，**Windows > Preferences > Android > Build**）

> **警告**：无法发布使用debug证书签名的应用程序


### Eclipse用户 ###

如果你使用Eclipse/ADT，debug签名模式是默认启用的。当你运行或者调试应用程序，ADT会使用debug证书签名 `.apk` 文件，并且对package应用 `zipalign` ，然后安装在被选择的模拟器或者设备上。不需要自己做什么额外的操作，ADT可以访问Keytool.

### Ant用户 ###

如果你正在使用Ant来构建你的 `.apk` 文件，通过Ant命令使用 `debug` 选项是可用的（假使你使用一个 `build.xml` 文件生成android工具）。当你运行你的Ant调试编译你的应用程序，build脚本生成一个keystore/key并且签名的apk文件给你，然后该脚本还会使用 `zipalign` 工具apigns这个apk文件。不需要你自己做任何操作。阅读[命令行编译和运行程序][1]了解更多信息

### Debug证书的有效期 ###

在debug模式会使用自签名的证书去签名你的应用程序，这个自签名的证书将会有365天的有效期（从创建的时候算起）。
当你的证书过期，你会得到一个编译错误，如果使用Ant编译，这个错误大概长这样：

```
debug:
[echo] Packaging bin/samples-debug.apk, and signing it with a debug key...
[exec] Debug Certificate expired on 8/4/08 3:43 PM
```

在Eclipse/ADT，你将在Android控制台看到相似的错误信息。

删除 `debug.keystore` 文件就可以解决此问题。如果是OS X或者Linux系统，默认存储在 `~/.android`目录下，如果是Windows XP系统，默认存储在 `C:\Documents and Settings\<user>\.android/` 目录下，如果是Windows Vista或者Windows 7，默认存储在 `C:\Users\<user>\.android/` 目录下。

在你下次编译的时候，编译工具会生成一个新的keystore和debug key.

需要注意的是，如果你使用的开发机器使用的是一个非公历的语言环境，构建工具可能会错误地生成一个已经过期的debug证书，所以你在编译的时候会出现错误。有关解决方法的信息，参阅[I can't compile my app because the build tools generated an expired debug certificate][2]

Release模式下签名
---------
----------

当你的应用程序已经准备好向用户发布，你必须：

  1. 获取一个合适的私钥
  2. 在release模式下编译你的应用程序
  3. 用私钥签名你的应用程序
  4. Align最终的Apk包

如果你使用Eclipse开发，你可以使用Export向导完成编译、签名和Align程序，Export向导过程中甚至可以让你生成一个新的keystore和private key.所以，如果你使用Eclipse，你可以跳至[Compile and sign with Eclipse ADT][3]

### 1. 获取一个合适的私钥 ###

在准备签名你的应用程序的时候，你必须首先确定你有一个合适的私钥。一个合适的私钥是一个：

  - 是你自己拥有的
  - 代表个人、企业或组织的实体去标记你的应用程序
  - 有效期超过应用程序或程序套件所期望的生命周期。有效期超过25年是被推荐的。
  - 如果你计划在Google Play上发布一个或多个应用程序，注意有效期应该是在2033年10月22日之后，如果有效期在这之前，你将不能上传到Google Play.
  - 不能是构建工具生的debug key

这个key可以是自签名的，如果你没有一个适合的key, 你必须使用Keytool生成一个，确保你有一个可用的Keytool，参考上面的基本设置。

要使用Keytool生成一个自签名key，使用keytool命令，并通过下面列出的选项（如果需要）。

> **警告**：保持你的私钥安全。在运行Keytool之前，请务必阅读[Securing Your Private Key][4]，它向用户描述了怎样保持你的私钥安全和为什么这样做很重要。特别的，当你生成你的密钥，你应该对keystore和key使用强密码类型。

</br>
> **警告**：保持生成的keystore文件在一个安全的地方，你必须使用相同的key去签名你将来要更新的应用程序。如果你使用一个新key发布一个应用程序，Google Play将认为它是一个新的app，在应用程序的整个生命周期，你必须保持相同的签名文件。详细请参考Android开发博客[Things That Cannot Change][5].

Keytool选项      | 描述
--------------- | ---------------------
-genkey  | Generate a key pair (public and private keys)
-v     | Enable verbose output.
-alias <alias_name>	 | An alias for the key. Only the first 8 characters of the alias are used.
-keyalg <alg>	| The encryption algorithm to use when generating the key. Both DSA and RSA are supported.
-keysize <size>	| The size of each generated key (bits). If not supplied, Keytool uses a default key size of 1024 bits. In general, we recommend using a key size of 2048 bits or higher.
-dname <name>	|  A Distinguished Name that describes who created the key. The value is used as the issuer and subject fields in the self-signed certificate. Note that you do not need to specify this option in the command line. If not supplied, Jarsigner prompts you to enter each of the Distinguished Name fields (CN, OU, and so on).
-keypass <password>	 | The password for the key. As a security precaution, do not include this option in your command line. If not supplied, Keytool prompts you to enter the password. In this way, your password is not stored in your shell history.
-validity <valdays>	| The validity period for the key, in days. **Note**: A value of 10000 or greater is recommended.
-keystore <keystore-name>.keystore |	A name for the keystore containing the private key.
-storepass <password>	| A password for the keystore. As a security precaution, do not include this option in your command line. If not supplied, Keytool prompts you to enter the password. In this way, your password is not stored in your shell history.

下面是一个使用keytool生成私钥的例子：

```
$ keytool -genkey -v -keystore my-release-key.keystore
-alias alias_name -keyalg RSA -keysize 2048 -validity 10000
```

运行上面例子中的命令，keytool将提示你为keystore和key输入密码，将为你的key提供专有名称字段，它将生成一个叫my-release-key.keystore的文件。这个keystore和key将受到密码保护。这个keystore只包含一个key,有效期1000天，这个别名是你将使用的名字。

关于keytool的更多信息，请参考

[http://docs.oracle.com/javase/6/docs/technotes/tools/windows/keytool.html](http://docs.oracle.com/javase/6/docs/technotes/tools/windows/keytool.html)

### 2. 在release模式下编译你的应用程序 ###

必须在release模式下编译你的应用程序。在release模式下编译应用程序不会使用debug签名，你将使用你的private key去签名它。

> **警告：**你不能发布未签名或者使用debug签名的应用程序

##### 使用Eclipse

从Eclipse里面导出一个未签名的APK，在Package Explorer视图右击项目名，选择**Android Tools > Export Unsigned Application Package**然后为未签名的APK指定位置（或者，打开`AndroidManifest.xml`文件，选择**Manifest**标签，点击**Export an unsigned APK**）

注意，你可以结合编译和签名步骤导出向导。请参见**使用Eclipse ADT编译和签名**

##### 使用Ant

如果你使用Ant,你可以在命令行使用`release`选项开启release模式。例如，如果你在包含build.xml的目录下运行ant,命令大概长这样：

```
$ ant relase
```

默认编译脚本编译这个应用程序APK，但不是为它签名。它将输出在`bin/`目录下面。类似`<your_project_name>-unsigned.apk`.因这个apk仍然未签名，你必须手动地为它签名，并且使用`zipalign`

然后，如果你有在ant.properties文件里面指定keystore的路径和key的名字，Ant的编译脚本也能执行签名和aligning，当你在执行`ant release`命令的时候将提示你输入密码。最终的输出文件在`bin/`目录下，类似`<your_project_name>-release.apk`,这些步骤是自动的。学习怎样在ant.properties文件中指定keystore和alias，参考[Building and Running Apps on the Command Line.][6]

### 3. 用私钥签名你的应用程序 ###

当你有一个应用程序包准备签名，你可以使用`jarsigner`工具为它签名，如在**基本设置**中所述请确保jarsigner工具在你的机器上可用。此处，确保keystore中所包含的private key是可用的。

运行jarsigner签名你的应用程序，引用将要签名的apk和所要签名的keystore，下表中列出了一些可以使用的参数：

Jarsigner选项       | 描述
------------------ | -----------------
-keystore <keystore-name>.keystore |	The name of the keystore containing your private key.
-verbose |	Enable verbose output.
-sigalg |	The name of the signature algorithim to use in signing the APK. Use the value SHA1withRSA.
-digestalg	| The message digest algorithim to use in processing the entries of an APK. Use the value SHA1.
-storepass <password>	| The password for the keystore. As a security precaution, do not include this option in your command line unless you are working at a secure computer. If not supplied, Jarsigner prompts you to enter the password. In this way, your password is not stored in your shell history.
-keypass <password>	| The password for the private key. As a security precaution, do not include this option in your command line unless you are working at a secure computer. If not supplied, Jarsigner prompts you to enter the password. In this way, your password is not stored in your shell history.

下面的例子演示了怎样使用jarsigner签名`my_application.apk`

```
$ jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -keystore my-release-key.keystore
my_application.apk alias_name
```

运行上面的例子命令，jarsigner提示你输入密码，然后它会原地修改这个apk,意味着这个apk现在已经被签名，注意，你可以使用不同的keys去多次签名一个apk。

> **警告：**在JDK 7中，默认的签名算法已经改变，当你签名一个apk需要你自己指定签名和算法摘要(-sigalg and -digestalg)

你可以使用命令验证你的apk签名：

```
$ jarsigner -verify my_signed.apk
```

如果这个apk被正确地签名了，Jarsigner打印“jar verified”,如果你想更多详细信息，你可以试试这些命令：

```
$ jarsigner -verify -verbose my_application.apk
```
或者

```
$ jarsigner -verify -verbose -certs my_application.apk
```

上面的命令，如果加上`-certs`,将显示`CN=`描述谁创建了这个key

> **注意：**如果你看到“CN=Android Debug”，意味着这个apk使用的是debug key签名，如果你准备发布你的应用程序，你必须使用private key去签名它。

更多Jarsigner的信息，请参考文档

[http://docs.oracle.com/javase/6/docs/technotes/tools/windows/jarsigner.html](http://docs.oracle.com/javase/6/docs/technotes/tools/windows/jarsigner.html)

### 4. Align最终的Apk包 ###

一旦你用你自己的private key签名了你的应用程序，在文件上运行 `zipalign`。此工具确保所有未压缩的数据开始一个特定的字节对齐，相对于文件的起点，当安装在设备上的时候，确保对齐在4个字节边界提供一个性能优化。当应用了aligned,Android系统可以使用mmap()读取文件，即使它们包含对齐限制的二进制数据，而不是从包里面复制所有的数据。其好处是在RAM中运行的应用程序所消耗的内存空间减少。

这个 `zipalign` 在Android SDK中是提供的，在 `tools/` 目录下面。使用align你的签名的apk,执行：

```
$ zipalign -v 4 your_project_name-unaligned.apk your_project_name.apk
```
 
这个`-v`标志打开详细输出（可选），4字节对齐（不要使用任何超过4的数字），第一个文件参数是你的签名的apk文件（输入），第二个文件参数是目标apk文件（输出），如果你覆盖一个已经存在的apk，增加 `-f` 参数。

> **警告：**在你使用 `zipalign` 优化这个包的时候，你输入的APK必须是使用private key签名过的。如果你使用 `zipalign` 之后再签名，对齐将被撤销。

更多信息，请阅读[ziplign](http://developer.android.com/tools/help/zipalign.html)工具。

### 使用Ecilpse ADT编译并签名

如果你使用Eclipse的ADT插件,你可以使用Export向导导出一个签名的apk(乃至创建一个新的keystore，如果需要的话)。Export向导使用keytool和jarsigner为你执行所有的交互，它允许你使用GUI而不是执行手动程序来编译、签名和对齐。如上面所讨论的签名包的所有互动。一旦向导编译并签名你的包，它也将使用zipalign对齐这个包。因为导出向导同时使用keytool和Jarsigner,你应该确保在你的电脑上面这些是有效的。如上面**基本签名**所述。

在Eclipse里面创建一个签名并且对齐的APK：

  1. 在Package Explorer视图选择项目，选择**File>Export**
  2. 打开Android文件夹，选择 Export Android Application,点击 Next。
  Export Android Application现在已经开始，这将引导你完成签名你的应用程序，包括用于选择private key与签名的apk（或者创建一个新的keystore和private key）步骤的过程。完成导出向导，你的应用程序将被编译、签名，对齐并准备进行发布。
  
私钥安全
---------
----------

对于你自己和用户来说，保持你的private key安全是至关重要的。如果你让别人使用你的key,或许如果你让你的keystore和密码在一个不安全的地主，使得第三方可以找到并使用它们，你的作者身份和用户的信任将会受到损害。

如果第三方在你不知情的情况下获得了你的key,这个可以签名和发布应用程序，恶意替换你真实的的应用程序或对齐造成损害。这样的人也可以根据你的身份签名或发布攻击其它应用程序或系统本身，或损坏或窃取用户数据。

你的private key需要签名你的应用程序的所有未来版本。如果你遗失或者找不到你的private key，你将无法更新发布到你现在有的应用程序。你不能再创建一个先前创建的key.

在任何时候，直到private key已过期，你作为一个开发者的声誉取决于确保private key的安全性。以下是保持你的private key安全的一些提示：

  - 对keystore和key使用强密码
  - 当你使用keytool生成你的key,不要在命令行上使用 `-storepass` 和 `-keypass` 选项，如果你这样做，你的密码将被记录在shell历史里面，你计算机上的任何用户都可以访问。
  - 同样的，当使用Jarsigner签名你的应用程序的时候，不要在命令行上使用 `-storepass` 和 `-keypass` 选项。
  - 不要把你的private key给或者借给任何人，不要让未经授权的人知道你的keystore和key密码。
  - 确保你使用keytool生成的包含private key的keystore在一个安全可靠的地方。
  
一般情况下，当你遵循常性性的预防措施生成、使用、存储你的key,它会保持安全。

*原文参考：[http://developer.android.com/tools/publishing/app-signing.html][7]*

[1]: http://developer.android.com/tools/building/building-cmdline.html#DebugMode]
[2]: http://developer.android.com/resources/faq/troubleshooting.html#signingcalendar
[3]: http://developer.android.com/tools/publishing/app-signing.html#ExportWizard
[4]: http://developer.android.com/tools/publishing/app-signing.html#secure-key
[5]: http://android-developers.blogspot.com/2011/06/things-that-cannot-change.html
[6]: http://developer.android.com/tools/publishing/app-signing.html#ExportWizard
[7]: http://developer.android.com/tools/publishing/app-signing.html
