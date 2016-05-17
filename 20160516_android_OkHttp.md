## 历史

最早的时候Android只有两个主要的HTTP客户端： HttpURLConnection, Apache HTTP Client。根据Google官方博客的内容，HttpURLConnection在早期的Android版本中可能存在一些Bug:

> 在Froyo版本之前，HttpURLConnection包含了一些很恶心的错误。特别是对于关闭可读的InputStream时候可能会污染整个连接池。

同时Google官方认为Apache HTTP Client中的API过于复杂而不想用它，在最新的6.0里也去掉了。

但对于大部分普通开发者而言，它们觉得应该根据不同的版本使用不同的客户端。对于Gingerbread(2.3)以及之后的版本，HttpURLConnection会是最佳的选择，它的API更简单并且体积更小。透明压缩与数据缓存可以减少网络压力，提升速度并且能够节约电量。当我们审视Google Volley的源代码的时候，可以看得出来它也是根据不同的Android版本选择了不同的底层的网络请求库：

    if(stack == null) {
        if(VERSION.SDK_INT >= 9) {
            stack = new HurlStack();
        } else {
            stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
        }
    }
    
这样会很让开发者头疼，2013年，Square为了解决这种分裂的问题发布了OkHttp。OkHttp是直接架构与Java Socket本身而没有依赖于其他第三方库，因此开发者可以直接用在JVM中，而不仅仅是Android。为了简化代码迁移速度，OkHttp也实现了类似于HttpUrlConnection与Apache Client的接口。

![](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/1131875060-56fe1c5f8bf0e_articlex.png)

## 使用

> Synchronous Get

    private final OkHttpClient client = new OkHttpClient();

    public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();
    
    Response response = client.newCall(request).execute();
    if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);
    
    Headers responseHeaders = response.headers();
    for (int i = 0; i < responseHeaders.size(); i++) {
      System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
    }
    
    System.out.println(response.body().string());
    }

> Asynchronous Get

    private final OkHttpClient client = new OkHttpClient();

    public void run() throws Exception {
        Request request = new Request.Builder()
            .url("http://publicobject.com/helloworld.txt")
            .build();
        
        client.newCall(request).enqueue(new Callback() {
          @Override public void onFailure(Call call, IOException e) {
            e.printStackTrace();
          }
        
          @Override public void onResponse(Call call, Response response) throws IOException {
            if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);
        
            Headers responseHeaders = response.headers();
            for (int i = 0, size = responseHeaders.size(); i < size; i++) {
              System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
            }
        
            System.out.println(response.body().string());
          }
        });
    }
    
> Accessing Headers

    private final OkHttpClient client = new OkHttpClient();

    public void run() throws Exception {
        Request request = new Request.Builder()
            .url("https://api.github.com/repos/square/okhttp/issues")
            .header("User-Agent", "OkHttp Headers.java")
            .addHeader("Accept", "application/json; q=0.5")
            .addHeader("Accept", "application/vnd.github.v3+json")
            .build();
        
        Response response = client.newCall(request).execute();
        if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);
        
        System.out.println("Server: " + response.header("Server"));
        System.out.println("Date: " + response.header("Date"));
        System.out.println("Vary: " + response.headers("Vary"));
    }

> Posting a String

    public static final MediaType MEDIA_TYPE_MARKDOWN
          = MediaType.parse("text/x-markdown; charset=utf-8");
    
    private final OkHttpClient client = new OkHttpClient();
    
    public void run() throws Exception {
        String postBody = ""
            + "Releases\n"
            + "--------\n"
            + "\n"
            + " * _1.0_ May 6, 2013\n"
            + " * _1.1_ June 15, 2013\n"
            + " * _1.2_ August 11, 2013\n";
        
        Request request = new Request.Builder()
            .url("https://api.github.com/markdown/raw")
            .post(RequestBody.create(MEDIA_TYPE_MARKDOWN, postBody))
            .build();
        
        Response response = client.newCall(request).execute();
        if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);
        
        System.out.println(response.body().string());
    }

> Post Streaming

    public static final MediaType MEDIA_TYPE_MARKDOWN
      = MediaType.parse("text/x-markdown; charset=utf-8");

    private final OkHttpClient client = new OkHttpClient();
    
    public void run() throws Exception {
        RequestBody requestBody = new RequestBody() {
          @Override public MediaType contentType() {
            return MEDIA_TYPE_MARKDOWN;
          }
        
          @Override public void writeTo(BufferedSink sink) throws IOException {
            sink.writeUtf8("Numbers\n");
            sink.writeUtf8("-------\n");
            for (int i = 2; i <= 997; i++) {
              sink.writeUtf8(String.format(" * %s = %s\n", i, factor(i)));
            }
          }
        
          private String factor(int n) {
            for (int i = 2; i < n; i++) {
              int x = n / i;
              if (x * i == n) return factor(x) + " × " + i;
            }
            return Integer.toString(n);
          }
        };
        
        Request request = new Request.Builder()
            .url("https://api.github.com/markdown/raw")
            .post(requestBody)
            .build();
        
        Response response = client.newCall(request).execute();
        if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);
        
        System.out.println(response.body().string());
    }

> Posting a File

    public static final MediaType MEDIA_TYPE_MARKDOWN
      = MediaType.parse("text/x-markdown; charset=utf-8");

    private final OkHttpClient client = new OkHttpClient();
    
    public void run() throws Exception {
        File file = new File("README.md");
        
        Request request = new Request.Builder()
            .url("https://api.github.com/markdown/raw")
            .post(RequestBody.create(MEDIA_TYPE_MARKDOWN, file))
            .build();
        
        Response response = client.newCall(request).execute();
        if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);
        
        System.out.println(response.body().string());
    }

> Posting form parameters

    private final OkHttpClient client = new OkHttpClient();

    public void run() throws Exception {
        RequestBody formBody = new FormBody.Builder()
            .add("search", "Jurassic Park")
            .build();
        Request request = new Request.Builder()
            .url("https://en.wikipedia.org/w/index.php")
            .post(formBody)
            .build();
        
        Response response = client.newCall(request).execute();
        if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);
        
        System.out.println(response.body().string());
    }

> Posting a multipart request

    private static final String IMGUR_CLIENT_ID = "...";
    private static final MediaType MEDIA_TYPE_PNG = MediaType.parse("image/png");
    
    private final OkHttpClient client = new OkHttpClient();
    
    public void run() throws Exception {
        // Use the imgur image upload API as documented at https://api.imgur.com/endpoints/image
        RequestBody requestBody = new MultipartBody.Builder()
            .setType(MultipartBody.FORM)
            .addFormDataPart("title", "Square Logo")
            .addFormDataPart("image", "logo-square.png",
                RequestBody.create(MEDIA_TYPE_PNG, new File("website/static/logo-square.png")))
            .build();
        
        Request request = new Request.Builder()
            .header("Authorization", "Client-ID " + IMGUR_CLIENT_ID)
            .url("https://api.imgur.com/3/image")
            .post(requestBody)
            .build();
        
        Response response = client.newCall(request).execute();
        if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);
        
        System.out.println(response.body().string());
    }
    
> Parse a JSON Response With Gson

    private final OkHttpClient client = new OkHttpClient();
    private final Gson gson = new Gson();
    
    public void run() throws Exception {
        Request request = new Request.Builder()
            .url("https://api.github.com/gists/c2a7c39532239ff261be")
            .build();
        Response response = client.newCall(request).execute();
        if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);
        
        Gist gist = gson.fromJson(response.body().charStream(), Gist.class);
        for (Map.Entry<String, GistFile> entry : gist.files.entrySet()) {
          System.out.println(entry.getKey());
          System.out.println(entry.getValue().content);
        }
    }
    
    static class Gist {
        Map<String, GistFile> files;
    }
    
    static class GistFile {
        String content;
    }

> Response Caching

    private final OkHttpClient client;

    public CacheResponse(File cacheDirectory) throws Exception {
        int cacheSize = 10 * 1024 * 1024; // 10 MiB
        Cache cache = new Cache(cacheDirectory, cacheSize);
        
        client = new OkHttpClient.Builder()
            .cache(cache)
            .build();
    }
    
    public void run() throws Exception {
        Request request = new Request.Builder()
            .url("http://publicobject.com/helloworld.txt")
            .build();
        
        Response response1 = client.newCall(request).execute();
        if (!response1.isSuccessful()) throw new IOException("Unexpected code " + response1);
        
        String response1Body = response1.body().string();
        System.out.println("Response 1 response:          " + response1);
        System.out.println("Response 1 cache response:    " + response1.cacheResponse());
        System.out.println("Response 1 network response:  " + response1.networkResponse());
        
        Response response2 = client.newCall(request).execute();
        if (!response2.isSuccessful()) throw new IOException("Unexpected code " + response2);
        
        String response2Body = response2.body().string();
        System.out.println("Response 2 response:          " + response2);
        System.out.println("Response 2 cache response:    " + response2.cacheResponse());
        System.out.println("Response 2 network response:  " + response2.networkResponse());
        
        System.out.println("Response 2 equals Response 1? " + response1Body.equals(response2Body));
    }

> Canceling a Call

    private final ScheduledExecutorService executor = Executors.newScheduledThreadPool(1);
    private final OkHttpClient client = new OkHttpClient();
    
    public void run() throws Exception {
        Request request = new Request.Builder()
            .url("http://httpbin.org/delay/2") // This URL is served with a 2 second delay.
            .build();
        
        final long startNanos = System.nanoTime();
        final Call call = client.newCall(request);
        
        // Schedule a job to cancel the call in 1 second.
        executor.schedule(new Runnable() {
          @Override public void run() {
            System.out.printf("%.2f Canceling call.%n", (System.nanoTime() - startNanos) / 1e9f);
            call.cancel();
            System.out.printf("%.2f Canceled call.%n", (System.nanoTime() - startNanos) / 1e9f);
          }
        }, 1, TimeUnit.SECONDS);
        
        try {
          System.out.printf("%.2f Executing call.%n", (System.nanoTime() - startNanos) / 1e9f);
          Response response = call.execute();
          System.out.printf("%.2f Call was expected to fail, but completed: %s%n",
              (System.nanoTime() - startNanos) / 1e9f, response);
        } catch (IOException e) {
          System.out.printf("%.2f Call failed as expected: %s%n",
              (System.nanoTime() - startNanos) / 1e9f, e);
        }
    }

> Timeouts

    private final OkHttpClient client;

    public ConfigureTimeouts() throws Exception {
        client = new OkHttpClient.Builder()
            .connectTimeout(10, TimeUnit.SECONDS)
            .writeTimeout(10, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build();
    }
    
    public void run() throws Exception {
        Request request = new Request.Builder()
            .url("http://httpbin.org/delay/2") // This URL is served with a 2 second delay.
            .build();
        
        Response response = client.newCall(request).execute();
        System.out.println("Response completed: " + response);
    }
    
> Handling authentication

    private final OkHttpClient client;
    
    public Authenticate() {
        client = new OkHttpClient.Builder()
            .authenticator(new Authenticator() {
              @Override public Request authenticate(Route route, Response response) throws IOException {
                System.out.println("Authenticating for response: " + response);
                System.out.println("Challenges: " + response.challenges());
                String credential = Credentials.basic("jesse", "password1");
                return response.request().newBuilder()
                    .header("Authorization", credential)
                    .build();
              }
            })
            .build();
    }
    
    public void run() throws Exception {
        Request request = new Request.Builder()
            .url("http://publicobject.com/secrets/hellosecret.txt")
            .build();
        
        Response response = client.newCall(request).execute();
        if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);
        
        System.out.println(response.body().string());
    }

## 原理解析

一个HTTP库的任务就是接收Request并产生返回Response，接口简单明确，实践起来却有些复杂。

先来说说Request：每个Http request都包含URL、method、headers和body，其中变化因素比较多的就是headers和body了。

OkHttp在处理接收到的Request时就是添加相关的headers：User-Agent、Host、Connection、Transfer-Encoding、Content-Type、Content-Length、Accept-Encoding、Cookie等等，另外还有些如If-Modified-Since、If-None-Match用来获取更新的response。

另外，OkHttp还支持request重定向及authorization。

综上，rewrites、redirects、follow-ups（authorization)、retries这些对request的处理最终得到response，处理过程中可能会有多次的中间request，这些过程在OkHttp中统一封装成了Calls。

> Calls

Calls有同步和异步两种处理方式。其中异步请求封装了Dispatcher实现并发请求，默认每个webserver的并发最多为5，总并发最多为64。

![](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/98641-b1d546fc300315a7.png)

> 多路复用

先看一张图：

![](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/http2_multiplexing_real_thumb.png)

HTTP/1.*中，当在等待上一个请求响应的时候发送另外一个请求时，是需要等待上一个请求收到响应的，这就是堵塞的源头。

而在HTTP/2中，多个请求可以交错的发送消息给服务器，这里的每一个消息是一帧，就如上图每个圈圈是一帧，相同颜色代表一个请求，服务器收到帧消息后会根据帧内的标识符组装成一整块数据表示请求，这就是HTTP/2的多路复用。

在HTTP/2中每个请求代表一个流（stream）（不是socket输入输出流），流被拆分成帧传输，因此在HTTP/2中有两个非常重要的概念：帧（frame）和流（stream）。

多路复用是一种性能优化，关于web的性能优化还有以下说明：

如果你用 Chrome 的来分析 network 的话，你就会发现小文件如 JS/CSS 瓶颈其实在延时。假设你有个 JS 大小是 100KB，然后你在用 2Mbps 的 ADSL，带宽耗时是 400ms。在开始传输这 100KB 前，DNS 查询要 1 个 RTT（往返时间，即 ping 时间），建立 TCP 连接要 1 个 RTT，再建立 SSL 要 3 个 RTT，之后 HTTP 发请求又 1 个 RTT。假设你的 ping 是 25ms，6 个 RTT 就是 150ms。总和 550ms，延时占总和的 27%。这绝对不是个小数字，能优化掉必须优化掉。

DNS 缓存、持久连接（keep alive）、SPDY server push/hint 分别用于解决上述几个 RTT。此外上面说的还只是 ADSL，如果你用 4G RTT 会上到 100ms，3G 能超过 200ms。实际网站也很少给你单个 100KB 的 JS，往往是分开多个更小的 JS。这时候优化延时就显得很重要。
（http://www.zhihu.com/question/26515427/answer/33303927）

> OkHttp多路复用实现

OkHttp在实现多路复用的同时要考虑应用层的同步返回response的需求，封装了FramedConnection、FramedStream、FrameWriter。详细解释说明见https://github.com/square/okhttp/wiki/Concurrency

> Connections （URL、Address、Routes）

应用层构造的request使用URL，OkHttp的连接则有三种形式：URL、Address、Route。

URL包含以下可能需要处理的：

1. 区分schema，http和https，https需要定义好证书认证（HostnameVerifier、SSLSocketFactory）。
2. webServer可能有多个IP地址，或需要authenticate。

Address定义了一个webserver及相关的配置:端口、HTTPS设置、协议（HTTP/2或SPDY）

URLs共享同一个Address同时可能共享同一个TCP socket连接。OkHttp使用ConnectionPool复用HTTP/1.x连接和HTTP/2及SPDY的多路复用。

Routes提供一个webserver的动态信息：如代理、多IP地址

创建request时，OkHttp根据URL按照如下步骤创建Connections：

1. It uses the URL and configured OkHttpClient to create an address. This address specifies how we'll connect to the webserver.
2. It attempts to retrieve a connection with that address from the connection pool.
3. If it doesn't find a connection in the pool, it selects a route to attempt. This usually means making a DNS request to get the server's IP addresses. It then selects a TLS version and proxy server if necessary.
4. If it's a new route, it connects by building either a direct socket connection, a TLS tunnel (for HTTPS over an HTTP proxy), or a direct TLS connection. It does TLS handshakes as necessary.
5. It sends the HTTP request and reads the response

> HTTPS

OkHttp通过ConnectionSpec定义Https相关配置，如：

    ConnectionSpec spec = new ConnectionSpec.Builder(ConnectionSpec.MODERN_TLS)  
        .tlsVersions(TlsVersion.TLS_1_2)
        .cipherSuites(
              CipherSuite.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
              CipherSuite.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
              CipherSuite.TLS_DHE_RSA_WITH_AES_128_GCM_SHA256)
        .build();
    
    OkHttpClient client = new OkHttpClient.Builder() 
        .connectionSpecs(Collections.singletonList(spec))
        .build();

> Interceptors

OkHttp使用拦截器模式实现AOP编程：

    class LoggingInterceptor implements Interceptor {
      @Override public Response intercept(Interceptor.Chain chain) throws IOException {
        Request request = chain.request();
    
        long t1 = System.nanoTime();
        logger.info(String.format("Sending request %s on %s%n%s",
            request.url(), chain.connection(), request.headers()));
    
        Response response = chain.proceed(request);
    
        long t2 = System.nanoTime();
        logger.info(String.format("Received response for %s in %.1fms%n%s",
            response.request().url(), (t2 - t1) / 1e6d, response.headers()));
    
        return response;
      }
    }
    
Interceptors有Application interceptors和Network Interceptors，如图：
    
![](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/interceptors@2x.png)

上述LoggingInterceptor使用可如下：

    OkHttpClient client = new OkHttpClient.Builder()
        .addInterceptor(new LoggingInterceptor())
        .build();
    
    Request request = new Request.Builder()
        .url("http://www.publicobject.com/helloworld.txt")
        .header("User-Agent", "OkHttp Example")
        .build();
    
    Response response = client.newCall(request).execute();
    response.body().close();

比如经典的Gzip实现：

    /** This interceptor compresses the HTTP request body. Many webservers can't handle this! */
    final class GzipRequestInterceptor implements Interceptor {
      @Override public Response intercept(Interceptor.Chain chain) throws IOException {
        Request originalRequest = chain.request();
        if (originalRequest.body() == null || originalRequest.header("Content-Encoding") != null) {
          return chain.proceed(originalRequest);
        }
    
        Request compressedRequest = originalRequest.newBuilder()
            .header("Content-Encoding", "gzip")
            .method(originalRequest.method(), gzip(originalRequest.body()))
            .build();
        return chain.proceed(compressedRequest);
      }
    
      private RequestBody gzip(final RequestBody body) {
        return new RequestBody() {
          @Override public MediaType contentType() {
            return body.contentType();
          }
    
          @Override public long contentLength() {
            return -1; // We don't know the compressed length in advance!
          }
    
          @Override public void writeTo(BufferedSink sink) throws IOException {
            BufferedSink gzipSink = Okio.buffer(new GzipSink(sink));
            body.writeTo(gzipSink);
            gzipSink.close();
          }
        };
      }
    }

## 源码架构

![](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/okhttp_okhttpclient_class.png)

![](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/okhttp_instructure.png)

## 参考

1. https://packetzoom.com/blog/which-android-http-library-to-use.html
2. https://segmentfault.com/a/1190000003965158
3. http://square.github.io/okhttp/
4. https://github.com/square/okhttp， https://github.com/square/okio
5. http://www.blogjava.net/yongboy/archive/2015/03/19/423611.aspx
6. http://www.kancloud.cn/digest/web-performance-http2/74825
7. https://github.com/square/okhttp/wiki/Recipes
8. http://frodoking.github.io/2015/03/12/android-okhttp/
9. http://www.jianshu.com/p/6637369d02e7
