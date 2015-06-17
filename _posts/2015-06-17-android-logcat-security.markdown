---
published: true
title: Android Logcat Security
layout: post
---
#0x00 Background Knowledge

-----

Development version:  beta version for internal test contains a lot of debugging logs

Release version:official version for customers contains less debugging logs.

Android.util.Log: offers five export functions

    Log.e(), Log.w(), Log.i(), Log.d(), Log.v()

    ERROR, WARN, INFO, DEBUG, VERBOSE

Android permission READ_LOGS:logs reading permission. App logs can be accessed by requesting READ_LOGS permission in Android  prior to 4.1. But Google finds it is a security risk. So in Android 4.1 and later version, App logs can not be accessed with READ_LOGS permission.  The signature of Logcat has been changed into `signature|system|development` in Android 4.1, which means that only system signed apps or apps with root permission are allows to gain this permission. Ordinary users can view all of the logs through ADB.

#0x01 test

-----

Test method is very simple. You can use a small tool, like monitor provided in the SDK or the logcat integrated in ADT to view logs. It's relatively more convenient to add these tool directories into environment variables. Of course, though it's cooler to use adb logcat. In overall, Android logs contain a very large amount of information, which makes it a necessity to filter out some irrelevant parts. The filter supports regular expressions so that keywords matching is allowed, such as password, token and email. At first, I was thinking about creating a small tool that enables automated collection. However such a tool is not very helpful. In the end, I gave up my original thoughts. Therefore, the focus of this post is on using logcat appropriately.

![image](https://quip.com/blob/QefAAAOvlFp/U4WA3IlnTtQJ46a4-YBtFw?s=WfhYAqYJunaN)

![image](https://quip.com/blob/QefAAAOvlFp/dWK57T5IIXjZyb8J33ZAHg?s=WfhYAqYJunaN)

Apparently, the other option is to write an app that directly captures logcat on cell phones. But as we mentioned above, App logs cannot be retrieved even if the following request is added in manifest.xml for android 4.1 and later version.

```
<uses-permission android:name="android.permission.READ_LOGS"/>
```

![image](https://quip.com/blob/QefAAAOvlFp/b8OfWNdHvCCRAYXwgjuiYg?s=WfhYAqYJunaN)

On the other hand, if you have root permission, you can easily access logcat. In other words, Google does no help to fix the vulnerabilities that logcat caused and their efforts on Android 4.1 are meaningless.

![image](https://quip.com/blob/QefAAAOvlFp/3GEQQTkwfT2n9sLtrYQIHQ?s=WfhYAqYJunaN)

#0x02 smali injected logcat

-----

An article http://drops.wooyun.org/tips/2986 stated that printing out sensitive information before encrytion is actually a method that uses static smali to inject logcat.  It's very convenient to use APK 改之理 to inject smali，but you need to remember that adding a register might destroy its logic. As a result, green hands are not suggested to do so. The best option for them is to use existing registers. 


    invoke-static {v0, v0}, Landroid/util/Log;->e(Ljava/lang/String; Ljava/lang/String;)I


![image](https://quip.com/blob/QefAAAOvlFp/TFuS5NETHe_QKS653Ftrkg?s=WfhYAqYJunaN)

![image](https://quip.com/blob/QefAAAOvlFp/eVV1NTQmcKElF8xSCnAJZg?s=WfhYAqYJunaN)

#0x03 recommendations

-----

Some people think that no log should be printed in a release version. But in order to collect errors of apps and feedback for abnormalities, necessary logs should be printed. So long as it complies with secure encoding schemes, the risk of that can be controlled.

Log.e () in/w ()/i (): recommended to print operation log 

Log.d ()/v (): recommended to print Development Logs 

1 use System.out/err, instead of Log.e()/w()/i() to print sensitive information.

2 it's recommended to print some sensitive information by using Log.d ()/v (). 
(Pre-requisite: it will be automatically removed in release version)


    @Override
    public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_proguard);
    // *** POINT 1 *** Sensitive information must not be output by Log.e()/w()/i(), System.out/err.
    Log.e(LOG_TAG, "Not sensitive information (ERROR)");
    Log.w(LOG_TAG, "Not sensitive information (WARN)");
    Log.i(LOG_TAG, "Not sensitive information (INFO)");
    // *** POINT 2 *** Sensitive information should be output by Log.d()/v() in case of need.
    // *** POINT 3 *** The return value of Log.d()/v()should not be used (with the purpose of substitution or comparison).
    Log.d(LOG_TAG, "sensitive information (DEBUG)");
    Log.v(LOG_TAG, "sensitive information (VERBOSE)");
    }


3. the returned value of Log.d ()/v () should not be used. (Only for development debugging observation)

Examination code which Log.v() that is specifeied to be deleted is not deketed


    int i = android.util.Log.v("tag", "message");
    System.out.println(String.format("Log.v() returned %d. ", i)); //Use the returned value of Log.v() for    examination


4. released verson of apps should automatically remove codes like Log.d()/v().

Configuring ProGuard in eclipse

![image](https://quip.com/blob/QefAAAOvlFp/B7AtqPhu75Zc02K7c7SAkQ?s=WfhYAqYJunaN)

All logs are printed out in development versions.

![image](https://quip.com/blob/QefAAAOvlFp/PBK5__3PKvhwUC1RmFm5nQ?s=WfhYAqYJunaN)

Release version of ProGuard removed logs of d/v.

![image](https://quip.com/blob/QefAAAOvlFp/j1JzEvEBtGb5Jezko--cwQ?s=WfhYAqYJunaN)

Viewing in decompilers, the logs are indeed removed.

![image](https://quip.com/blob/QefAAAOvlFp/T72S5YyCezUTVCZcNKbqSA?s=WfhYAqYJunaN)

5. APK files should be the release version and not the development version when going to public.

#0x04 native code
-----
The constructor of android.util.Log is private and will not be instantiated, which only provides static properties and methods.  However, android.util.log relies on native code printIn_native() to record logs.Log.v()/Log.d()/Log.i()/Log.w()/Log.e() all eventually call println_native().

Log.e(String tag, String msg)


    public static int v(String tag, String msg) {
        return println_native(LOG_ID_MAIN, VERBOSE, tag, msg);
    }


println_native(LOG_ID_MAIN, VERBOSE, tag, msg)

    /*
     * In class android.util.Log:
     *  public static native int println_native(int buffer, int priority, String tag, String msg)
     */
    static jint android_util_Log_println_native(JNIEnv* env, jobject clazz,
        jint bufID, jint priority, jstring tagObj, jstring msgObj)
    {
    const char* tag = NULL;
    const char* msg = NULL;
    
    if (msgObj == NULL) {
        jniThrowNullPointerException(env, "println needs a message");
        return -1;
    }
    
    if (bufID < 0 || bufID >= LOG_ID_MAX) {
        jniThrowNullPointerException(env, "bad bufID");
        return -1;
    }
    
    if (tagObj != NULL)
        tag = env->GetStringUTFChars(tagObj, NULL);
    msg = env->GetStringUTFChars(msgObj, NULL);
    
    int res = __android_log_buf_write(bufID, (android_LogPriority)priority, tag, msg);
    
    if (tag != NULL)
        env->ReleaseStringUTFChars(tagObj, tag);
    env->ReleaseStringUTFChars(msgObj, msg);
    
    return res;
    }

__Android_log_buf_write () called   write_to_log again.

    static int __write_to_log_init(log_id_t log_id, struct iovec *vec, size_t nr)
    {
    #ifdef HAVE_PTHREADS
        pthread_mutex_lock(&log_init_lock);
    #endif
    
        if (write_to_log == __write_to_log_init) {
            log_fds[LOG_ID_MAIN] = log_open("/dev/"LOGGER_LOG_MAIN, O_WRONLY);
            log_fds[LOG_ID_RADIO] = log_open("/dev/"LOGGER_LOG_RADIO, O_WRONLY);
            log_fds[LOG_ID_EVENTS] = log_open("/dev/"LOGGER_LOG_EVENTS, O_WRONLY);
            log_fds[LOG_ID_SYSTEM] = log_open("/dev/"LOGGER_LOG_SYSTEM, O_WRONLY);
    
            write_to_log = __write_to_log_kernel;
    
            if (log_fds[LOG_ID_MAIN] < 0 || log_fds[LOG_ID_RADIO] < 0 ||
                log_fds[LOG_ID_EVENTS] < 0) {
                log_close(log_fds[LOG_ID_MAIN]);
                log_close(log_fds[LOG_ID_RADIO]);
                log_close(log_fds[LOG_ID_EVENTS]);
                log_fds[LOG_ID_MAIN] = -1;
                log_fds[LOG_ID_RADIO] = -1;
                log_fds[LOG_ID_EVENTS] = -1;
                write_to_log = __write_to_log_null;
            }
    
            if (log_fds[LOG_ID_SYSTEM] < 0) {
                log_fds[LOG_ID_SYSTEM] = log_fds[LOG_ID_MAIN];
            }
        }
    
    #ifdef HAVE_PTHREADS
        pthread_mutex_unlock(&log_init_lock);
    #endif
    
        return write_to_log(log_id, vec, nr);
    }

总的来说println_native()的操作就是打开设备文件然后写入数据。

#0x05 Something else
-----
1. use Log.d ()/v () to print the abnormal object. (For example, SQLiteException can lead to SQL injection issues)

2. use android.util.Log class to output logs. the use of System.out/err is not recommended.

3.  the version of BuildConfig.DEBUG ADT is later than 21

```
public final static boolean DEBUG = true;
```

In the release version, this will automatically be set to false

```
if (BuildConfig.DEBUG) android.util.Log.d(TAG, "Log output information");
```

4. when the Activity is started, ActivityManager will output intent information as follows:

* The target package name
* The target class name
* Intent.setData (URL) URL

![image](https://quip.com/blob/QefAAAOvlFp/rQsEHBwZloPRmDnwAw7qGw?s=WfhYAqYJunaN)

5. related information can still be output even without using System.out/err , such as the use of Exception.printStackTrace ()

6. ProGuard cannot remove the following log: ("result:" + value).

```
Log.d(TAG, "result:" + value);
```

When encountering such situation you should use BulidConfig (note that the ADT version)

```
if (BuildConfig.DEBUG) Log.d(TAG, "result:" + value);
```

7. Don't output logs to sdscard, which would make logs  become globally-readable

#0x06 Wooyun Cases
-----
[WooYun: tuniu app logcat leaks uer's chat content in tuanchat](http://www.wooyun.org/bugs/wooyun-2014-079241)

[WooYun: Surfing browser Locat users SMS](http://www.wooyun.org/bugs/wooyun-2014-079357)

[WooYun: Bank of Hangzhou disclosure local Android client login account password information](http://www.wooyun.org/bugs/wooyun-2014-082717)

#0x07 log tools
-----

    import android.util.Log;  
     
    /** 
     * Log Management class 
     *  
     *  
     *  
     */
    public class L  
    {  
     
        private L()  
        {  
            /* cannot be instantiated */
            throw new UnsupportedOperationException("cannot be instantiated");  
        }  
     
        public static boolean isDebug = true;// Whether you want to print bug, in onCreate function application can initialize
        private static final String TAG = "way";  
      // Four of the following are default tag function
        public static void i(String msg)  
        {  
            if (isDebug)  
                Log.i(TAG, msg);  
        }  
     
        public static void d(String msg)  
        {  
            if (isDebug)  
                Log.d(TAG, msg);  
        }  
     
        public static void e(String msg)  
        {  
            if (isDebug)  
                Log.e(TAG, msg);  
        }  
     
        public static void v(String msg)  
        {  
            if (isDebug)  
                Log.v(TAG, msg);  
        }  
     // the following is a function of incoming custom tag 
        public static void i(String tag, String msg)  
        {  
            if (isDebug)  
                Log.i(tag, msg);  
        }  
     
        public static void d(String tag, String msg)  
        {  
            if (isDebug)  
                Log.i(tag, msg);  
        }  
     
        public static void e(String tag, String msg)  
        {  
            if (isDebug)  
                Log.i(tag, msg);  
        }  
     
        public static void v(String tag, String msg)  
        {  
            if (isDebug)  
                Log.i(tag, msg);  
        }  
    }

#0x08 reference

-----

http://www.jssec.org/dl/android_securecoding_en.pdf

http://source.android.com/source/code-style.html#log-sparingly

http://developer.android.com/intl/zh-cn/reference/android/util/Log.html

http://developer.android.com/intl/zh-cn/tools/debugging/debugging-log.html

http://developer.android.com/intl/zh-cn/tools/help/proguard.html

https://www.securecoding.cert.org/confluence/display/java/DRD04-J.+Do+not+log+sensitive+information

https://android.googlesource.com/platform/frameworks/base.git/+/android-4.2.2_r1/core/jni/android_util_Log.cpp