# 如何從這份原始碼包出自己的 APK

這是 [MediaTek-Labs/linkit-remote-android](https://github.com/MediaTek-Labs/linkit-remote-android) 的原始碼，
最後一次更新是 **2017 年**。原始版本用最新版 Android Studio 直接開啟**一定會建置失敗**，
主要原因有三個：

1. `app/build.gradle` 一開啟就強制讀取一個叫 `keystore.properties` 的檔案，但這個檔案
   從來沒有放進 repo 裡（本來就該由開發者自己保管），所以連 Gradle sync 都會直接噴錯。
2. 專案只設定了 `jcenter()` 這個套件庫，沒有 `google()`。而 App 用到的
   `com.android.support` 系列函式庫是放在 Google 自己的 Maven repo 上，沒有 `google()`
   就抓不到。
3. 專案釘死在 2017 年的 Gradle 外掛版本（AGP 3.0.1 + Gradle 4.1），跟現在的
   Android Studio 差了將近 10 個大版本，語法（例如 `compile`）也早就被移除了。

我已經在這份 zip 裡把上述問題都修好了，你可以直接拿去建置：

- ✅ `keystore.properties` 改成「選用」：**沒有它也能建置 Debug APK**，只有要做
  簽署過的 Release APK 時才需要。
- ✅ 加上 `google()` + `mavenCentral()`。
- ✅ 升級成 AGP 8.13.2 + Gradle 8.13（目前 Android Studio 完全支援、且不用改寫 DSL
  語法的穩定組合）。
- ✅ `compile` / `androidTestCompile` / `testCompile` 改成現在的
  `implementation` / `androidTestImplementation` / `testImplementation`。
- ✅ 補上 AGP 8 要求的 `namespace`。
- ✅ `targetSdkVersion` 提高到 24，避免 Android 15 以上安裝時出現
  `INSTALL_FAILED_DEPRECATED_SDK_VERSION`。

> 如果你打開專案時 Android Studio 又跳出「Upgrade Gradle Plugin」的提示，
> 直接按接受升級就好，不會影響這份專案。

---

## 1. 事前準備

- 安裝 [Android Studio](https://developer.android.com/studio)（內建 JDK，不用另外裝）。
- 用手機或模擬器測試的話，開啟手機的「開發者選項」→「USB 偵錯」。

## 2. 建置 Debug APK（自己裝，最簡單）

### 方法 A：用 Android Studio 圖形介面

1. 開啟 Android Studio → `Open` → 選這個資料夾。
2. 等右下角 Gradle sync 跑完（第一次會下載 Gradle 8.4，需要一點時間）。
3. 上方選單 `Build` → `Build Bundle(s) / APK(s)` → `Build APK(s)`。
4. 建置完成後，右下角會跳出通知，點 `locate` 就能找到 APK。

### 方法 B：命令列

```bash
cd linkit-remote-android
./gradlew assembleDebug      # macOS / Linux
gradlew.bat assembleDebug    # Windows
```

建置完成後，APK 會在：

```
app/build/outputs/apk/debug/app-debug.apk
```

把這個檔案傳到手機（USB / 雲端硬碟 / Email 都可以）安裝即可，或是接上手機後直接：

```bash
adb install app/build/outputs/apk/debug/app-debug.apk
```

Debug APK 是用 Android SDK 內建的 debug key 簽署的，**可以正常安裝、正常使用**，
只是不能上架 Google Play（上架需要用你自己的正式 key 簽署，也就是下面的 Release 流程）。

> Android 15 以上會拒絕安裝 `targetSdkVersion` 低於 24 的 APK。這份專案已經把
> target SDK 從 23 提高到 24；如果你的手機仍顯示「這個應用程式是為舊版 Android
> 打造」或類似訊息，請重新建置 APK 後再安裝，不要使用舊的 `app-debug.apk`。

## 3.（進階、非必要）建置已簽署的 Release APK

只有你想要一份「正式簽名」的版本（例如要長期使用同一把 key 持續更新）才需要做這步，
自己單純想裝來玩，做到上面第 2 步就完全足夠。

1. 產生一把你自己的 keystore（第一次做，之後每次更新 App 都要用同一把）：

   ```bash
   keytool -genkey -v -keystore my-release-key.jks -keyalg RSA -keysize 2048 -validity 10000 -alias my-key-alias
   ```

   過程會請你設定密碼、輸入姓名等資訊，跟著填就好，**keystore 檔案跟密碼要自己保管好，
   弄丟就無法用同一個簽名再更新這個 App 了**。

2. 在專案根目錄（跟 `build.gradle` 同一層）新增一個 `keystore.properties` 檔案，內容：

   ```properties
   storeFile=/絕對路徑/my-release-key.jks
   storePassword=你剛剛設的 store 密碼
   keyAlias=my-key-alias
   keyPassword=你剛剛設的 key 密碼
   ```

   （這個檔案已經被加進 `.gitignore` 邏輯，不會被 git 追蹤，密碼不會外洩到 repo 裡。）

3. 建置 Release APK：

   ```bash
   ./gradlew assembleRelease
   ```

   輸出位置：

   ```
   app/build/outputs/apk/release/app-release.apk
   ```

## 4. 如果建置還是失敗

- **抓不到某個套件版本**：把該行版本號改成 [mvnrepository.com](https://mvnrepository.com/) 上
  查得到的最新相近版本即可，例如把 `constraint-layout:1.0.2` 改成
  `androidx.constraintlayout:constraintlayout:2.1.4`（但如果你要換成 androidx 版本，
  記得同時做 `Refactor` → `Migrate to AndroidX`，讓 Android Studio 自動把程式碼裡的
  `import android.support.xxx` 換成 `import androidx.xxx`）。
- **Android Studio 顯示 SDK Platform 33 未安裝**：`Tools` → `SDK Manager` 把
  Android 13.0（API 33）的 SDK Platform 勾起來下載即可。
- 這個 App 需要藍牙與定位權限（BLE 掃描在 Android 上需要定位權限），第一次執行時
  記得在手機上同意權限請求，掃不到裝置十之八九是權限或藍牙沒開。
