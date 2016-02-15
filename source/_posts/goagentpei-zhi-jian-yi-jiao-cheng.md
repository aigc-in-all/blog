---
layout: post
title: "GoAgent配置简易教程"
date: 2014-02-22 20:30:26 +0800
comments: true
categories: GoAgent
---
GoAgent是一个基于Google Appengine的，全面兼容IE，FireFox，chrome的代理工具，使用Python和Google App EngineSDK编写，程序可以在MicrosoftWindows，Mac，Linux，Android，iPod Touch，iPhone，iPad，webOS，OpenWrt，Maemo上使用。关键它还是免费的。

## 简易教程

* 部署 goagent
   1. 申请 Google Appengine并创建appid
   2. 下载 `goagent` 最新版[http://goo.gl/qFyRk](http://	goo.gl/qFyRk)
   3. 个性 `local/proxy.ini` 中的 `[gae]` 下的 `appid=` 你的	`appid` (多 `appid` 请用 `|` 隔开)
   4. 双击 `server\uploader.bat` 开始上传, 成功后即可使用了(地	址127.0.0.1:8087)
   		* MacOS/Linux 请在 Terminal 执行 `cd server && 	python uploader.zip`
   		
* 使用goagent
	* Chrome请安装 `SwitchySharp` 插件（拖放 `SwitchySharp.crx` 到	扩展设置），然后导入 `SwitchyOptions.bak`
	* 这里暂不介绍IE或其它浏览器
	
<!-- more -->

---

## 图文教程

### goagent GAE平台部署教程

* goagent GAE平台部署教程
	* 一、申请Google App Engine并创建appid
	* 二、下载goagent并上传到Google App Engine
	* 三、运行客户端
	* 谷歌Chrome配合Proxy Switchy Sharp扩展
	* goagent适用环境
	* 关于软件更新

#### 一、申请Google Ap Engine并创建appid

1. 申请注册一个Google App Engine账号[https://appengine.google.com](https://appengine.google.com)。没有Gmail账号先注册一个， 用你的Gmaill账号登录。
2. 登录之后，自动转向Application注册页面.
3. 接下来的页面，输入你的手机号码，需要注意的是，手机号码前面要+86（中国区号） 格式如：+86 13888888888。
	* 然后等待收取手机短信，收到短信后（一串数字号码）填入表	单，点send提交.（有的手机收不到信息，解决办法：详细教程 到	https://appengine.google.com/waitlist/sms_issues 提交	该情况，一个工作日就能收到谷歌提示Google App Engine成功开	通）。
	
4. 提交完成之后，GAE账号即被激活，然后就可以创建新的应用程序了。转入“My Applications”页面，点击“Create an Application”新建应用。
	* 一个Gmail账户最多可以创建十个GAE应用，每个应用每天1G免费流	量。这里我们只创建一个应用就可以了。进入下一步，填写新应用的必要	信息，如下图。在图中第一处添加一个应用名称，如abc555,验证一下	是否可用，如果显示“Yes”那么abc555就是你的Appid（记住这个	id），而abc555.appspot.com就是你的应用服务器地址了。第二个空	可随便填，点击Create Application按钮提交

#### 二、下载goagent并上传到Google App Engine
1. 下载goagent并解压，[http://goo.gl/qFyRk](http://goo.gl/qFyRk)
2. 上传
	* Windows用户：双击 `server` 文件夹下的 `upload.bat` ，输入你上步创	建的 `appid`（同时上传多appid在appid之间用 `|` 隔开,一次只能上传	同一个谷歌帐户下的 `appid` ）填完按回车。根据提示填你的谷歌帐户邮箱	地址，填完按回车。根据提示填你的谷歌帐户密码(注意：如果开启了两	步验证，密码应为16位的应用程序专用密码而非谷歌帐户密码，否则会	出现 `AttributeError: can't set attribute` 错误），填完按回	车。如果要上传多个谷歌帐户下的appid，先上传一个账号的，传完一个	账号后删除 `uploader.bat` 同目录下的 `.appcfg_cookies` 文件再传另	一个。
	* Linux/Mac用户上传方法：在server目录下执行：`python 	uploader.zip`。
	* 如遇到 `getaddrinfo failed，error10054，Error 10061` 目	标计算机积极拒绝等错误而不能上传，可以先运行 `goagent.exe` (要先	修改appid)并把IE代理设置为 `127.0.0.1：8087` 再运行		` uploader.bat` 。
	* 要使用IPv6上传或者上传遇到11004错误可以按照此贴进行修改或者	下载这个已经修改好的 `uploader.zip` 文件覆盖原 `uploader.zip` 文	件。
3. 上传成功后编辑 `local\proxy.ini`，把其中 `appid = goagent` 中的 `goagent` 改成你已经上传成功的应用的appid (用windows的记事本也可以）。
	* 如果要使用多个appid，appid之间用 `|` 隔开，如：`appid1|appid2|appid3` ，每个	appid必须确认上传成功才能使用。
	
		```
		[gae]
		appid = appid1|appid2|appid3
		```
		
#### 三、运行客户端
1. Windows用户运行local文件夹中的 `goagent.exe`， Linux/Mac用户运行 `proxy.py`
	* 设置浏览器或其他需要代理的程序代理地址为 `127.0.0.1:8087`
	* 注意：使用过程中要一直运行 `goagent.exe/proxy.py`
	* 代理地址 `127.0.0.1:8087`；如需使用PAC，设置pac地址为 `http://	127.0.0.1:8086/proxy.pac`；也可以配合 `SwitchySharp/AutoProxy` 等浏览器扩	展（SwitchySharp用户可从local文件夹中的 `SwitchyOptions.bak` 文件导入配置）
2. 导入证书
	* IE/Chrome：使用管理员身份运行 `goagent.exe` 会自动向系统导入IE/Chrome的证	书，你也可以双击 `local` 文件夹中的 `CA.crt` 安装证书（需要安装到“ `受信任的根证书颁发	机构` ”）；
	* 注意：请勿重复安装证书

#### 四、Chrome浏览器设置（安装Proxy Switchy Sharp扩展）
1. 安装扩展
	* 地址栏输入 `chrome://extensions/` 后按回车，打开扩展管理页，将local文件夹中	的 `SwitchySharp-0.9-beta-r48.crx` 拖拽到该页面之后点击确定即可安装，扩展也可	以从chrome应用商店获得 `https://chrome.google.com/webstore/detail/	proxy-switchysharp/dpplabbmogkhghncfbfdeeokoefdjegm`
2. 导入设置
	* 点击 Proxy SwitchySharp图标》选项》倒入/导出》从文件恢复
	* 浏览到SwitchyOptions.bak，点击确定导入设置
	* 更新自动切换规则（如果遇到无法更新规则列表，可以先运行goagent，并把浏览器代理设置为GoAgent?模式再更新规则，不更新规则只会影响自动切换模式，不会影响其他模式的使用，若确实无法更新也可不更新，直接使用PAC模式即可）
		* 在扩展设置页点击“切换规则”，点击“立即更新列表”，最后点击“保存”。
	* 单击地址栏右侧Proxy SwitchySharp图标即可进行模式选择
		1. GoAgent模式 除匹配proxy.ini中sites的直连外，其他全部通过GAE
		2. GoAgent PAAS模式 全部通过PAAS
		3. GoAgent Socks5模式 全部通过Socks5（暂不可用）
		4. 自动切换模式 根据切换规则自动选择是否进行代理，自动选择使用何种代理
		5. 遇到规则中没有的，可以使用扩展的“新建规则”按钮自行添加
		6. 这个扩展偶尔会出BUG，出现设置无误但浏览器提示错误130无法连接到代理服务器，可以将自己的设置导出之后卸载重装
		7. 如果遇到无法更新规则列表，可以先运行goagent，并把浏览器代理设置为GoAgent模式再更新规则，不更新规则只会影响自动切换模式，不会影响其他模式的使用，若确实无法更新也可不更新，直接使用PAC模式即可
		
#### 附：Mac电脑上面关于安装证书的问题
关于在Mac电脑上面双击CA.crt，成都把证书导入到钥匙串以后，访问某些网站可能会提示"无法连接到真正的 twitter.com"，此时需要修改证书的信任信息。
1. 打开钥匙串访问app
2. 从右上角的搜索框中搜索 `goagent`,然后下面列表中会列出刚刚安装的证书。
3. 鼠标右击证书》显示简介
4. 展开 `信任`，下拉 `使用此证书时：`选择 `总是信任即可`
