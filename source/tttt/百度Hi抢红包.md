---
title: 百度Hi抢红包
date: 2020-01-23 13:07:38
tags:
comments: true
---
## 百度Hi自动抢红包

### 使用的工具
---
* [MachOView](https://github.com/gdbinit/MachOView)：用来查看二进制可执行文件
* [class—dump](https://github.com/nygard/class-dump): 静态分析，生成头文件
* [frida](https://github.com/frida/frida)：砸壳工具
* [chisel](https://github.com/facebook/chisel/wiki)：LLDB调试工具

### 获取砸壳的ipa文件
---
获取砸壳安装包的几种途径
1. 从pp助手的平台直接下载
2. 通过越狱手机dump运行的程序
我过越狱手机使用frida获取ipa安装包
### 安装包重签名
---
1. 获取证书列表
> security find-identity -p codesigning -v
2. 生成 entitlements.plist
3. 复制 mobileprovision
> 将 xxx.mobileprovision 文件复制到 .app 目录下
4. 签名

对.app文件夹中的所有动态库，插件进行签名

```
/usr/bin/codesign --force --sign "001B0FBF740108C5AF480CF01127C0E0BF7E8F35" "$1/$file"
```

对整个。app文件夹进行签名

```
codesign --force --sign "001B0FBF740108C5AF480CF01127C0E0BF7E8F35" --entitlements entitlements.plist $APP_PATH
```

5. 打包自己的ipa


重签名脚本文件如下
```
#!/bin/bash
SHELL_FOLDER=$(cd `dirname $0`; pwd) #获取当前目录
echo "TARGET_APP_PATH: $SHELL_FOLDER \n" > out.txt
APP_NAME='MyNewHi'
APP_PATH=$SHELL_FOLDER/$APP_NAME.app
echo "APP_PATH: $APP_PATH \n" >> out.txt
PROFILE_PATH=$APP_PATH/embedded.mobileprovision #项目中描述文件
SIGN_PROFILE_PATH=$SHELL_FOLDER/BaiduHaoKan_Dev.mobileprovision #签名描述文件

/usr/bin/security cms -D -i $PROFILE_PATH > profile.plist
/usr/libexec/PlistBuddy -x -c 'Print :Entitlements' profile.plist > entitlements.plist #授权文件
# /usr/libexec/PlistBuddy
#3 拷贝描述文件到 .app目录  下
# cp -rf $SIGN_PROFILE_PATH $APP_PATH/

#4 签名 对app文件中所有动态库、插件 watch目录下的extension进行签名

function codesign() {
    # echo "入参0: $0" >> out.txt
    echo "入参1: $1 " >> out.txt
    echo "入参: $@" >> out.txt
    for file in `ls $1`;
    do
        echo "file: $file" >> out.txt
        extension="${file#*.}"
        echo "extension: $extension" >> out.txt
        if [[ -d "$1/$file" ]]; then
            if [[ "$extension" == "framework" ]]; then
                /usr/bin/codesign --force --sign "001B0FBF740108C5AF480CF01127C0E0BF7E8F35" "$1/$file"
            else
                codesign "$1/$file"   
            fi
        elif [[ -f "$1/$file" ]]; then
            if [[ "$extension" == "dylib" ]]; then
                 /usr/bin/codesign --force --sign "001B0FBF740108C5AF480CF01127C0E0BF7E8F35" "$1/$file"
            elif [[ "$extension" == "appex" ]]; then
                /usr/bin/codesign --force --sign "$EXPANDED_CODE_SIGN_IDENTITY" "$1/$file"
            fi
        fi
    done
}

codesign "$APP_PATH"
# 对整个.app    文件夹进行签名
/usr/bin/codesign --force --sign "001B0FBF740108C5AF480CF01127C0E0BF7E8F35" --entitlements entitlements.plist $APP_PATH

mkdir Payload

cp -r $APP_PATH ./Payload
zip -qr $APP_NAME.ipa ./Payload
```
---
### 静态分析
`class-dump -H <mach-o-file> -o dir`

获取hi的头文件,分析为oc，swift混编。主要是OC代码
{% asset_img baiduhuiHeaders.jpg This is an example image %}
![头文件](./%e7%99%be%e5%ba%a6Hi%e6%8a%a2%e7%ba%a2%e5%8c%85/baiduhuiHeaders.jpg)

---
### 动态分析
通过Xcode运行调试HI程序，分析页面结构。跟踪汇编代码定位，消息接收方法，
```
红包消息体
<msg><font c="0" b="0" i="0" n="宋体" ul="0" s="10" cs="134" /><text c="" cfn="3"
apns="[百度红包]恭喜发财，一起来Hi">
<luckymoney id="46737c31b30541a3a156402647e32e05" message="恭喜发财，一起来Hi" type="1" source="1" sender_uid="3000711113" sender_header="51D08B3EFA137BF99291A352EEDB8D39.jpg" sender_name="jorgon1988" />
</text><text c="发红包了，快用新版手机Hi去抢：" cfn="999" />
<url ref="http://im.baidu.com/upgrade?t=luckymoney" c="http://im.baidu.com/upgrade?t=luckymoney" cfn="999" t="2" />
</msg>
```
和红包点击请求方法

---
### 代码注入

通过在moch-o文件添加动态加载库，注入自己编写的库
![头文件](./%e7%99%be%e5%ba%a6Hi%e6%8a%a2%e7%ba%a2%e5%8c%85/dyld.png)
{% asset_img dyld.png This is an example image %}
dyld.png

---
### 体验包
---