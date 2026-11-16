# Botrobat
import os, zipfile, textwrap, json
base="/mnt/data/BAT_ROBOT"
os.makedirs(base+"/app/src/main/java/com/batrobot/app", exist_ok=True)
os.makedirs(base+"/app/src/main/res/xml", exist_ok=True)
os.makedirs(base+"/app/src/main/res/values", exist_ok=True)
files={
"settings.gradle": """pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
dependencyResolutionManagement { repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS); repositories { google(); mavenCentral() } }
rootProject.name = "BAT_ROBOT"
include(":app")
""",
"build.gradle": """plugins {
    id 'com.android.application' version '8.6.1' apply false
    id 'org.jetbrains.kotlin.android' version '2.0.20' apply false
}
""",
"gradle.properties": "org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8\nandroid.useAndroidX=true\nkotlin.code.style=official\n",
"app/build.gradle": """plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
}
android {
    namespace 'com.batrobot.app'
    compileSdk 35
    defaultConfig {
        applicationId 'com.batrobot.app'
        minSdk 23
        targetSdk 28
        versionCode 1
        versionName '1.0'
    }
}
""",
"app/src/main/AndroidManifest.xml": """<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application android:theme="@style/AppTheme" android:label="BAT ROBOT" android:allowBackup="false">
        <activity android:name=".MainActivity" android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <service
            android:name=".AppBlockService"
            android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE"
            android:exported="true"
            android:label="BAT ROBOT - محدودکننده برنامه‌ها">
            <intent-filter><action android:name="android.accessibilityservice.AccessibilityService"/></intent-filter>
            <meta-data android:name="android.accessibilityservice" android:resource="@xml/accessibility_config"/>
        </service>
    </application>
</manifest>
""",
"app/src/main/res/xml/accessibility_config.xml": """<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
    android:accessibilityEventTypes="typeWindowStateChanged"
    android:accessibilityFeedbackType="feedbackGeneric"
    android:notificationTimeout="100"
    android:canRetrieveWindowContent="false"
    android:description="@string/accessibility_description" />
""",
"app/src/main/res/values/styles.xml": """<resources>
    <style name="AppTheme" parent="@android:style/Theme.Material.Light.NoActionBar">
        <item name="android:fontFamily">sans</item>
        <item name="android:colorAccent">#E53935</item>
        <item name="android:statusBarColor">#101010</item>
        <item name="android:navigationBarColor">#101010</item>
        <item name="android:windowLightStatusBar">false</item>
    </style>
</resources>
""",
"app/src/main/res/values/strings.xml": """<resources>
    <string name="accessibility_description">برای محدودکردن برنامه‌های انتخاب‌شده در زمان‌بندی فعال استفاده می‌شود. تماس اضطراری و تنظیمات سیستم مستثنا هستند.</string>
</resources>
""",
"app/src/main/java/com/batrobot/app/MainActivity.kt": r'''package com.batrobot.app

import android.app.Activity
import android.content.Intent
import android.os.Bundle
import android.provider.Settings
import android.graphics.Color
import android.view.Gravity
import android.view.ViewGroup
import android.widget.*
import android.text.InputType
import android.content.Context

class MainActivity : Activity() {
    private val prefs by lazy { getSharedPreferences("bat_robot", Context.MODE_PRIVATE) }
    private var mode = "selected"

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        window.statusBarColor = Color.rgb(16,16,16)
        window.navigationBarColor = Color.rgb(16,16,16)
        val root = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            gravity = Gravity.CENTER_HORIZONTAL
            setPadding(28, 34, 28, 24)
            setBackgroundColor(Color.rgb(16,16,16))
        }
        fun label(text: String, size: Float, color: Int = Color.WHITE): TextView =
            TextView(this).apply { this.text=text; textSize=size; setTextColor(color); gravity=Gravity.CENTER }
        root.addView(label("🤖", 54f))
        root.addView(label("BAT ROBOT", 30f, Color.rgb(239,68,68)))
        root.addView(label("کنترل زمان استفاده از گوشی", 15f, Color.LTGRAY))
        val status = label("", 15f, Color.rgb(120,220,140))
        status.setPadding(0,24,0,12)
        root.addView(status)

        root.addView(label("حالت محدودیت", 18f))
        val group = RadioGroup(this).apply { orientation = RadioGroup.VERTICAL; setPadding(8,8,8,8) }
        val selected = RadioButton(this).apply { text="فقط برنامه‌های انتخابی (نسخه پایه: فهرست انتخاب در نسخه بعدی)"; setTextColor(Color.WHITE); id=1 }
        val full = RadioButton(this).apply { text="حالت گسترده (برنامه‌های معمولی)"; setTextColor(Color.WHITE); id=2 }
        group.addView(selected); group.addView(full); selected.isChecked=true
        group.setOnCheckedChangeListener { _, id -> mode = if(id==2) "all" else "selected" }
        root.addView(group)

        root.addView(label("مدت محدودیت (دقیقه)", 17f))
        val duration = EditText(this).apply {
            setText("60"); inputType=InputType.TYPE_CLASS_NUMBER
            setTextColor(Color.WHITE); setHintTextColor(Color.GRAY); gravity=Gravity.CENTER
            hint="مثلاً 60"; setPadding(12,12,12,12)
        }
        root.addView(duration, LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT))

        val start = Button(this).apply {
            text="شروع محدودیت"; setTextColor(Color.WHITE); setBackgroundColor(Color.rgb(190,35,35))
        }
        root.addView(start, LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, 58).apply { topMargin=22 })
        val access = Button(this).apply { text="فعال‌کردن دسترسی لازم" }
        root.addView(access)
        access.setOnClickListener { startActivity(Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS)) }

        val emergency = Button(this).apply { text="تماس اضطراری"; setBackgroundColor(Color.DKGRAY); setTextColor(Color.WHITE) }
        root.addView(emergency)
        emergency.setOnClickListener {
            try { startActivity(Intent(Intent.ACTION_DIAL)) }
            catch (_: Exception) { Toast.makeText(this,"امکان بازکردن شماره‌گیر نیست",Toast.LENGTH_SHORT).show() }
        }

        start.setOnClickListener {
            val mins = duration.text.toString().toLongOrNull()
            if (mins == null || mins !in 1..1440) {
                Toast.makeText(this,"مدت باید بین ۱ تا ۱۴۴۰ دقیقه باشد",Toast.LENGTH_LONG).show()
                return@setOnClickListener
            }
            val until = System.currentTimeMillis() + mins*60_000L
            prefs.edit().putLong("lock_until",until).putString("mode",mode).apply()
            status.text = "محدودیت فعال شد • ${mins} دقیقه"
            Toast.makeText(this,"زمان‌بندی ذخیره شد. سرویس دسترس‌پذیری باید روشن باشد.",Toast.LENGTH_LONG).show()
        }
        val until = prefs.getLong("lock_until",0L)
        status.text = if(until > System.currentTimeMillis()) "زمان‌بندی ذخیره‌شده فعال است" else "آماده"
        setContentView(root)
    }
}
''',
"app/src/main/java/com/batrobot/app/AppBlockService.kt": r'''package com.batrobot.app

import android.accessibilityservice.AccessibilityService
import android.view.accessibility.AccessibilityEvent
import android.view.WindowManager
import android.view.WindowManager.LayoutParams
import android.view.Gravity
import android.graphics.Color
import android.graphics.PixelFormat
import android.widget.LinearLayout
import android.widget.TextView
import android.content.Intent
import android.provider.Settings
import android.os.Handler
import android.os.Looper
import android.view.View

class AppBlockService : AccessibilityService() {
    private var blocker: View? = null
    private val handler = Handler(Looper.getMainLooper())

    // Android deliberately keeps Settings and the dialer reachable for recovery and emergencies.
    private val safePackages = setOf(
        "com.android.settings", "com.android.phone", "com.android.dialer",
        "com.google.android.dialer", "com.samsung.android.dialer",
        "com.samsung.android.incallui", "com.android.emergency",
        "com.batrobot.app"
    )

    override fun onAccessibilityEvent(event: AccessibilityEvent?) {
        if (event?.eventType != AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED) return
        val pkg = event.packageName?.toString() ?: return
        val prefs = getSharedPreferences("bat_robot", MODE_PRIVATE)
        val until = prefs.getLong("lock_until", 0L)
        if (until <= System.currentTimeMillis()) {
            prefs.edit().remove("lock_until").apply()
            removeBlocker()
            return
        }
        if (safePackages.contains(pkg)) { removeBlocker(); return }

        // Selected-app filtering is intentionally not claimed in this starter build.
        // "all" mode blocks ordinary foreground apps; selected mode is a timer-only preview.
        val mode = prefs.getString("mode", "selected")
        if (mode == "all") showBlocker(until) else removeBlocker()
    }

    private fun showBlocker(until: Long) {
        if (blocker != null) return
        val panel = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            gravity = Gravity.CENTER
            setPadding(32,32,32,32)
            setBackgroundColor(Color.rgb(16,16,16))
        }
        panel.addView(TextView(this).apply {
            text = "🤖  BAT ROBOT"
            textSize = 28f; setTextColor(Color.rgb(239,68,68)); gravity=Gravity.CENTER
        })
        val countdown = TextView(this).apply {
            textSize=20f; setTextColor(Color.WHITE); gravity=Gravity.CENTER
            setPadding(0,24,0,24)
        }
        panel.addView(countdown)
        panel.addView(TextView(this).apply {
            text="این برنامه در مدت تمرکز محدود شده است."
            textSize=16f; setTextColor(Color.LTGRAY); gravity=Gravity.CENTER
        })
        val dial = TextView(this).apply {
            text="تماس اضطراری"; textSize=18f; gravity=Gravity.CENTER
            setTextColor(Color.WHITE); setPadding(20,32,20,20)
            setOnClickListener {
                try { startActivity(Intent(Intent.ACTION_DIAL).addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)) }
                catch (_: Exception) {}
            }
        }
        panel.addView(dial)
        blocker = panel
        val params = LayoutParams(
            LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT,
            LayoutParams.TYPE_ACCESSIBILITY_OVERLAY,
            LayoutParams.FLAG_LAYOUT_IN_SCREEN or LayoutParams.FLAG_FULLSCREEN,
            PixelFormat.TRANSLUCENT
        ).apply { gravity=Gravity.TOP or Gravity.START }
        try { (getSystemService(WINDOW_SERVICE) as WindowManager).addView(panel,params) }
        catch (_: Exception) { blocker=null; return }

        val tick = object : Runnable {
            override fun run() {
                val left = until-System.currentTimeMillis()
                if (left <= 0) { removeBlocker(); return }
                val s=left/1000
                countdown.text="زمان باقی‌مانده\n%02d:%02d:%02d".format(s/3600,(s%3600)/60,s%60)
                handler.postDelayed(this,1000)
            }
        }
        handler.post(tick)
    }

    private fun removeBlocker() {
        handler.removeCallbacksAndMessages(null)
        val old=blocker ?: return
        try { (getSystemService(WINDOW_SERVICE) as WindowManager).removeView(old) } catch (_: Exception) {}
        blocker=null
    }
    override fun onInterrupt() { removeBlocker() }
    override fun onDestroy() { removeBlocker(); super.onDestroy() }
}
''',
"README_FA.txt": """BAT ROBOT - پروژه شروع اندروید

این فایل یک پروژه اولیه Android Studio است، نه APK آماده نصب.

ساخت:
1) ZIP را استخراج کنید.
2) پوشه BAT_ROBOT را با Android Studio باز کنید.
3) اجازه دهید Gradle Sync انجام شود.
4) گوشی را با USB وصل کنید و Run را بزنید، یا از Build > Build APK(s) خروجی بگیرید.

محدودیت‌های نسخه 1:
- حالت گسترده (all) با سرویس Accessibility روی برنامه‌های معمولی یک صفحه محدودیت نشان می‌دهد.
- تنظیمات سیستم، شماره‌گیر و خود BAT ROBOT عمداً مستثنا هستند تا راه بازیابی و تماس اضطراری بسته نشود.
- حالت انتخابی فعلاً فقط زمان‌بندی را ذخیره می‌کند؛ فهرست انتخاب برنامه‌ها هنوز پیاده‌سازی نشده است.
- این روش قفل امنیتی/غیرقابل‌دورزدن نیست. کاربر می‌تواند سرویس دسترس‌پذیری را از تنظیمات اندروید خاموش کند یا برنامه را متوقف کند.
- برای قفل سازمانیِ قوی‌تر، Android Enterprise / Device Owner لازم است و راه‌اندازی آن معمولاً نیازمند بازنشانی و ثبت مدیریتی دستگاه است.
- قبل از استفاده واقعی، روی دستگاه آزمایشی تست شود. تماس اضطراری و دسترسی به تنظیمات حفظ شده‌اند.

سازگاری هدف: Android 6+؛ پیکربندی اولیه برای دستگاه‌هایی مانند Samsung A50 با Android 11.
"""
}
for p,c in files.items():
    path=os.path.join(base,p)
    os.makedirs(os.path.dirname(path),exist_ok=True)
    with open(path,"w",encoding="utf-8") as f: f.write(c)
zip_path="/mnt/data/BAT_ROBOT_Android_Project.zip"
with zipfile.ZipFile(zip_path,"w",zipfile.ZIP_DEFLATED) as z:
    for root,dirs,fnames in os.walk(base):
        for fn in fnames:
            full=os.path.join(root,fn)
            z.write(full,os.path.relpath(full,base))
print(zip_path, os.path.getsize(zip_path))
