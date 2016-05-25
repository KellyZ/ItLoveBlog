研究众多的android http库之后，我们会发现可以用如下图表示：

![](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/httplibs.png)

Retrofit - 一个类型安全的HTTP客户端支持REST接口

Picasso - 针对Android的图片下载与缓存库

Retrofit和Picasso都是移动支付公司Square的开源作品，目前已经广泛应用于New York Times，Paypay，Ouya，Spotify和更多应用中。

## Retrofit

官方向导：https://futurestud.io/blog/retrofit-2-upgrade-guide-from-1-9

包含下列丰富的示例：

1. [Getting Started and Create an Android Client](https://futurestud.io/blog/retrofit-getting-started-and-android-client)
2. [Basic Authentication on Android](https://futurestud.io/blog/android-basic-authentication-with-retrofit)
3. [Token Authentication on Android](https://futurestud.io/blog/retrofit-token-authentication-on-android)
4. [OAuth on Android](https://futurestud.io/blog/oauth-2-on-android-with-retrofit)
5. [Multiple Query Parameters of Same Name](https://futurestud.io/blog/retrofit-multiple-query-parameters-of-same-name)
6. [Synchronous and Asynchronous Requests](https://futurestud.io/blog/retrofit-synchronous-and-asynchronous-requests)
7. [Send Objects in Request Body](https://futurestud.io/blog/retrofit-send-objects-in-request-body)
8. [Define a Custom Response Converter](https://futurestud.io/blog/retrofit-replace-the-integrated-json-converter)
9. [Add Custom Request Header](https://futurestud.io/blog/retrofit-add-custom-request-header)
10. [Optional Query Parameters](https://futurestud.io/blog/retrofit-optional-query-parameters)
11. [How to Integrate XML Converter](https://futurestud.io/blog/retrofit-how-to-integrate-xml-converter)
12. [Using the Log Level to Debug Requests](https://futurestud.io/blog/retrofit-using-the-log-level-to-debug-requests)
13. [How to Upload Files to Server](https://futurestud.io/blog/retrofit-how-to-upload-files)
14. [Series Round-Up](https://futurestud.io/blog/retrofit-series-round-up)
15. Retrofit 2 — Upgrade Guide from 1.9
16. [Retrofit 2 — How to Upload Files to Server](https://futurestud.io/blog/retrofit-2-how-to-upload-files-to-server)
17. [Retrofit 2 — Log Requests and Responses](https://futurestud.io/blog/retrofit-2-log-requests-and-responses)
18. [Retrofit 2 — Hawk Authentication on Android](https://futurestud.io/blog/retrofit-2-hawk-authentication-on-android)
19. [Retrofit 2 — Simple Error Handling](https://futurestud.io/blog/retrofit-2-simple-error-handling)
20. [How to use OkHttp 3 with Retrofit 1](https://futurestud.io/blog/retrofit-how-to-use-okhttp-3-with-retrofit-1)
21. [Retrofit 2 — Book Update & Release Celebration](https://futurestud.io/blog/retrofit-2-book-update-release-celebration)
22. [Send Data Form-Urlencoded](https://futurestud.io/blog/retrofit-send-data-form-urlencoded)
23. [Send Data Form-Urlencoded Using FieldMap](https://futurestud.io/blog/retrofit-send-data-form-urlencoded-using-fieldmap)
24. [Retrofit 2 — Manage Request Headers in OkHttp Interceptor](https://futurestud.io/blog/retrofit-2-manage-request-headers-in-okhttp-interceptor)
25. [Retrofit 2 — How to Add Query Parameters to Every Request](https://futurestud.io/blog/retrofit-2-how-to-add-query-parameters-to-every-request)
26. [Retrofit 2 — Add Multiple Query Parameter With QueryMap](https://futurestud.io/blog/retrofit-2-add-multiple-query-parameter-with-querymap)
27. [Retrofit 2 — How to Use Dynamic Urls for Requests](https://futurestud.io/blog/retrofit-2-how-to-use-dynamic-urls-for-requests)
28. [Retrofit 2 — Url Handling, Resolution and Parsing](https://futurestud.io/blog/retrofit-2-url-handling-resolution-and-parsing)
29. [Retrofit 2 — Constant, Default and Logic Values for POST and PUT Requests](https://futurestud.io/blog/retrofit-2-constant-default-and-logic-values-for-post-and-put-requests)
30. [Retrofit 2 — How to Download Files from Server](https://futurestud.io/blog/retrofit-2-how-to-download-files-from-server)
31. [Retrofit 2 — Cancel Requests](https://futurestud.io/blog/retrofit-2-cancel-requests)
32. [Retrofit 2 — Reuse and Analyze Requests](https://futurestud.io/blog/retrofit-2-reuse-and-analyze-requests-2)
33. [Retrofit 2 — How to Change API Base Url at Runtime](https://futurestud.io/blog/retrofit-2-how-to-change-api-base-url-at-runtime-2)
34. [Optional Path Parameters](https://futurestud.io/blog/retrofit-optional-path-parameters)
35. How to Refresh an Access Token
36. [Retrofit 2 — How to Send Plain Text Request Body](https://futurestud.io/blog/retrofit-2-how-to-send-plain-text-request-body)
37. Retrofit 2 — Pagination Using Query Parameter
38. Retrofit 2 — Pagination Using Link Header and Dynamic Urls (Like GitHub)
39. Retrofit 2 — Pagination Using Range Header Fields (Like Heroku)
40. [Retrofit 2 — Introduction to (Multiple) Converters](https://futurestud.io/blog/retrofit-2-introduction-to-multiple-converters)
41. Retrofit 2 — Adding & Customizing the Gson Converter
42. Retrofit 2 — Implementing Custom Converters
43. Retrofit 2 — Enable Logging for Development Builds Only
44. Retrofit 2 — How to Upload Multiple Files to Server
45. Retrofit 2 — Passing Multiple Parts Along a File with @PartMap
46. Retrofit 2 — Basics of Mocking Server Responses
47. Retrofit 2 — Customizing Network Behavior of Mocked Server Responses


## Picasso使用

Picasso不仅实现了图片异步加载的功能，还解决了android中加载图片时需要解决的一些常见问题：

1. 在adapter中需要取消已经不在视野范围的ImageView图片资源的加载，否则会导致图片错位，Picasso已经解决了这个问题。
2. 使用复杂的图片压缩转换来尽可能的减少内存消耗
3. 自带内存和硬盘二级缓存功能

> ADAPTER 中的下载：Adapter的重用会被自动检测到，Picasso会取消上次的加载

    @Override public void getView(int position, View convertView, ViewGroup parent) {
    SquaredImageView view = (SquaredImageView) convertView;
        if (view == null) {
        view = new SquaredImageView(context);
        }
        String url = getItem(position);
        Picasso.with(context).load(url).into(view);
    }

> 图片转换：转换图片以适应布局大小并减少内存占用

    Picasso.with(context)
      .load(url)
      .resize(50, 50)
      .centerCrop()
      .into(imageView);

> Place holders-空白或者错误占位图片：picasso提供了两种占位图片，未加载完成或者加载发生错误的时需要一张图片作为提示。

    Picasso.with(context)
    .load(url)
    .placeholder(R.drawable.user_placeholder)
    .error(R.drawable.user_placeholder_error)
    .into(imageView);
    
   如果加载发生错误会重复三次请求，三次都失败才会显示erro Place holder

> 资源文件的加载：除了加载网络图片picasso还支持加载Resources, assets, files, content providers中的资源文件。

    Picasso.with(context).load(R.drawable.landing_screen).into(imageView1);
    Picasso.with(context).load(new File(...)).into(imageView2);

## Picasso原理

参考http://blog.csdn.net/xu_fu/article/details/17043231

## 参考

1. https://futurestud.io/blog/retrofit-2-upgrade-guide-from-1-9
2. http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2014/0731/1639.html
3. https://packetzoom.com/blog/which-android-http-library-to-use.html