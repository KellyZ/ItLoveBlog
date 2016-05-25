## 介绍

官方文档：http://developer.android.com/training/volley/index.html

1. 设计目标就是非常适合去进行数据量不大，但通信频繁的网络操作，而对于大数据量的网络操作，比如说下载文件等，Volley的表现就会非常糟糕
2. SDK 小于9则使用Apache的http，否则使用 HttpUrlConnectin （SDK小于9的HttpUrlConnectin有bug）；
3. 对外提供RequestQueue、Request组件的概念，内部封装HttpStack、NetworkDispatcher组件；

## 原理解析

![](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/volley.png)

> RequestQueue

见：./src/main/java/com/android/volley/toolbox/Volley.java

    public static RequestQueue newRequestQueue(Context context, HttpStack stack) {
        File cacheDir = new File(context.getCacheDir(), "volley");
        String userAgent = "volley/0";

        try {
            String network = context.getPackageName();
            PackageInfo queue = context.getPackageManager().getPackageInfo(network, 0);
            userAgent = network + "/" + queue.versionCode;
        } catch (NameNotFoundException var6) {
            ;
        }

        if(stack == null) {
            if(VERSION.SDK_INT >= 9) {
                stack = new HurlStack();
            } else {
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }

        BasicNetwork network1 = new BasicNetwork((HttpStack)stack);
        RequestQueue queue1 = new RequestQueue(new DiskBasedCache(cacheDir), network1);
        queue1.start();
        return queue1;
    }

    public static RequestQueue newRequestQueue(Context context) {
        return newRequestQueue(context, (HttpStack)null);
    }

在Volley.newRequestQueue初始化了RequestQueue，并调用了start初始化了一个CacheDispatcher线程，四个NetworkDispatcher线程：
   
    public void start() {
        this.stop();
        this.mCacheDispatcher = new CacheDispatcher(this.mCacheQueue, this.mNetworkQueue, this.mCache, this.mDelivery);
        this.mCacheDispatcher.start();

        for(int i = 0; i < this.mDispatchers.length; ++i) {
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(this.mNetworkQueue, this.mNetwork, this.mCache, this.mDelivery);
            this.mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }

    }

> HttpStack

   API>9,在HttpURLConnection基础封装了HurlStack，否则用封装了HttpClient的HttpClientStack（见上面）；

   最后暴露给NetworkDispatcher的是Network（BasicNetwork实现）

   执行request请求分别在HttpStack.performRequest中：
   
   如**HurlStack.performRequest**:
   
        public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders) throws IOException, AuthFailureError {
            String url = request.getUrl();
            HashMap map = new HashMap();
            map.putAll(request.getHeaders());
            map.putAll(additionalHeaders);
            if(this.mUrlRewriter != null) {
                String parsedUrl = this.mUrlRewriter.rewriteUrl(url);
                if(parsedUrl == null) {
                    throw new IOException("URL blocked by rewriter: " + url);
                }
    
                url = parsedUrl;
            }
    
            URL parsedUrl1 = new URL(url);
            HttpURLConnection connection = this.openConnection(parsedUrl1, request);
            Iterator responseCode = map.keySet().iterator();
    
            while(responseCode.hasNext()) {
                String protocolVersion = (String)responseCode.next();
                connection.addRequestProperty(protocolVersion, (String)map.get(protocolVersion));
            }
    
            setConnectionParametersForRequest(connection, request);
            ProtocolVersion protocolVersion1 = new ProtocolVersion("HTTP", 1, 1);
            int responseCode1 = connection.getResponseCode();
            if(responseCode1 == -1) {
                throw new IOException("Could not retrieve response code from HttpUrlConnection.");
            } else {
                BasicStatusLine responseStatus = new BasicStatusLine(protocolVersion1, connection.getResponseCode(), connection.getResponseMessage());
                BasicHttpResponse response = new BasicHttpResponse(responseStatus);
                response.setEntity(entityFromConnection(connection));
                Iterator var12 = connection.getHeaderFields().entrySet().iterator();
    
                while(var12.hasNext()) {
                    Entry header = (Entry)var12.next();
                    if(header.getKey() != null) {
                        BasicHeader h = new BasicHeader((String)header.getKey(), (String)((List)header.getValue()).get(0));
                        response.addHeader(h);
                    }
                }
    
                return response;
            }
        }

   如**HttpClientStack.performRequest：
   
        public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders) throws IOException, AuthFailureError {
            HttpUriRequest httpRequest = createHttpRequest(request, additionalHeaders);
            addHeaders(httpRequest, additionalHeaders);
            addHeaders(httpRequest, request.getHeaders());
            this.onPrepareRequest(httpRequest);
            HttpParams httpParams = httpRequest.getParams();
            int timeoutMs = request.getTimeoutMs();
            HttpConnectionParams.setConnectionTimeout(httpParams, 5000);
            HttpConnectionParams.setSoTimeout(httpParams, timeoutMs);
            return this.mClient.execute(httpRequest);
        }

> CacheDispatcher

    public class CacheDispatcher extends Thread {
        private static final boolean DEBUG;
        private final BlockingQueue<Request<?>> mCacheQueue;
        private final BlockingQueue<Request<?>> mNetworkQueue;
        private final Cache mCache;
        ......
        
        @Override
        public void run() {
            if (DEBUG) VolleyLog.v("start new dispatcher");
            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    
            // Make a blocking call to initialize the cache.
            mCache.initialize();
    
            while (true) {
                try {
                    // Get a request from the cache triage queue, blocking until
                    // at least one is available.
                    final Request<?> request = mCacheQueue.take();
                    request.addMarker("cache-queue-take");
    
                    // If the request has been canceled, don't bother dispatching it.
                    if (request.isCanceled()) {
                        request.finish("cache-discard-canceled");
                        continue;
                    }
    
                    // Attempt to retrieve this item from cache.
                    Cache.Entry entry = mCache.get(request.getCacheKey());
                    if (entry == null) {
                        request.addMarker("cache-miss");
                        // Cache miss; send off to the network dispatcher.
                        mNetworkQueue.put(request);
                        continue;
                    }
    
                    // If it is completely expired, just send it to the network.
                    if (entry.isExpired()) {
                        request.addMarker("cache-hit-expired");
                        request.setCacheEntry(entry);
                        mNetworkQueue.put(request);
                        continue;
                    }
                    // We have a cache hit; parse its data for delivery back to the request.
                    request.addMarker("cache-hit");
                    Response<?> response = request.parseNetworkResponse(
                            new NetworkResponse(entry.data, entry.responseHeaders));
                    request.addMarker("cache-hit-parsed");
    
                    if (!entry.refreshNeeded()) {
                        // Completely unexpired cache hit. Just deliver the response.
                        mDelivery.postResponse(request, response);
                    } else {
                        // Soft-expired cache hit. We can deliver the cached response,
                        // but we need to also send the request to the network for
                        // refreshing.
                        request.addMarker("cache-hit-refresh-needed");
                        request.setCacheEntry(entry);
    
                        // Mark the response as intermediate.
                        response.intermediate = true;
    
                        // Post the intermediate response back to the user and have
                        // the delivery then forward the request along to the network.
                        mDelivery.postResponse(request, response, new Runnable() {
                            @Override
                            public void run() {
                                try {
                                    mNetworkQueue.put(request);
                                } catch (InterruptedException e) {
                                    // Not much we can do about this.
                                }
                            }
                        });
                    }
    
                } catch (InterruptedException e) {
                    // We may have been interrupted because it was time to quit.
                    if (mQuit) {
                        return;
                    }
                    continue;
                }
            }
        }

> NetworkDispatcher

    public class NetworkDispatcher extends Thread {
        private final BlockingQueue<Request<?>> mQueue;
        private final Network mNetwork;
        private final Cache mCache;
        ......
        
        @Override
        public void run() {
            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            while (true) {
                long startTimeMs = SystemClock.elapsedRealtime();
                Request<?> request;
                try {
                    // Take a request from the queue.
                    request = mQueue.take();
                } catch (InterruptedException e) {
                    // We may have been interrupted because it was time to quit.
                    if (mQuit) {
                        return;
                    }
                    continue;
                }
    
                try {
                    request.addMarker("network-queue-take");
    
                    // If the request was cancelled already, do not perform the
                    // network request.
                    if (request.isCanceled()) {
                        request.finish("network-discard-cancelled");
                        continue;
                    }
    
                    addTrafficStatsTag(request);
    
                    // Perform the network request.
                    NetworkResponse networkResponse = mNetwork.performRequest(request);
                    request.addMarker("network-http-complete");
    
                    // If the server returned 304 AND we delivered a response already,
                    // we're done -- don't deliver a second identical response.
                    if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                        request.finish("not-modified");
                        continue;
                    }
    
                    // Parse the response here on the worker thread.
                    Response<?> response = request.parseNetworkResponse(networkResponse);
                    request.addMarker("network-parse-complete");
                    
                    // Write to cache if applicable.
                    // TODO: Only update cache metadata instead of entire record for 304s.
                    if (request.shouldCache() && response.cacheEntry != null) {
                        mCache.put(request.getCacheKey(), response.cacheEntry);
                        request.addMarker("network-cache-written");
                    }
    
                    // Post the response back.
                    request.markDelivered();
                    mDelivery.postResponse(request, response);
                } catch (VolleyError volleyError) {
                    volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                    parseAndDeliverNetworkError(request, volleyError);
                } catch (Exception e) {
                    VolleyLog.e(e, "Unhandled exception %s", e.toString());
                    VolleyError volleyError = new VolleyError(e);
                    volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                    mDelivery.postError(request, volleyError);
                }
            }
        }
    ......   
     

> Request

RequestQueue初始化了线程池及HttpStack（可以称为Http处理器），引起使用者只需构造好Request及Response对并加入Request队列即可。

Request对外提供的接口有：

    public abstract class Request<T> implements Comparable<Request<T>> {
        private static final String DEFAULT_PARAMS_ENCODING = "UTF-8";
        ...
        private static final long SLOW_REQUEST_THRESHOLD_MS = 3000L;
        ...
        public Request(int method, String url, ErrorListener listener) {
            ...
        } 
        public Request<?> setRetryPolicy(RetryPolicy retryPolicy) {
            this.mRetryPolicy = retryPolicy;
            return this;
        }
        ...
        public Map<String, String> getHeaders() throws AuthFailureError {
            return Collections.emptyMap();
        }
        public String getBodyContentType() {
            return "application/x-www-form-urlencoded; charset=" + this.getParamsEncoding();
        }
        public byte[] getBody() throws AuthFailureError {
            Map params = this.getParams();
            return params != null && params.size() > 0?this.encodeParameters(params, this.getParamsEncoding()):null;
        }
    
        private byte[] encodeParameters(Map<String, String> params, String paramsEncoding) {
            StringBuilder encodedParams = new StringBuilder();
    
            try {
                Iterator var5 = params.entrySet().iterator();
    
                while(var5.hasNext()) {
                    java.util.Map.Entry uee = (java.util.Map.Entry)var5.next();
                    encodedParams.append(URLEncoder.encode((String)uee.getKey(), paramsEncoding));
                    encodedParams.append('=');
                    encodedParams.append(URLEncoder.encode((String)uee.getValue(), paramsEncoding));
                    encodedParams.append('&');
                }
    
                return encodedParams.toString().getBytes(paramsEncoding);
            } catch (UnsupportedEncodingException var6) {
                throw new RuntimeException("Encoding not supported: " + paramsEncoding, var6);
            }
        }
        
        public final int getTimeoutMs() {
            return this.mRetryPolicy.getCurrentTimeout();
        }
        
        protected abstract Response<T> parseNetworkResponse(NetworkResponse var1);
        protected abstract void deliverResponse(T var1);
        
        
1. 默认的DefaultRetryPolicy是重试1次，间隔2500ms，backoff因子1.0F（backoff算法可以在网上了解）；
2. 可覆写getBodyContentType()；
3. 可覆写getBody()；
4. 可覆写getHeaders()实现自定义header;
4. 超时是通过mRetryPolicy获取的，默认是2500ms，注：HurlStack连接超时和读取数据超时都是该接口，但HttpClientStack的连接超时是硬编码5000ms；
5. Request实现类需要实现parseNetworkResponse，deliverResponse；

如StringRequest的实现：

    public class StringRequest extends Request<String> {
        private final Listener<String> mListener;
    
        public StringRequest(int method, String url, Listener<String> listener, ErrorListener errorListener) {
            super(method, url, errorListener);
            this.mListener = listener;
        }
    
        public StringRequest(String url, Listener<String> listener, ErrorListener errorListener) {
            this(0, url, listener, errorListener);
        }
    
        protected void deliverResponse(String response) {
            this.mListener.onResponse(response);
        }
    
        protected Response<String> parseNetworkResponse(NetworkResponse response) {
            String parsed;
            try {
                parsed = new String(response.data, HttpHeaderParser.parseCharset(response.headers));
            } catch (UnsupportedEncodingException var4) {
                parsed = new String(response.data);
            }
    
            return Response.success(parsed, HttpHeaderParser.parseCacheHeaders(response));
        }
    }
    
如JsonRequest的实现：

    public abstract class JsonRequest<T> extends Request<T> {
        /** Default charset for JSON request. */
        protected static final String PROTOCOL_CHARSET = "utf-8";
    
        /** Content type for request. */
        private static final String PROTOCOL_CONTENT_TYPE =
            String.format("application/json; charset=%s", PROTOCOL_CHARSET);
    
        private final Listener<T> mListener;
        private final String mRequestBody;
               
        public JsonRequest(int method, String url, String requestBody, Listener<T> listener,
                ErrorListener errorListener) {
            super(method, url, errorListener);
            mListener = listener;
            mRequestBody = requestBody;
        }
    
        @Override
        protected void deliverResponse(T response) {
            mListener.onResponse(response);
        }
    
        @Override
        abstract protected Response<T> parseNetworkResponse(NetworkResponse response);
        
        @Override
        public String getBodyContentType() {
            return PROTOCOL_CONTENT_TYPE;
        }
    
        @Override
        public byte[] getBody() {
            try {
                return mRequestBody == null ? null : mRequestBody.getBytes(PROTOCOL_CHARSET);
            } catch (UnsupportedEncodingException uee) {
                VolleyLog.wtf("Unsupported Encoding while trying to get the bytes of %s using %s",
                        mRequestBody, PROTOCOL_CHARSET);
                return null;
            }
        }
        
> RequestFuture

Request默认是加入NetworkDispatcher线程中执行的，如果需要同步则需要通过RequestFuture，使用方法如下：
    
    RequestFuture<JSONObject> future = RequestFuture.newFuture();
    StringRequest myReq = new StringRequest(Request.Method.POST, url, future, future);
    requestQueue.add(myReq);

    try {
        Response response = future.get();
    } catch (InterruptedException e) {
        // handle the error
    } catch (ExecutionException e) {
        // handle the error
    }

原理是通过wait和notify实现的：

    @Override
    public T get(long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException {
        return doGet(TimeUnit.MILLISECONDS.convert(timeout, unit));
    }

    private synchronized T doGet(Long timeoutMs)
            throws InterruptedException, ExecutionException, TimeoutException {
        if (mException != null) {
            throw new ExecutionException(mException);
        }

        if (mResultReceived) {
            return mResult;
        }

        if (timeoutMs == null) {
            wait(0);
        } else if (timeoutMs > 0) {
            wait(timeoutMs);
        }

        if (mException != null) {
            throw new ExecutionException(mException);
        }

        if (!mResultReceived) {
            throw new TimeoutException();
        }

        return mResult;
    }
    
    public synchronized void onResponse(T response) {
        this.mResultReceived = true;
        this.mResult = response;
        this.notifyAll();
    }

    public synchronized void onErrorResponse(VolleyError error) {
        this.mException = error;
        this.notifyAll();
    }
    
> ImageLoader&NetworkImageView

* 缓存需要自己实现

        public class LruBitmapCache extends LruCache<String, Bitmap> implements
            ImageCache {
            public static int getDefaultLruCacheSize() {
                final int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);
                final int cacheSize = maxMemory / 8;
         
                return cacheSize;
            }
         
            public LruBitmapCache() {
                this(getDefaultLruCacheSize());
            }
         
            public LruBitmapCache(int sizeInKiloBytes) {
                super(sizeInKiloBytes);
            }
         
            @Override
            protected int sizeOf(String key, Bitmap value) {
                return value.getRowBytes() * value.getHeight() / 1024;
            }
         
            @Override
            public Bitmap getBitmap(String url) {
                return get(url);
            }
         
            @Override
            public void putBitmap(String url, Bitmap bitmap) {
                put(url, bitmap);
            }
        }
    
* 初始化ImageLoader：

        public RequestQueue getRequestQueue() {
            if (mRequestQueue == null) {
                mRequestQueue = Volley.newRequestQueue(getApplicationContext());
            }
     
            return mRequestQueue;
        }
     
        public ImageLoader getImageLoader() {
            getRequestQueue();
            if (mImageLoader == null) {
                mImageLoader = new ImageLoader(this.mRequestQueue,
                        new LruBitmapCache());
            }
            return this.mImageLoader;
        }

* 使用ImageLoader：


        ImageLoader imageLoader = getImageLoader();
     
        // If you are using normal ImageView
        imageLoader.get(Const.URL_IMAGE, new ImageListener() {
         
            @Override
            public void onErrorResponse(VolleyError error) {
                Log.e(TAG, "Image Load Error: " + error.getMessage());
            }
         
            @Override
            public void onResponse(ImageContainer response, boolean arg1) {
                if (response.getBitmap() != null) {
                    // load image into imageview
                    imageView.setImageBitmap(response.getBitmap());
                }
            }
        });


## 参考

1. http://www.androidhive.info/2014/05/android-working-with-volley-library-1/