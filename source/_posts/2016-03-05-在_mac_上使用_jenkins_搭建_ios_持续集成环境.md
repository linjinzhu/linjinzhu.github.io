---
layout: post
title:  "在 Mac 上使用 Jenkins 搭建 iOS 持续集成环境"
date:   2016-03-05 16:10:08 +0800
categories: jenkins iOS
---

## 下载安装 Jenkins

1. 在 [Jenkins](http://mirrors.jenkins-ci.org/war/latest/jenkins.war) 官网下载 jenkins.war；
2. 配置环境变量 JENKINS_HOME。
	- 创建 jenkins home 目录：`~/jenkins/home` ；
	- 在 ~/.bash_profile 文件追加 `export JENKINS_HOME=~/jenkins/home`，如果~/.bash_profile 文件不存在要先创建。使用 `source ~/.bash_profile` 命令让 JENKINS_HOME 马上生效；
3. 将 jenkins.war 文件放到 `~/jenkins/bin` 目录下，目录可以自定义，后面介绍启动命令时会用到 jenkins.war 的路径。
4. 启动 jenkins：

		$ nohup java -jar /Applications/jenkins.war --httpPort=8080 &

5. 自定义启动、停止、显示进程脚本*(可以忽略不看)*。
	- 自定义启动脚本，在 `~/jenkins/bin/` 目录下创建 startup.sh 文件：
	
			#!/bin/bash

			nohup java -jar jenkins.war --httpPort=8080 &
	- 自定义停止脚本，在 `~/jenkins/bin/` 目录下创建 shutdown.sh 文件：
			
			#!/bin/bash

			ps x|grep jenkins|grep -v grep |awk '{print $1}'|xargs kill -9

	- 自定义显示进程的脚本，在 `~/jenkins/bin/` 目录下创建 showpid.sh 文件：
	
			#!/bin/bash

			ps x|grep jenkins|grep -v grep |awk '{print $1}'

	- 为脚本添加可执行权限：
		
			$ chmod a+x startup.sh
			$ chmod a+x shutdown.sh
			$ chmod a+x showpid.sh
			
	- 启动 jenkins 的命令就变成了 `sh ~/jenkins/bin/startup.sh`，关闭 ： `sh ~/jenkins/bin/shutdown.sh`，显示进程 ：`sh ~/jenkins/bin/showpid.sh`。 

## 配置 Jenkins

### 注册用户

1. 允许用户注册。用浏览器访问 http://localhost:8080，进入 `Manage Jenkins > Configure Global Security`，点选 Jenkins’ own user database，勾选 `Allow users to sign up`；
2. 注册用户。现在在右上角可以看到注册按钮，点击注册用户；

### 安装 git 插件

进入 `Manage Jenkins > Manage Plugins > Available`，搜索 **git plugin**，勾选 “git plugin”，点击 “Install without restart” 按钮；

## 创建脚本

#### 配置文件脚本

存放全局变量，应用名称、版本号、私钥密码、描述文件名称、账号信息等；

	config.sh
	
	#!/bin/bash
	
	# Standard app config
	export APP_NAME=JenkinsExample
	export DEVELOPER_NAME="iPhone Distribution: JinZhu Lin (9JGQ936LPE)"
	export PROFILE_NAME=Baidu_Distribution
	export INFO_PLIST=JenkinsExample/Info.plist
	export BUNDLE_DISPLAY_NAME=JenkinsExample CI
	
	# Edit this for local testing only, DON'T COMMIT it:
	export ENCRYPTION_SECRET=foo
	export KEY_PASSWORD=111111
	export TESTFLIGHT_API_TOKEN=
	export TESTFLIGHT_TEAM_TOKEN=
	export HOCKEY_APP_ID=2ee4b5b87d7c47acbb398bc3cf56cda1
	export HOCKEY_APP_TOKEN=549b49be50814e00b74e48322bbd2c35
	
	# This just emulates Travis vars locally
	export TRAVIS_PULL_REQUEST=false
	export TRAVIS_BRANCH=master
	export TRAVIS_BUILD_NUMBER=0

#### 更新编译信息

原理是更新 xcode 工程的 Info.plist 文件，修改 build number、bundle identifier；

	#!/bin/bash
	
	if [ ! -z "$INFO_PLIST" ]; then
	  /usr/libexec/PlistBuddy -c "Set :CFBundleVersion $TRAVIS_BUILD_NUMBER" "$INFO_PLIST"
	  echo "Set CFBundleVersion to $TRAVIS_BUILD_NUMBER"
	fi
	
	if [ ! -z "$BUNDLE_IDENTIFIER" ]; then
	  /usr/libexec/PlistBuddy -c "Set :CFBundleIdentifier $BUNDLE_IDENTIFIER" "$INFO_PLIST"
	  echo "Set CFBundleIdentifier to $BUNDLE_IDENTIFIER"
	fi

#### 准备证书和描述文件

- 执行编译和打包命令都会依赖证书和描述文件。这个步骤是导入苹果授权的证书到 keychain，并设置默认 keychain，设置 keychain 超时时间。
- 复制 provisioning profile 到系统目录 `~/Library/MobileDevice/Provisioning\ Profiles/`，必须要放到这个目录，否则打包时找不到对应的 provisioning profile。


		#!/bin/sh
		
		# Create a custom keychain
		#security create-keychain -p jenkins ios-build.keychain
		
		# Make the custom keychain default, so xcodebuild will use it for signing
		security default-keychain -s login.keychain
		
		# Unlock the keychain
		security unlock-keychain -p jenkins login.keychain
		
		# Set keychain timeout to 1 hour for long builds
		# see http://www.egeek.me/2013/02/23/jenkins-and-xcode-user-interaction-is-not-allowed/
		security set-keychain-settings -t 3600 -l ~/Library/Keychains/login.keychain
		
		# Add certificates to keychain and allow codesign to access them
		#security import ./jenkins/scripts/certs/apple.cer -k ~/Library/Keychains/login.keychain -T /usr/bin/codesign
		#security import ./jenkins/scripts/certs/dist.cer -k ~/Library/Keychains/login.keychain -T /usr/bin/codesign
		#security import ./jenkins/scripts/certs/dist.p12 -k ~/Library/Keychains/login.keychain -P $KEY_PASSWORD -T /usr/bin/codesign
		
		
		# Put the provisioning profile in place
		mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
		cp "./jenkins/scripts/profile/$PROFILE_NAME.mobileprovision" ~/Library/MobileDevice/Provisioning\ Profiles/


#### 执行编译命令

xcode 自带 xcodebuild 命令行工具，原生支持命令行编译打包，这里使用的是 Facebook 的命令行工具 xctool。

	xctool -workspace JenkinsExample.xcworkspace -scheme JenkinsExample -sdk iphoneos -configuration Release OBJROOT=$PWD/build SYMROOT=$PWD/build ONLY_ACTIVE_ARCH=NO 'CODE_SIGN_RESOURCE_RULES_PATH=$(SDKROOT)/ResourceRules.plist'

#### 签名打包并上传到服务器

签名用的是 xcrun 工具，将编译好的 JenkinsExample.app 签名并输出成 JenkinsExample.ipa 文件，使用内嵌在 provisioning profile 里的 sign identifier 给 JenkinsExample.app 签名。

	#!/bin/sh
	if [[ "$TRAVIS_PULL_REQUEST" != "false" ]]; then
	  echo "This is a pull request. No deployment will be done."
	  exit 0
	fi
	if [[ "$TRAVIS_BRANCH" != "master" ]]; then
	  echo "Testing on a branch other than master. No deployment will be done."
	  exit 0
	fi
	
	# Thanks @djacobs https://gist.github.com/djacobs/2411095
	# Thanks @johanneswuerbach https://gist.github.com/johanneswuerbach/5559514
	
	PROVISIONING_PROFILE="$HOME/Library/MobileDevice/Provisioning Profiles/$PROFILE_NAME.mobileprovision"
	OUTPUTDIR="$PWD/build/Release-iphoneos"
	
	echo "***************************"
	echo "*        Signing          *"
	echo "***************************"
	xcrun -log -sdk iphoneos PackageApplication "$OUTPUTDIR/$APP_NAME.app" -o "$OUTPUTDIR/$APP_NAME.ipa" -sign "$DEVELOPER_NAME" -embed "$PROVISIONING_PROFILE"
	
	zip -r -9 "$OUTPUTDIR/$APP_NAME.app.dSYM.zip" "$OUTPUTDIR/$APP_NAME.app.dSYM"
	
	RELEASE_DATE=`date '+%Y-%m-%d %H:%M:%S'`
	RELEASE_NOTES="Build: $TRAVIS_BUILD_NUMBER\nUploaded: $RELEASE_DATE"
	
	if [ ! -z "$TESTFLIGHT_TEAM_TOKEN" ] && [ ! -z "$TESTFLIGHT_API_TOKEN" ]; then
	  echo ""
	  echo "***************************"
	  echo "* Uploading to Testflight *"
	  echo "***************************"
	  curl http://testflightapp.com/api/builds.json \
	    -F file="@$OUTPUTDIR/$APP_NAME.ipa" \
	    -F dsym="@$OUTPUTDIR/$APP_NAME.app.dSYM.zip" \
	    -F api_token="$TESTFLIGHT_API_TOKEN" \
	    -F team_token="$TESTFLIGHT_TEAM_TOKEN" \
	    -F distribution_lists='Internal' \
	    -F notes="$RELEASE_NOTES"
	fi
	
	if [ ! -z "$HOCKEY_APP_ID" ] && [ ! -z "$HOCKEY_APP_TOKEN" ]; then
	  echo ""
	  echo "***************************"
	  echo "* Uploading to Hockeyapp  *"
	  echo "***************************"
	  curl https://rink.hockeyapp.net/api/2/apps/$HOCKEY_APP_ID/app_versions \
	    -F status="2" \
	    -F notify="0" \
	    -F notes="$RELEASE_NOTES" \
	    -F notes_type="0" \
	    -F ipa="@$OUTPUTDIR/$APP_NAME.ipa" \
	    -F dsym="@$OUTPUTDIR/$APP_NAME.app.dSYM.zip" \
	    -H "X-HockeyAppToken: $HOCKEY_APP_TOKEN"
	fi

#### 删除证书和描述文件

打包完毕，清理证书和描述文件，以后下次打包；
	
	#!/bin/sh
	#security delete-keychain ios-build.keychain
	rm -f ~/Library/MobileDevice/Provisioning\ Profiles/$PROFILE_NAME.mobileprovision


## 在 Jenkins 中创建编译打包任务

1. Item name 设置为 JenkinsExample；
2. Source Code Management 选 git，设置项目的 git 库地址；
3. Build 项下点击 “Add build step” 按钮，添加执行命令：
	
		. ./jenkins/config.sh
		
		openssl aes-256-cbc -k "$ENCRYPTION_SECRET" -in jenkins/scripts/profile/Baidu_Distribution.mobileprovision.enc -d -a -out jenkins/scripts/profile/Baidu_Distribution.mobileprovision
		openssl aes-256-cbc -k "$ENCRYPTION_SECRET" -in jenkins/scripts/certs/dist.cer.enc -d -a -out jenkins/scripts/certs/dist.cer
		openssl aes-256-cbc -k "$ENCRYPTION_SECRET" -in jenkins/scripts/certs/dist.p12.enc -d -a -out jenkins/scripts/certs/dist.p12
		
		#. ./jenkins/scripts/add-key.sh
		. ./jenkins/scripts/update-bundle.sh
		
		xctool -workspace JenkinsExample.xcworkspace -scheme JenkinsExample -sdk iphoneos -configuration Release OBJROOT=$PWD/build SYMROOT=$PWD/build ONLY_ACTIVE_ARCH=NO 'CODE_SIGN_RESOURCE_RULES_PATH=$(SDKROOT)/ResourceRules.plist'
		
		. ./jenkins/scripts/sign-and-upload.sh
		#. ./jenkins/scripts/remove-key.sh
		
4. 点击保存，点击“Build Now”开始执行任务；

## 链接

- [JenkinsExample Github 地址](https://github.com/linjinzhu/JenkinsExample.git)
- [Jenkins+GitHub+Xcode+fir搭了一个持续集成环境(解决了为什么启动时会用 jenkins 用户启动的问题)](http://xuanyiliu.com/chixujicheng/)