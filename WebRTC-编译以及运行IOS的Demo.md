######注：本篇文章的前提是，你已经下载好官方8G多的源代码，如还没有得到代码，请参照其他攻略先下载好代码。此外需说明下，我的系统环境为Mac OSX 10.11.4

######（1）下载安装ninja（如已安装，则可跳过这一步）：
######因为Xcode是不能直接编译webRTC的代码的，必须使用ninja。
- 获得并编译ninja的代码
```
$ git clone git://github.com/martine/ninja.git
$ cd ninja/
$ ./bootstrap.py
```
- 上述步骤会在当前目录下产生一个ninja的可运行文件。使用以下命令把它复制到/usrlocal/bin下
```
$ sudo cp ninja /usr/local/bin/
$ sudo chmod a+rx /usr/local/bin/ninja
```

######（2）配置需要的编译的目标环境
- cd 至下载好的源代码的src目录下，其实就是主目录就这一个大文件夹。
- 配置命令，以下命令根据demo的需要情况，只选其中一种执行，根据不同的情况，最后输出的demo也不一样。
  - 32位真机
```
export GYP_DEFINES="OS=ios target_arch=arm"
export GYP_GENERATOR_FLAGS="output_dir=out_ios"
```
  - 64位真机
```
export GYP_DEFINES="OS=ios target_arch=arm64"
export GYP_GENERATOR_FLAGS="output_dir=out_ios64"
```
  - 32位模拟器
```
export GYP_DEFINES="OS=ios target_arch=ia32"
export GYP_GENERATOR_FLAGS="output_dir=out_sim"
```
 - 64位模拟器
```
export GYP_DEFINES="OS=ios target_arch=x64"
export GYP_GENERATOR_FLAGS="output_dir=out_sim"
```
  - OSX
```
export GYP_DEFINES="OS=mac target_arch=x64"
export GYP_GENERATOR_FLAGS="output_dir=out_mac"
```
- 执行完上面的配置命令后 执行脚本
```
webrtc/build/gyp_webrtc.py
```

######（3）编译运行demo
- 这里有两种方式，第一种，使用ninja命令直接编译运行，第二种，生成xcode工程，在xcode里运行，但实际上并不是xcode编译的，而实际上还是运行了第一种的命令脚本。只不过使用xcode可以看见一些源代码，比较清晰些

- 直接编译运行
```
ninja -C out_ios/Debug-iphoneos AppRTCDemo
ninja -C out_ios/Release-iphoneos AppRTCDemo
ninja -C out_sim/Debug-iphonesimulator AppRTCDemo
```

- 生成xcode编译运行
  - 还是配置一下环境
```
export GYP_GENERATOR_FLAGS="xcode_project_version=3.2 xcode_ninja_target_pattern=All_iOS xcode_ninja_executable_target_pattern=AppRTCDemo|libjingle_peerconnection_unittest|libjingle_peerconnection_objc_test output_dir=out_ios"
export GYP_GENERATORS="ninja,xcode-ninja"
```
 - 还是要再执行一下上面那个脚本
```
webrtc/build/gyp_webrtc.py
```
 - 执行完后再src目录下就是生成一个all.ninja.xcworkspace工程，打开后会根据你在之前配置的情况生成不同的target，后面就可以command+R运行就好了，上面配置的是真机的话也是可以的。

######（4）测试demo
- 在真机装一个demo（因为真机有摄像头）
- 用谷歌或火狐浏览器打开
  https://apprtc.appspot.com/  需要翻墙
- 两边同时进入同一个房间号即可连接。