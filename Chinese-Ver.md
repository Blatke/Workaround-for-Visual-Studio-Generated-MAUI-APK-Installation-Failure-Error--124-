# 解决Visual Studio生成的MAUI APK安装失败（错误-124）的变通方案

<img height="300" alt="96c382f9877534deea787043542b2e34" src="https://github.com/user-attachments/assets/84f0b8fe-d900-43de-aea7-46674542fed7" />

> [!NOTE]
>
> 本方法由[DeepSeek](https://chat.deepseek.com)生成，经Bl@ke测试。

## 0. 测试环境
- Visual Stiduo 2026 18.7.0
- Xiaomi HyperOS 3.0.302.0 based on Android 16
- Windows 10 Home Edition 22H2 19045.7291

## 1. 问题原因

当您将Visual Studio生成的`.apk`文件（无论是直接发布还是通过“分发”向导）安装到小米HyperOS 3.0（基于Android 11+）手机上时，出现错误：  
`-124: Failed parse during installPackageLI: Targeting R+ (version 30 and above) requires the resources.arsc of installed APKs to be stored uncompressed and aligned on a 4-byte boundary`

**根本原因**：  
从Android 11（API 30）开始，系统强制要求APK包内的`resources.arsc`文件必须满足两个条件：  
- **不压缩**（存储模式，而非Deflate压缩）  
- **4字节对齐**（文件在APK中的起始偏移地址是4的整数倍）

Visual Studio在默认发布流程中（即使是Release配置）打包APK时，会使用`aapt2`工具并默认将`resources.arsc`进行压缩（Deflate）。这导致生成的APK不符合Android 11+的安装要求，从而被HyperOS系统拒绝。

## 2. 解决思路

由于在项目文件（`.csproj`）中添加`<AndroidNoCompress>resources.arsc</AndroidNoCompress>`等配置在您的环境中未能生效（可能被MAUI构建系统覆盖或存在工具链问题），我们采用**手动后处理**的方案：  
- 解压VS生成的APK  
- 删除原有的签名文件夹（META-INF）  
- **使用7-Zip重新打包**：将`resources.arsc`以**不压缩（存储）**方式添加，其他文件正常压缩  
- 对新生成的APK执行**对齐**（zipalign）  
- 使用您原有的密钥库**重新签名**  
- 最后通过ADB安装

该方案绕过VS的构建缺陷，保证最终APK完全符合系统要求。

## 3. 操作步骤（编制BAT批处理文件并执行）

### 前提条件
- 已安装[7-Zip](https://www.7-zip.org)（默认路径`C:\Program Files\7-Zip\7z.exe`）
- [Android SDK](https://developer.android.com/studio) Build Tools已安装（包含`zipalign`和`apksigner`），您的路径为：  
  `C:\Users\YourUserName\AppData\Local\Android\Sdk\build-tools\37.0.0`
- 已有可用的密钥库文件（例如`KeyName.keystore`），已知别名`keyname`
- VS生成的原始APK文件名为`VS_Generated.apk`，存放在`D:\APKs`文件夹

### 3.1 创建批处理文件
在`D:\APKs`文件夹下新建一个文本文件，命名为`FixAPK.bat`，右键选择“编辑”，复制粘贴以下内容：

```batch
@echo off
title APK修复工具 - 解决resources.arsc压缩问题

:: 设置工作目录（根据实际情况修改）
set WORK_DIR=D:\APKs
set ORIGINAL_APK=%WORK_DIR%\VS_Generated.apk
set OUTPUT_FINAL=%WORK_DIR%\final.apk

:: 7-Zip路径
set SEVENZIP="C:\Program Files\7-Zip\7z.exe"

:: Android SDK Build Tools路径（根据您的实际路径修改）
set BUILD_TOOLS=C:\Users\YourUserName\AppData\Local\Android\Sdk\build-tools\37.0.0

:: 密钥库信息
set KEYSTORE_PATH=C:\Users\YourUserName\AppData\Local\Xamarin\Mono for Android\Keystore\KeyName\KeyName.keystore
set KEY_ALIAS=keyname

cd /d %WORK_DIR%

:: 检查原始APK是否存在
if not exist "%ORIGINAL_APK%" (
    echo 错误：找不到原始APK文件 "%ORIGINAL_APK%"
    pause
    exit /b 1
)

:: 检查7-Zip
if not exist %SEVENZIP% (
    echo 错误：未找到7-Zip，请安装并确认路径。
    pause
    exit /b 1
)

:: 清理临时文件夹
rmdir /s /q temp_repack 2>nul

:: 1. 解压原始APK
echo [1/6] 正在解压 %ORIGINAL_APK% ...
%SEVENZIP% x "%ORIGINAL_APK%" -otemp_repack -y >nul

:: 2. 删除旧签名
echo [2/6] 移除旧签名文件...
rmdir /s /q temp_repack\META-INF 2>nul

:: 3. 重新打包（resources.arsc不压缩，其他文件最大压缩）
echo [3/6] 重新打包（强制resources.arsc存储模式）...
cd temp_repack
:: 先添加不压缩的resources.arsc
%SEVENZIP% a -tzip -mx=0 ..\repacked_unaligned.apk resources.arsc >nul
:: 再添加其余所有文件（排除resources.arsc），使用最大压缩
%SEVENZIP% a -tzip -mx=9 ..\repacked_unaligned.apk * -x!resources.arsc >nul
cd ..

:: 4. 对齐APK
echo [4/6] 执行4字节对齐...
"%BUILD_TOOLS%\zipalign" -v -p 4 repacked_unaligned.apk aligned.apk >nul

:: 5. 签名APK
echo [5/6] 使用密钥库签名...
"%BUILD_TOOLS%\apksigner" sign --ks "%KEYSTORE_PATH%" --ks-key-alias %KEY_ALIAS% --out "%OUTPUT_FINAL%" aligned.apk
if errorlevel 1 (
    echo 签名失败！请检查密钥库密码是否正确。
    pause
    exit /b 1
)

:: 6. 清理临时文件
echo [6/6] 清理临时文件...
del repacked_unaligned.apk aligned.apk 2>nul
rmdir /s /q temp_repack 2>nul

echo ========================================
echo 修复完成！最终APK已生成：%OUTPUT_FINAL%
echo ========================================
pause
```

> [!NOTE]
>
> 请根据您实际的SDK路径、密钥库路径和别名修改上述脚本中的变量:
> ```
> set BUILD_TOOLS=C:\Users\YourUserName\AppData\Local\Android\Sdk\build-tools\37.0.0
> set KEYSTORE_PATH=C:\Users\YourUserName\AppData\Local\Xamarin\Mono for Android\Keystore\KeyName\KeyName.keystore
> set KEY_ALIAS=keyname
> ```

**注意**：

### 3.2 执行批处理文件
- 将VS生成的`VS_Generated.apk`文件复制到`D:\APKs`文件夹（或直接在该文件夹下生成）
- 双击运行`FixAPK.bat`（或右键“以管理员身份运行”）
- 脚本运行过程中会提示每个步骤，约10~20秒完成
- 如果签名时需要输入密钥库密码，命令行会提示`Keystore password for signer #1:`，手动输入密码（屏幕不显示）后回车

**执行结果**：  
在`D:\APKs`文件夹下生成`final.apk`，该文件中的`resources.arsc`已变为**未压缩**（可用7-Zip验证，Method显示`Store`），且已对齐并签名。

## 4. 通过CMD向手机安装生成的APK

1. 打开命令提示符（CMD）或PowerShell
2. 进入存放最终APK的目录：
   ```cmd
   cd /d D:\APKs
   ```
3. 确保手机已通过USB连接到电脑，并已开启“开发者选项”中的“USB调试”
4. 执行安装命令：
   ```cmd
   adb install -r final.apk
   ```
   （`-r`参数表示如果已安装则覆盖）
5. 等待安装完成，手机屏幕上会显示“应用安装成功”或CMD中输出`Success`

**验证**：  
打开手机上的应用，确认功能正常。若未来需要更新，只需重新执行上述批处理流程（使用新的VS生成的APK作为`VS_Generated.apk`），再次安装`final.apk`即可。

## 5. 附录：长期建议

虽然手动修复可以工作，但为了节省时间，建议您尝试彻底修复项目文件。在`.csproj`的第一个`PropertyGroup`中添加：
```xml
<AndroidAapt2AdditionalArguments>--no-compress resources.arsc</AndroidAapt2AdditionalArguments>
<AndroidNoCompress>resources.arsc</AndroidNoCompress>
```
然后删除`bin`和`obj`，重新生成APK。如果生成的APK仍为压缩状态，请将问题反馈至.NET MAUI社区，并将此批处理作为临时解决方案继续使用。

---

**总结**：通过上述步骤，您可以在不依赖VS正确配置的情况下，将任何VS生成的APK转换为符合Android 11+要求的可安装包，成功在HyperOS 3.0上运行。
