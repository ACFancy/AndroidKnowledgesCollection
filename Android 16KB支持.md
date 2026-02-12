## Android 16KB 对齐
- 参考官方 [支持16KB](https://developer.android.com/guide/practices/page-sizes?hl=zh-cn#update-packaging)
- [16KB环境中测试和向后兼容](https://developer.android.com/guide/practices/page-sizes?hl=zh-cn#build)
### 三库的支持
- 需要注意一些依赖的库需要升级到支持16KB的版本
- build.gradle配置, 比如下面的 compileSdk targetSdk都需要指定为34
  ```gradle
    android {
          compileSdk 34
  
          defaultConfig {
              ...
              targetSdk 34
          }
     }
  ```
  - gradle.properties配置如下示例
  ```gradle
    # Suppress AGP 7.4.2 warning when using compileSdk 34 (required by embedded deps)
    android.suppressUnsupportedCompileSdk=34
    # Fix Transform API deprecation warning (fat-aar plugin) 如有需要
    android.experimental.legacyTransform.forceNonIncremental=true
  ```

  ### App编译
  - build.graldle的配置，如有下面的配置可以去掉，或者 将`jniLibs.useLegacyPackaging = false`
  
  ```gradle
     packagingOptions {
          jniLibs.useLegacyPackaging = true
     }
  ```

  - AndroidManifest.xml中的配置 `android:extractNativeLibs="true"`需要设置为`false`或者去掉
  ```gradle
  android:extractNativeLibs="false"
  // 或者去掉
  ```
  - ndkVersion配置,最好 r28+以上版本
  ```gradle
    android {
      ndkVersion "28.0.12433566"
      ...
    }
  ```

  ### App打包apk, 需要build-tools版本 35.0.0+以上，以配置zipalign
  ``` python
    # build-tools 35.0.0+ required for zipalign -P 16 (16 KB page alignment)
    API_LEVEL = os.environ.get("ANDROID_API_LEVEL") or "35.0.0"
    zipalign = os.path.join(ENV['ANDROID_HOME'], "build-tools", API_LEVEL, add_ext("zipalign"))
  
    def zipalign_apk(src_apk):
      # 检查是否需要zip对齐
      # checkZipalignShell = zipalign + " -c -v 16 " + src_apk
      # log("checkZipalignShell: {}".format(checkZipalignShell))
      # os.system(checkZipalignShell)
  
      aligned_apk = OUT_PUT_DIR_NAME + APK_NAME
      deleteFile(aligned_apk)
      zipalignShell = zipalign + " -P 16 -f 4 " + src_apk + " " + aligned_apk
      log("zipalignShell: {}".format(zipalignShell))
      if os.system(zipalignShell) != 0:
          raise Exception("16KB 对齐失败")
      return aligned_apk
  ```

  ### 检查apk是否支持16KB的官方脚本（以后可能会更新）
  ```shell
        #!/bin/bash
      
      APK_PATH=$1
      
      if [ -z "$APK_PATH" ]; then
          echo "使用方法: ./check16k.sh your_app.apk"
          exit 1
      fi
      
      # 1. 检查共享库是否未压缩 (必须为 STORE 模式)
      echo "--- 检查压缩模式 (应为 Stored) ---"
      unzip -v "$APK_PATH" | grep ".so" | head -n 20
      
      echo -e "\n--- 检查 ELF 对齐 ---"
      # 2. 自动解压并检查每个 .so 文件的对齐偏移
      # 这里模拟官方检查逻辑：偏移量应能被 16384 (16KB) 整除
      unzip -ql "$APK_PATH" | grep ".so" | awk '{print $4}' | while read so_file; do
          # 提取 so 文件到临时目录
          unzip -p "$APK_PATH" "$so_file" > /tmp/temp.so
          # 使用 macOS 自带的 objdump 检查
          ALIGN=$(objdump -p /tmp/temp.so | grep -E "LOAD" | head -n 1)
          echo "$so_file: $ALIGN"
      done
  ```

  ### 检查xx.so是否支持 16KB的脚本（如果安装了Xcode）
  ```shell
   objdump -p <xx.so> | grep -A 1 LOAD
   objdump -p /Users/leedanatech/Downloads/gdx-backend-android-textureview-1.11.0-1/jni/arm64-v8a/libgdx.so | grep -A 1 LOAD
  ```

  ### 打包成aab格式的包 上传给Googe play后台，让后台自动处理
  - 需要后台配置好Google play的签名验证，否则上传不了aab的包
  
  
