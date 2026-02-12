## Android 16KB 对齐
- 参考官方 [支持16KB](https://developer.android.com/guide/practices/page-sizes?hl=zh-cn#update-packaging)
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
  
