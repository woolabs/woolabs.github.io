---
published: true
title: Research into a new file inclusion attack
layout: post
---
Author:[phith0n](http://drops.wooyun.org/author/phith0n)

    Drops is one of the greatest platforms for security-related technical blogs in China. We dedicate to translate every amazing post into English and push it to this website, meanwhile we would be very honored if you could submit your atricles at drops@wooyun.org.

Origin：http://drops.wooyun.org/papers/5040

#0x00 Preface


Previously, @boooom exposed several File Inclusion Attacks on [wooyun.org](http://wooyun.org/)  in form of this:

```
http://target/../../../../etc/passwd
```

At that time I was quite curioufs about this problem, for the reason that the sever-side middleware is typically not allowed to directly access files out of web directories. What confuses me is how could such vulnerabilities appear in many cases.

Later @lijiejie explained the reason in an article: http://www.lijiejie.com/Python-Django-directory-traversal/. It turns out that Pyhon, a new web development method, caused this problem. After knowing that, I examined some of the applications that I had developed by using web.py and tornado. With no exception, these applications have the same problem.

As @lijiejie mentioned, on the one hand, the early django framework itself contains some vulnerabilities. One the other hand, it is developer's mistake that results in this vulnerability.

In this case, it is necessary to distinguish the development frameworks that we use nowadays and those that we used before.

Whether it's in Python, node or ruby, the URL assignment is allowed to be customized. However, in frameworks like php or asp, a file can only be requested based on a directory structure. All requests have to comply with the rules defined by users, and the kernel part of the framework is responsible for reading data, controling user input and sending data to the model. For example, as we request URL "/login/", this URL is likely to be assigned to a LoginHandler class, rather than a request to the/login/index.php.

But this brings another problem, what should we do if all we need is to request a real file such as static files like css and js ?

Generally there will be some distinctions. Some advanced applications mostly adopt CDN , load  nginx as a balance processor to work on load distribution. If the requested URL is a static file, CDN or nginx would directly return the corresponding file, which is showed in the following picture:

![image](https://quip.com/-/blob/CJVAAAAPn1i/5niukYD42ydhC_5Mc2_ZBQ)

There is also a question in definition. What sorts of request can be viewed as a request for "static files"? As long as the ending is .css、.js ? Of course, this is a possible way, but generally we need define a directory, such as /static/. All requests matching "/static/(. *)" will be considered a static file, so developers usually place static files in this directory. Thus, users can easily request these files.

If there is no platforms like CDN or niginx,  frameworks such as web.py and tornado also define static directories. For web.py, h,  /static/ is the static directory by default, in which requests will not pass URLPath router. The documentation of Web.py says:

http://webpy.org/cookbook/staticfiles.zh-cn

I wondered since the framework uses /static/(. *) to match a request, if we could request

`/static/../../../../../etc/passwd`

to access files in /etcs/passwd?

#0x01 Research into the File Incusion Attacks in web.py


First, let's check out how web.py deal with such a request:

`/static/../../../../../etc/passwd`

We can see the following code in Web/httpserver.py:

![image](https://quip.com/-/blob/CJVAAAAPn1i/dLTlUtIxUMlq2ySfX8wBEg)

If a request starts with /static/, then SimpleHTTPServer would process the request. SimpleHTTPServer is a simple HTTP Server embeded in Python. We could perform 

```
python -m SimpleHTTPServer
```

in arbitrary directories to start a Web server. And you can directly access the files in this directory through the HTTP protocol.

In fact, the server of web.py is the inheritance and ovewrite of a SimpleHTTPServer, whicn simply sends this request to a parent-class SimpleHTTPServer to process. Of course, the HTTP Server only allows a request to a Web directory (i.e./static/) , so it got an response error 404:

![image](https://quip.com/-/blob/CJVAAAAPn1i/rWLu6oiQc07jvM49HPR-kg)

The framework makes sure the static files cannot be arbitrarily accessed. But for applications with complex logic, developers often are not content with just setting one /static/ directory. For instance, when encountering a website which allows users to upload and download files, maybe we'll create a UploadFile directory to save the uploaded files by date and time.

Therefore, in order to make files under /UploadFile directory accessible, developers typically would write:

![image](https://quip.com/-/blob/CJVAAAAPn1i/70QBHb1iYzQhG5hophgduA)

There is a download class exclusively resolving such requests, which can read the file through GET method and write it into the HTTP packets as a response.

When we request for a normal file/UploadFile/01.txt, you would get its content:

![image](https://quip.com/-/blob/CJVAAAAPn1i/Ach2KpMaAw-uCCO0Qt3XYQ)

But we can also request a file not under /UploadFile/ directory, which leads to a File Inclusion Attack.

![image](https://quip.com/-/blob/CJVAAAAPn1i/HzuBhdVlSlohB9LH0HxkVg)

This is due to developer's mistake. The developer failed to check if the uploaded path is legal. There's nothing wrong with the framework.

#0x02 possible File Inclusion Attacks in Tornado

Tornado is a completely asynchronous web framework,which allows us to define a static directory-static_path in configuration . Tornado provides a method specifically to verify if our request is in permitted directories:

![image](https://quip.com/-/blob/CJVAAAAPn1i/sZYoZhXF7JpTLX9neM6Oig)

In this way, if the requests are not in our defined static directories, Tornado would return "is not in root static directory" error:

![image](https://quip.com/-/blob/CJVAAAAPn1i/gbuSrymSEx3h-hjodnprew)

Then,what should we do if we need to define a"/UploadFile/" as user's upload directory?

As mentioned in the document, we only need to customize a URLPath. There is a static file controller-web.StaticFileHandler inside Tornado to do this work.

![image](https://quip.com/-/blob/CJVAAAAPn1i/UZCHWcU__7zkMp5BhTdQFA)

Even without considering the security problem, we still are not allowed to use synchronous I/O functions such as *read* and *write *to process these static files in an asynchronous framework.

However, in a later study, I discovered the way that Tornado used is not perfect, more details will be available after the vulnerability is released: [WooYun: a flaw in Tornado could cause file access vulnerability](http://www.wooyun.org/bugs/wooyun-2015-098978)

#0x03 problems in Django


The older versions of Django contain flaws that may cause file access vulnerability. The reason for that is older Django didn't check static files as I mentioned, like the BUG existed in 2009:

https://bugzilla.redhat.com/show_bug.cgi?id=CVE-2009-2659

![image](https://quip.com/-/blob/CJVAAAAPn1i/2QE79lhXksr-z0cvxK1BBA)


As I said in Web.py, if Django is simply using *open*, *read *to read files without checking the legality of PATH, then it may result in File Inclusion Attacks.

For instance, we write a Django view as follow:

![image](https://quip.com/-/blob/CJVAAAAPn1i/lSyXLv72YHLlbgm6xXL59A)

Files can still be File Inclusion Attacks without checking the legality of PATH:

![image](https://quip.com/-/blob/CJVAAAAPn1i/6X2TKCv1PRw6iP66hEf_jg)

As indicated in the figure above, the contents of sqlite database are directly accessed, the administrative account and password leaked.

So how could we prevent a File Inclusion Attack in such an application? All we need is to simply modify the Django view. afterwards, we would defense the vulnerability :

![image](https://quip.com/-/blob/CJVAAAAPn1i/zPmfX-ZCEwvEiJH6jsYuHg)

Actually some frameworks have implemented its unique way to check static files. Django should have done the same. Some light-weighted framework, such as web.py, would not check its own functions. In this case, you could use the above method to remove illegal static file directories.

#0x04 Do languages such as PHP have the same problem?

This issue is indeed worth thinking.

As I said in the beginning, modern Web development frameworks including Python/node/Ruby are different from PHP, ASP and other languages. Because of URL reading data, static files can only be accessed by defining, which is one of the cases of this vulnerability. Once the access parameter is not checked, it will result in arbitrary file access problem.

However, the difference lies in the fact that static files are accessed by directories instead of through php in traditional PHP applications.

Let's put aside if our request will get through php. Here is a real example in the latest version of zblog.

`/zb_system/function/c_system_event.php`

About 415 lines:

![image](https://quip.com/-/blob/CJVAAAAPn1i/gslq6n0DwCZ9Ef20S8TlrA)

Determine if

`$_SERVER['SERVER_SOFTWARE']`

contains Microsoft-IIS. If server-side middleware is IIS, and

`$_GET['rewrite']`

, then enter the if flow. After that, Determine whether the $URL points to a existing file. If the file exists, then use *file_get_contents* to read it and display it. This code actually does the same work as the python code we mentioned above. Why PHP needs this kind of work, the files that our URL requests are not supposed to immediately returne to user by the webserver?

In fact, developers take the problems caused by url rewrite into consideration. The rewrite rules for zblog is to point all requests to index.php and finally use index.php to process the requests. Therefore, our static files could be rewritten into the index.php. The work we did here is to directly display those files.

Unfortunately, I am not very good at it. Besides, I do not know how to write rewrite rules to make static files be rewritten into the index.php. As a result, files cannot be arbitrarily accessed.

But we obviously can be found here, zblog developers did not check whether the $URL directory within the Web site, or whether static files within the directory, is directly displayed. So our request http://10.211.55.3/zblog/index.php?action=&rewrite=1 can read the source code of index.php (because I don't have IIS environment, I manually transform SERVER_SOFTWARE into IIS ~):

![image](https://quip.com/-/blob/CJVAAAAPn1i/4uSAvims-1YtCxe4X-y7Gg)

Therefore, the question that if traditional PHP applications contain such a security vulnerability, remains to be examined. In theory, if we write a rewrite rule like this: all requests need to be processed by index.php. Then, the functionalities of index.php are very similar to those of python framework's 

#0x05 conclusion

All in all, I just simply researched and discussed several python frameworks and an vulnerability in php. However, i'm not necessarily familiar with modern development techniques and enterprising environment. If there's any imperfection, please help me to supplement and correct.
