Table of Contents
=================
 * [1. build_image.py](#build_image_py)
 * [2. Integrate Prebuilt APK](#Integrate_Prebuilt_APK)
 * [3. Alarm Manager](#Alarm_manager)

<h1 id="build_image_py">1. build_image.py</h1>

<b>build_image.py</b> would be invoked when <br>
&nbsp; &nbsp; [1]. make systemimage and <br>
&nbsp; &nbsp; [2]. make target-files-package (adding the system image to target files first)<br>
The whole picture looks like: <br>
&nbsp; &nbsp; <img src="https://github.com/YuwenLee/Android_P/blob/master/pic/makefile_add-img-to-target-files.png" width=720/> <br>
It takes at least 3 arguments (input_dir, prop-dictionary, and output) and then invokes mkuserimg_mke2fs.sh to generate the output image.
<pre>
  build_image.py \
      out/target/product/abc/system \
      system_image_info.txt \
      obj/PACKING/targetfiles/system.img \
      out/target/product/abc/system
</pre>
[1]. input_dir
--------------
The source of the file system. Take the image of system as example, the input directory looks like
Note:
The symbolic links product and vendor are created in the procedure of making $(BUILT_SYSTEMIMAGE)

[2]. prop-dictionary
--------------------
A lookup table used to generate the command to make the image. In the makefile (LINUX/android/build/core/Makefile), the table is generated by generate-userimage-prop-dictionary:
<pre>
$(if $(BOARD_SYSTEMIMAGE_PARTITION_SIZE), \
   $(hide) echo "system_size=$(BOARD_SYSTEMIMAGE_PARTITION_SIZE)" \
   >> $(1))
</pre>
A complete dictionary looks like

[3]. out_file
-------------
Path of the output image file. This parameter is used as the 2nd argument passed to the build_command (ie mkuserimg_mke2fs.sh) :

[4]. target_out (optional)
--------------------------
According the comment in the script, this parameter is "the path of the product out directory to read device specific FS config files".
In the script, this parameter is used as -D option of the build_command. The arguments to the build command (mkuserimg.sh) are:
When building the system.img, build_image.py uses the yellow parts of the above prop-dictionary to generate the command:


<h1 id="Integrate_Prebuilt_APK">2. Integrate Prebuilt APK</h1>
Trace <b>build/make/core/prebuilt_internal.mk</b> to figure out how the build process handles the makefile (<b>Android.mk</b>) of a pre-built APK, for example, like the following one:<br/>

<pre>
LOCAL_PATH := $(call my-dir)

# XXX APK
include $(CLEAR_VARS)

# Module name should match apk name to be installed.
LOCAL_MODULE := XXX
ifeq ($(TARGET_PRODUCT),SKUA)
LOCAL_SRC_FILES := $(LOCAL_MODULE)_a.apk
else ifeq ($(TARGET_PRODUCT),SKUB)
LOCAL_SRC_FILES := $(LOCAL_MODULE)_b.apk
else ifeq ($(TARGET_PRODUCT),SKUC)
LOCAL_SRC_FILES := $(LOCAL_MODULE)_c.apk
else
LOCAL_SRC_FILES := $(LOCAL_MODULE).apk
endif
LOCAL_MODULE_PATH := $(TARGET_OUT_PRODUCT_APPS_PRIVILEGED)
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE_CLASS := APPS
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
LOCAL_CERTIFICATE := platform
LOCAL_PRIVILEGED_MODULE := true <---
LOCAL_DEX_PREOPT := true

include $(BUILD_PREBUILT)
</pre>
where <b>LOCAL_PRIVILEGED_MODULE := true</b> can have the signature of the APK removed.

<h1 id="Alarm_manager">3. Alarm Manager</h1>

  [1]. 概念<br/>
  [2]. 測試APK<br/>
  [3]. 鬧鐘類型<br/>
  [4]. Evidence/Log</br>

<h2>[1]. 概念</h2>
AlarmManager讓應用程式可以利用系統的alarm services，於未來的時間送Request給自己的某個class，執行規劃的動作。此要執行動作的class，須從BroadcastReceiver extends而來。<br/><br/>
應用程式先以Intent把要執行動作的Class包裝成一個訊息，再以PendingIntent Class包裝Intent。最後用AlarmManager的setRepeating( ) method把PendingIntent排預定的時間。<br/><br/>
當預定的時間到達時，AlarmManager的onReceive( ) method會被系統呼叫。AlarmManager的onReceive( )會佔住CPU Wake Lock，以確保在被規劃要做的動作完成前，Device不會進入Sleep。AlarmManager的onReceive( ) method會透過PendingIntent找到Intent，再由Intent找到包裝於其中的class，呼叫它的onReceive( )執行預先規劃的動作。<br/>

<h2>測試APK</h2>
此APK使用AlarmManager安排一個每兩分鐘執行一次的動作。

* 安裝
<pre>adb install</pre>

* 執行
<pre>adb shell am start -n com.example.tryalarmmanager/.MainActivity</pre>

按完SET ALARM Button後，離開MainActivity (Known Issue)，等待定期被執行。可觀察adb logcat<br/>
<pre>==YWLEE==: onReceive: null</pre><br/>
兩分鐘出現一次。測試APK使用的alarm type在Device進入Suspending Mode後仍然會運作。<br/>

[Known Issue]<br/>
測試APK若不離開MainActivity，仍然可以看到log，但Device無法進入Suspending Mode。<br/>

<h2>鬧鐘類型 (Alarm Type)</h2>

[Ref] https://www.cnblogs.com/zyw-205520/p/4040923.html<br/>
若要於固定的時間重複執行PendingIntent，要透過AlarmManager的Method：<br/>
<pre>
setRepeating(
  int  type, 
  long startTime,
  long intervalTime,
  PendingIntent pi
);
</pre>
此method第一個參數表示鬧鐘類型 (Alarm Type)，第二個參數表示鬧鐘首次執行時間，第三個參數表示鬧鐘兩次執行的間隔時間。鬧鐘類型會影響PendingIntent如何被執行。<br/>
[1]. AlarmManager.ELAPSED_REALTIME<br/>
    此狀態下鬧鐘使用相對時間，鬧鐘在睡眠狀態下不可用<br/><br/>
[2]. AlarmManager.ELAPSED_REALTIME_WAKEUP<br/>
    此狀態下鬧鐘使用相對時間，鬧鐘在睡眠狀態下會喚醒系統並執行提示功能<br/><br/>
[3]. AlarmManager.RTC<br/>
    此狀態下鬧鐘使用絕對時間，即當前系統時間。鬧鐘在睡眠狀態下不可用。<br/><br/>
[4]. AlarmManager.RTC_WAKEUP<br/>
    此狀態下鬧鐘使用絕對時間。鬧鐘在睡眠狀態下會喚醒系統並執行提示功能。<br/><br/>
[5]. AlarmManager.POWER_OFF_WAKEUP<br/>
    鬧鐘在手機關機狀態下也能正常進行提示功能。不過本狀態受SDK版本影響，某些版本並不支持。<br/><br/>

AlarmManager還有其他method可以設定PendingIntent如何被執行 (例如執行一次)。<br/>

<h2>Evidence of Waking from Suspending Mode</h2>
[Ref] KBA-161215181455_6_how_to_check_rtc_alarm_wakeup.pdf <br/>
測試APK使用的alarm type在Device進入Suspending Mode後仍然會運作。以下步驟可確認Device的確進入Suspending Mode並從Suspending Mode醒來。<br/>
[1]. 設定debug_mask<br/>
<pre>adb shell echo 1 > /sys/module/msm_show_resume_irq/parameters/debug_mask</pre>
[2]. Device進入Suspending Mode 的Log如下<br/>
<pre>PM : Suspending system (mem)
gic_show_resume_irq: 200 triggered qcom,smd-rpm
gic_show_resume_irq: 203 triggered 681b8.qcom,mpm
gic_show_resume_irq: 358 triggered 400f000.qcom,spmi
pm_system_irq_wakeup: 279 triggered qpnp_rtc_alarm</pre>
[3]. Device從Suspending Mode醒來的Log如下<br/>
<pre>PM : Finishing wakeup
==YWLEE==: onReceivie:null</pre>
[4]. 如果看不到[1]. 的Log，表示Linux Alarm Framework沒有使用qpnc_RTC裝置，沒有將alarm註冊到qpnc_RTC。這時候用以下指讓qpnc_RTC被使用
  <pre>adb shell echo 0 >/sys/module/qpnp_rtc/parameters/poweron_alarm</pre>
這是因為Qualcomm比較新的code在poweron_alarm為TRUE的時候，不使用qpnc_RTC。詳情可以參考kernel\kernel\time\alarmtimer.c：
<pre>static int alarmtimer_suspend(struct device *dev) 
{
  ...
  if (poweron_alarm) {
    ...
  } else {
    ...
  }
  ...
}
</pre>
