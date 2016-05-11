afinal:

https://github.com/yangfuhai/afinal

## 模块

1. FinalDB模块：android中的orm框架，一行代码就可以进行增删改查。支持一对多，多对一等查询。
2. FinalActivity模块：android中的ioc框架，完全注解方式就可以进行UI绑定和事件绑定。无需findViewById和setClickListener等
3. FinalHttp模块：通过httpclient进行封装http数据请求，支持ajax方式加载
4. FinalBitmap模块：通过FinalBitmap，imageview加载bitmap的时候无需考虑bitmap加载过程中出现的oom和android容器快速滑动时候出现的图片错位等现象。FinalBitmap可以配置线程加载线程数量，缓存大小，缓存路径，加载显示动画等。FinalBitmap的内存管理使用lru算法，没有使用弱引用（android2.3以后google已经不建议使用弱引用，android2.3后强行回收软引用和弱引用，详情查看android官方文档），更好的管理bitmap内存。FinalBitmap可以自定义下载器，用来扩展其他协议显示网络图片，比如ftp等。同时可以自定义bitmap显示器，在imageview显示图片的时候播放动画等（默认是渐变动画显示）


## FinalHttp源码分析

1. http请求常用配置：超时、最大连接数、TcpNoDelay、SocketBuffer大小、Scheme、请求头等，见FinalHttp构造函数：

        public FinalHttp() {
            BasicHttpParams httpParams = new BasicHttpParams();
    
            ConnManagerParams.setTimeout(httpParams, socketTimeout);
            ConnManagerParams.setMaxConnectionsPerRoute(httpParams, new ConnPerRouteBean(maxConnections));
            ConnManagerParams.setMaxTotalConnections(httpParams, 10);
    
            HttpConnectionParams.setSoTimeout(httpParams, socketTimeout);
            HttpConnectionParams.setConnectionTimeout(httpParams, socketTimeout);
            HttpConnectionParams.setTcpNoDelay(httpParams, true);
            HttpConnectionParams.setSocketBufferSize(httpParams, DEFAULT_SOCKET_BUFFER_SIZE);
    
            HttpProtocolParams.setVersion(httpParams, HttpVersion.HTTP_1_1);
    
            SchemeRegistry schemeRegistry = new SchemeRegistry();
            schemeRegistry.register(new Scheme("http", PlainSocketFactory.getSocketFactory(), 80));
            schemeRegistry.register(new Scheme("https", SSLSocketFactory.getSocketFactory(), 443));
            ThreadSafeClientConnManager cm = new ThreadSafeClientConnManager(httpParams, schemeRegistry);
    
            httpContext = new SyncBasicHttpContext(new BasicHttpContext());
            httpClient = new DefaultHttpClient(cm, httpParams);
            httpClient.addRequestInterceptor(new HttpRequestInterceptor() {
                public void process(HttpRequest request, HttpContext context) {
                    if (!request.containsHeader(HEADER_ACCEPT_ENCODING)) {
                        request.addHeader(HEADER_ACCEPT_ENCODING, ENCODING_GZIP);
                    }
                    for (String header : clientHeaderMap.keySet()) {
                        request.addHeader(header, clientHeaderMap.get(header));
                    }
                }
            });
    
            httpClient.addResponseInterceptor(new HttpResponseInterceptor() {
                public void process(HttpResponse response, HttpContext context) {
                    final HttpEntity entity = response.getEntity();
                    if (entity == null) {
                        return;
                    }
                    final Header encoding = entity.getContentEncoding();
                    if (encoding != null) {
                        for (HeaderElement element : encoding.getElements()) {
                            if (element.getName().equalsIgnoreCase(ENCODING_GZIP)) {
                                response.setEntity(new InflatingEntity(response.getEntity()));
                                break;
                            }
                        }
                    }
                }
            });
    
            httpClient.setHttpRequestRetryHandler(new RetryHandler(maxRetries));
    
            clientHeaderMap = new HashMap<String, String>();
            
        }

## Download

1. 初始化线程池（静态）

        private static int httpThreadCount = 3;//http线程池数量
        private static final ThreadFactory  sThreadFactory = new ThreadFactory() {
            private final AtomicInteger mCount = new AtomicInteger(1);
            public Thread newThread(Runnable r) {
            	Thread tread = new Thread(r, "FinalHttp #" + mCount.getAndIncrement());
            	tread.setPriority(Thread.NORM_PRIORITY - 1);
                return tread;
            }
        };
    
    private static final Executor executor =Executors.newFixedThreadPool(httpThreadCount, sThreadFactory);

2. 使用HttpHandler异步下载

        public HttpHandler<File> download( String url,AjaxParams params, String target,boolean isResume, AjaxCallBack<File> callback) {
        	final HttpGet get =  new HttpGet(getUrlWithQueryString(url, params));
        	HttpHandler<File> handler = new HttpHandler<File>(httpClient, httpContext, callback,charset);
        	handler.executeOnExecutor(executor,get,target,isResume);
        	return handler;
        }

3. HttpHandler继承AsyncTask，调用的是AsyncTask的executeOnExecutor，最终执行的doInBackground：

        @Override
    	protected Object doInBackground(Object... params) {
    		if(params!=null && params.length == 3){
    			targetUrl = String.valueOf(params[1]);
    			isResume = (Boolean) params[2];
    		}
    		try {
    			publishProgress(UPDATE_START); // 开始
    			makeRequestWithRetries((HttpUriRequest)params[0]);
    			
    		} catch (IOException e) {
    			publishProgress(UPDATE_FAILURE,e,0,e.getMessage()); // 结束
    		}
    
    		return null;
    	}
    	
4. Http断点下载是通过RANGE头参数进行分段下载的：request.setHeader("RANGE", "bytes="+fileLen+"-"); 由服务器来确定一段的大小

        private void makeRequestWithRetries(HttpUriRequest request) throws IOException {
    		if(isResume && targetUrl!= null){
    			File downloadFile = new File(targetUrl);
    			long fileLen = 0;
    			if(downloadFile.isFile() && downloadFile.exists()){
    				fileLen = downloadFile.length();
    			}
    			if(fileLen > 0)
    				request.setHeader("RANGE", "bytes="+fileLen+"-");
    		}
    		
    		boolean retry = true;
    		IOException cause = null;
    		HttpRequestRetryHandler retryHandler = client.getHttpRequestRetryHandler();
    		while (retry) {
    			try {
    				if (!isCancelled()) {
    					HttpResponse response = client.execute(request, context);
    					if (!isCancelled()) {
    						handleResponse(response);
    					} 
    				}
    				return;
    			} catch (UnknownHostException e) {
    				publishProgress(UPDATE_FAILURE,e,0,"unknownHostException：can't resolve host");
    				return;
    			} catch (IOException e) {
    				cause = e;
    				retry = retryHandler.retryRequest(cause, ++executionCount,context);
    			} catch (NullPointerException e) {
    				// HttpClient 4.0.x 之前的一个bug
    				// http://code.google.com/p/android/issues/detail?id=5255
    				cause = new IOException("NPE in HttpClient" + e.getMessage());
    				retry = retryHandler.retryRequest(cause, ++executionCount,context);
    			}catch (Exception e) {
    				cause = new IOException("Exception" + e.getMessage());
    				retry = retryHandler.retryRequest(cause, ++executionCount,context);
    			}
    		}
    		if(cause!=null)
    			throw cause;
    		else
    			throw new IOException("未知网络错误");
    	}
    	
5. 下载成功后会调用AsyncTask的publishPress，其实就是回调onPregressUpdate

        private void handleResponse(HttpResponse response) {
        	StatusLine status = response.getStatusLine();
        	if (status.getStatusCode() >= 300) {
        		String errorMsg = "response status error code:"+status.getStatusCode();
        		if(status.getStatusCode() == 416 && isResume){
        			errorMsg += " \n maybe you have download complete.";
        		}
        		publishProgress(UPDATE_FAILURE,new HttpResponseException(status.getStatusCode(), status.getReasonPhrase()),status.getStatusCode() ,errorMsg);
        	} else {
        		try {
        			HttpEntity entity = response.getEntity();
        			Object responseBody = null;
        			if (entity != null) {
        				time = SystemClock.uptimeMillis();
        				if(targetUrl!=null){
        					responseBody = mFileEntityHandler.handleEntity(entity,this,targetUrl,isResume);
        				}
        				else{
        					responseBody = mStrEntityHandler.handleEntity(entity,this,charset);
        				}
        					
        			}
        			publishProgress(UPDATE_SUCCESS,responseBody);
        			
        		} catch (IOException e) {
        			publishProgress(UPDATE_FAILURE,e,0,e.getMessage());
        		}
        		
        	}
        }
        
6. 下载成功的时候是通过mFileEntityHandler.handleEntity(entity,this,targetUrl,isResume)保存到文件里：

        public Object handleEntity(HttpEntity entity, EntityCallBack callback,String target,boolean isResume) throws IOException {
    		if (TextUtils.isEmpty(target) || target.trim().length() == 0)
    			return null;
    
    		File targetFile = new File(target);
    
    		if (!targetFile.exists()) {
    			targetFile.createNewFile();
    		}
    
    		if(mStop){
    			return targetFile;
    		}
    			
    		
    		long current = 0;
    		FileOutputStream os = null;
    		if(isResume){
    			current = targetFile.length();
    			os = new FileOutputStream(target, true);
    		}else{
    			os = new FileOutputStream(target);
    		}
    		
    		if(mStop){
    			return targetFile;
    		}
    			
    		InputStream input = entity.getContent();
    		long count = entity.getContentLength() + current;
    		
    		if(current >= count || mStop){
    			return targetFile;
    		}
    		
    		int readLen = 0;
    		byte[] buffer = new byte[1024];
    		while (!mStop && !(current >= count) && ((readLen = input.read(buffer,0,1024)) > 0) ) {//未全部读取
    			os.write(buffer, 0, readLen);
    			current += readLen;
    			callback.callBack(count, current,false);
    		}
    		callback.callBack(count, current,true);
    		
    		if(mStop && current < count){ //用户主动停止
    			throw new IOException("user stop download thread");
    		}
    		
    		return targetFile;
    	}
    	
7. 每读取保存1024个字节就调用callback.callBack(count, current,false);即HttpHandler的callback实现，用于断点下载量和速度的显示：

        private long time;
    	@Override
    	public void callBack(long count, long current,boolean mustNoticeUI) {
    		if(callback!=null && callback.isProgress()){
    			if(mustNoticeUI){
    				publishProgress(UPDATE_LOADING,count,current);
    			}else{
    				long thisTime = SystemClock.uptimeMillis();
    				if(thisTime - time >= callback.getRate()){ //速度是每秒更新一次
    					time = thisTime ;
    					publishProgress(UPDATE_LOADING,count,current);
    				}
    			}
    		}
    	}
    	
总结：

1. 异步借助于AsyncTask中线程池；
2. 断点使用RANGE头信息，读取已下载文件的大小；
3. 每读取1024并距离上次更新进度1秒钟后即更新进度；

## Bitmap缓存

** 内存缓存 SoftMemoryCacheImpl、BaseMemoryCacheImpl**

1. 配置内存缓存大小：
2. 软引用方式：`private final HashMap<String, SoftReference<Bitmap>> mMemoryCache;`
3. 常规Lru算法：`private final LruMemoryCache<String, Bitmap> mMemoryCache;`其实就是`private final LinkedHashMap<K, V> map;` (有序链接表)

** 磁盘缓存 DiskCache ** 

1. 配置磁盘缓存路径及大小
2. 创建 ".idx"和".0"和".1"文件，".0"和".1"是数据缓存，他们只有一个“激活”状态的，当缓存数据达到用户配置量的时候会清空一个数据，然后切换另一个

        // The index file format: (all numbers are stored in little-endian)
        // [0]  Magic number: 0xB3273030
        // [4]  MaxEntries: Max number of hash entries per region.
        // [8]  MaxBytes: Max number of data bytes per region (including header).
        // [12] ActiveRegion: The active growing region: 0 or 1.
        // [16] ActiveEntries: The number of hash entries used in the active region.
        // [20] ActiveBytes: The number of data bytes used in the active region.
        // [24] Version number.
        // [28] Checksum of [0..28).
        // [32] Hash entries for region 0. The size is X = (12 * MaxEntries bytes).
        // [32 + X] Hash entries for region 1. The size is also X.
        //
        // Each hash entry is 12 bytes: 8 bytes key and 4 bytes offset into the data
        // file. The offset is 0 when the slot is free. Note that 0 is a valid value
        // for key. The keys are used directly as index into a hash table, so they
        // should be suitably distributed.
        //
        // Each data file stores data for one region. The data file is concatenated
        // blobs followed by the magic number 0xBD248510.
        //
        // The blob format:
        // [0]  Key of this blob
        // [8]  Checksum of this blob
        // [12] Offset of this blob
        // [16] Length of this blob (not including header)
        // [20] Blob


** 显示图片 **

    private void doDisplay(View imageView, String uri, BitmapDisplayConfig displayConfig) {
		if(!mInit ){
			init();
		}
		
		if (TextUtils.isEmpty(uri) || imageView == null) {
			return;
		}
		
		if(displayConfig == null)
			displayConfig = mConfig.defaultDisplayConfig;
	
		Bitmap bitmap = null;
	
	    // 首先从内存中取
		if (mImageCache != null) {
			bitmap = mImageCache.getBitmapFromMemoryCache(uri);
		}
	
		if (bitmap != null) {
			if(imageView instanceof ImageView){
				((ImageView)imageView).setImageBitmap(bitmap);
			}else{
				imageView.setBackgroundDrawable(new BitmapDrawable(bitmap));
			}
			
			
		}else if (checkImageTask(uri, imageView)) {
			final BitmapLoadAndDisplayTask task = new BitmapLoadAndDisplayTask(imageView, displayConfig );
			//设置默认图片
			final AsyncDrawable asyncDrawable = new AsyncDrawable(mContext.getResources(), displayConfig.getLoadingBitmap(), task);
	       
			if(imageView instanceof ImageView){
				((ImageView)imageView).setImageDrawable(asyncDrawable);
			}else{
				imageView.setBackgroundDrawable(asyncDrawable);
			}
	        
	        task.executeOnExecutor(bitmapLoadAndDisplayExecutor, uri);
	    }
	}
	
	
## Http Multipart支持