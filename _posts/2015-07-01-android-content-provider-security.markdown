---
published: true
title: Android Content Provider Security
layout: post
---
Author:[瘦蛟舞](http://drops.wooyun.org/author/瘦蛟舞)

    Drops is one of the greatest platforms for security-related technical blogs in China. We dedicate to translate every amazing post into English and push it to this website, meanwhile we would be very honored if you could submit your atricles at drops@wooyun.org.

Translated from：http://drops.wooyun.org/tips/4314

#0x00 Background Knowledge

------

Content providers manage access to a structured set of data. They encapsulate the data which can be accessed by any application. Other than Public Storage Area which is accessible by all android app packages, they are the only method for applications to share data. Android itself includes content providers that manage data such as audio, video, images, and personal contact information.   You can see some of them listed in the reference documentation for the android.provider package. You can also query the data which content providers include. Of course, certain sensitive content provider can only be accessed with corresponding permission.

If you want to expose your own data, you've got two choices: you can create your own content provider (a ContentProvider subclass) or you can add data to existing providers, if you can read/write a content provider that controls the same type of the data.

![image](https://quip.com/blob/dXSAAALLrmV/qpWEIh-4NrEPS_bczvOAKg?s=A8ncAE5UDw1l)

#0x01 Key points

----------

Reference:http://developer.Android.com/guide/topics/providers/content-providers.HTML 

**Content URIs**

A content URI is a URI that identifies data in a provider.  Content URIs include the symbolic name of the entire provider (its authority) and a name that points to a table (a path). When you call a client method to access a table in a provider, the content URI for the table is one of the arguments.

![image](https://quip.com/blob/dXSAAALLrmV/WPGGdBNGsxLgXJ17VCpamg?s=A8ncAE5UDw1l)

A. standard prefix, indicates that the data is controlled by a content provider. It will not be modified.

B. permission portion of the URI; it identifies the content provider. For third-party applications, this should be a full class name (lowercase) to ensure uniqueness. Permissions are declared in the permission attributes of the element:

     <provider name=".TransportationProvider" authorities="com.example.transportationprovider" . . . >

C. used to determine the path of the requested data type. This can be 0 or more segment length. If the content provider exposes only one data type (for example, only trains), the segment can be missed. If the provider exposes more data types, including subclass type, it can be several segment lengths, for instance, to provide the values of "land/bus", "land/train", "sea/ship" and "sea/submarine".

D. is the requested ID of a particular record, if there is. This is the _ID of the requested record. If the request is not limited to a single record, this segment and the following slash are ignored:

    content://com.example.transportationprovider/trains

**ContentResolver**

The methods of ContentProvider provide basic "CRUD" (Create-Read-Update-Delete) functions to data storage.

    getIContentProvider() 
          Returns the Binder object for this provider.
     
    delete(Uri uri, String selection, String[] selectionArgs) -----abstract
          A request to delete one or more rows.
     
    insert(Uri uri, ContentValues values) 
          Implement this to insert a new row.
     
    query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) 
          Receives a query request from a client in a local process, and returns a Cursor.
     
    update(Uri uri, ContentValues values, String selection, String[] selectionArgs) 
          Update a content URI.
     
    openFile(Uri uri, String mode) 
          Open a file blob associated with a content URI.

**Sql injection**

SQL statement mosaic

    //construct an optional clause with a placeholder 
    String mSelectionClause = "var =?";

**Permissions**

The following  element requests read access to the User Dictionary Provider:

<uses-permission android:name="android.permission.READ_USER_DICTIONARY">

Request some permissions with protection level="dangerous"

    <uses-permission android:name="com.huawei.dbank.v7.provider.DBank.READ_DATABASE"/>
    <permission android:name="com.huawei.dbank.v7.provider.DBank.READ_DATABASE" android:protectionLevel="dangerous"></permission>

android:protectionLevel

Normal: The default value.  A lower-risk permission that gives requesting applications access to isolated application-level features, with minimal risk to other applications, the system, or the user. The system automatically grants this type of permission to a requesting application at installation, without asking for the user's explicit approval.

Dangerous: WRITE_SETTING and SEND_SMS permissions are higher-risk permissions, because these permissions can be used to reconfigure the device, or lead charges. This protectionLevel is used to flag the permissions which the user needs to pay attention to. Android will warn the user about the requirements of such permissions. Specific acts may vary according to Android versions or the installed mobile devices.

Signature: A permission that the system grants only if the requesting application is signed with the same certificate as the application that declared the permission.

SignatureOrSystem: similar to signature class, except for the fact that applications in Android image also require permission. In this case, applications that vendors build in system can still be granted with permission.  By doing so, customized Android can be granted permissions. This protection level is helpful to integrate system compiling process.

**API**

The exported attribute of Contentprovider has been changed from true to false in API-17（android4.2）and later versions.

Contentprovider cannot be declared as private in android2.2 (API-8) .

    <!-- *** POINT 1 *** Do not (Cannot) implement Private Content Provider in Android 2.2 (API Level 8) or earlier. -->
    <uses-sdk android:minSdkVersion="9" android:targetSdkVersion="17" />

**Key methods**

* public void addURI (String authority, String path, int code)
* public static String decode (String s)
* public ContentResolver getContentResolver()
* public static Uri parse(String uriString)
* public ParcelFileDescriptor openFile (Uri uri, String mode)
* public final Cursor query(Uri uri, String[] projection,String selection, String[] selectionArgs, String sortOrder)
* public final int update(Uri uri, ContentValues values, String where,String[] selectionArgs)
* public final int delete(Uri url, String where, String[] selectionArgs)
* public final Uri insert(Uri url, ContentValues values)

#0x02 Types of content provider

-----------

![image](https://quip.com/blob/dXSAAALLrmV/Ggf6JX27LJiXZne6i6L0kw?s=A8ncAE5UDw1l)

![image](https://quip.com/blob/dXSAAALLrmV/byeodIlL8t1DTT10AbEfMA?s=A8ncAE5UDw1l)

Personally, I think private、public、in-house are enough.

#0x03 Security tips

-------------

1. Set MinSdkVersion to 9 or higher
2. If this content provider doesn't provide private data to external apps, set the attribute of exported to false.
3. Use parameterized queries to avoid injection
4. Internal apps use content providers to exchange data. Set protectionLevel=“signature” to verify the signature.
5. Make sure Public content provider doesn't store sensitive data.
6. Uri.decode() before use ContentProvider.openFile()
7. Protect permissions when providing asset files

#0x04 Test methods

-------------


1, Decompile and view the AndroidManifest.xml (drozer scan) to locate its content provider statement. Check if content provider is exported, permissions are configured, and its authority is determined.

    drozer: run app.provider.info -a cn.etouch.ecalendar

2, Decompile and find path, keywordaddURI, hook API. zjdroid is recommended to be used in dynamic monitoring.

3, After determining the authority and path, use Content Provider Helper or adb shell to write POC based on business preparation of //report error if there's no corresponding permission.

    adb shell：
    adb shell content query --uri <URI> [--user <USER_ID>] [--projection <PROJECTION>] [--where <WHERE>] [--sort <SORT_ORDER>]
    content query --uri content://settings/secure --projection name:value --where "name='new_setting'" --sort "name ASC"
    adb shell content insert --uri content://settings/secure --bind name:s:new_setting --bind value:s:new_value
    adb shell content update --uri content://settings/secure --bind value:s:newer_value --where "name='new_setting'"
    adb shell content delete --uri content://settings/secure --where "name='new_setting'"
    
    drozer：
    run app.provider.query content://telephony/carriers/preferapn --vertical

#0x05 Cases

-------------

**Case 1: direct expose**

* WooYun: Shengda Youni for Android leaking  sensitive information (read user's local message) (http://www.wooyun.org/bugs/wooyun-2013-041595)
* WooYun: SINA weibo for Android leaking local message  (http://www.wooyun.org/bugs/wooyun-2013-016854)
* WooYun: Shengda Qidian reader for Android leaking user's sensitive information, such as client token (http://www.wooyun.org/bugs/wooyun-2013-021089)
* WooYun: Maxthon browser limitations are not strict, whcih can leading Web fraud attacks (http://www.wooyun.org/bugs/wooyun-2013-039290)
* WooYun: sogou browser for Phone leaking privacy and containing homepage tampering vulnerability (malware has to install on the mobile phone) (http://www.wooyun.org/bugs/wooyun-2013-042609)
* WooYun: Coolpad S6 bypass monitoring to sneakly output traffic (http://www.wooyun.org/bugs/wooyun-2014-085432)
* WooYun: Coolpad S6 bypass notification bar management permission  (http://www.wooyun.org/bugs/wooyun-2014-084500)

**Case 2: requires permissions to access **

* WooYun: Michat for Android leaking sensitive information(read local message) (http://www.wooyun.org/bugs/wooyun-2013-041521)
* WooYun: Huawei Netdisk content provider components may leak user information (http://www.wooyun.org/bugs/wooyun-2014-057590)
* WooYun: Renren client permissions problems leaking privacy (http://www.wooyun.org/bugs/wooyun-2013-039697)

**Case 3:openFile file traverse**

* WooYun: Ganji Android client's Content Provider module has arbitrary file access vulnerability (http://www.wooyun.org/bugs/wooyun-2013-044407)
* WooYun: Liebao browser's (Android Edition) any private file can be accessed through local third-party theft vulnerability (http://www.wooyun.org/bugs/wooyun-2013-047098)
* WooYun: 58 Android client-side remote file write vulnerability (http://www.wooyun.org/bugs/wooyun-2013-044411)

Override openFile method

Error writing 1:

    private static String IMAGE_DIRECTORY = localFile.getAbsolutePath();
    public ParcelFileDescriptor openFile(Uri paramUri, String paramString)
        throws FileNotFoundException {
      File file = new File(IMAGE_DIRECTORY, paramUri.getLastPathSegment());
      return ParcelFileDescriptor.open(file, ParcelFileDescriptor.MODE_READ_ONLY);
    }

Error writing 2:URI.parse ()

    private static String IMAGE_DIRECTORY = localFile.getAbsolutePath();
    public ParcelFileDescriptor openFile(Uri paramUri, String paramString)
        throws FileNotFoundException {
        File file = new File(IMAGE_DIRECTORY, Uri.parse(paramUri.getLastPathSegment()).getLastPathSegment());
        return ParcelFileDescriptor.open(file, ParcelFileDescriptor.MODE_READ_ONLY);
    }

POC1：

    String target = "content://com.example.android.sdk.imageprovider/data/" + "..%2F..%2F..%2Fdata%2Fdata%2Fcom.example.android.app%2Fshared_prefs%2FExample.xml";
     
    ContentResolver cr = this.getContentResolver();
    FileInputStream fis = (FileInputStream)cr.openInputStream(Uri.parse(target));
     
    byte[] buff = new byte[fis.available()];
    in.read(buff);

POC2：double encode

    String target = "content://com.example.android.sdk.imageprovider/data/" + "%252E%252E%252F%252E%252E%252F%252E%252E%252Fdata%252Fdata%252Fcom.example.android.app%252Fshared_prefs%252FExample.xml";
     
    ContentResolver cr = this.getContentResolver();
    FileInputStream fis = (FileInputStream)cr.openInputStream(Uri.parse(target));
     
    byte[] buff = new byte[fis.available()];
    in.read(buff);

Solution Uri.decode ()

    private static String IMAGE_DIRECTORY = localFile.getAbsolutePath();
      public ParcelFileDescriptor openFile(Uri paramUri, String paramString)
          throws FileNotFoundException {
        String decodedUriString = Uri.decode(paramUri.toString());
        File file = new File(IMAGE_DIRECTORY, Uri.parse(decodedUriString).getLastPathSegment());
        if (file.getCanonicalPath().indexOf(localFile.getCanonicalPath()) != 0) {
          throw new IllegalArgumentException();
        }
        return ParcelFileDescriptor.open(file, ParcelFileDescriptor.MODE_READ_ONLY);
      }

#0x06 reference

------------

https://www.securecoding.cert.org/confluence/pages/viewpage.action?pageId=111509535

http://www.jssec.org/dl/android_securecoding_en.pdf

http://developer.android.com/intl/zh-cn/reference/android/content/ContentProvider.html

#0x07 related reading

---------------

http://zone.wooyun.org/content/15097

http://drops.wooyun.org/tips/2997
