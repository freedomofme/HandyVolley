
## HandyVolley文档

本文主要介绍了一个为[仿网易新闻APP](https://github.com/freedomofme/Netease)的而定制的Volley框架。

这也是为什么要去修改一个2013年发布的老框架的原因，主要是为了兼容以前的代码。（如果是新上线没有历史负担的项目，网络库(retrofit等)和图片库(glide)等库的性能要更优秀。）
<!--more-->


### 一、介绍

本项目的基于2016年12月29号的原版Volley库开发，也是截止目前的最新版本。从提交日志来看项目代码基于稳定，不会存在重大缺陷。

目前项目最新版本为1.0.3版本，后期可以会针对做一些微小的优化调整。
本项目兼容原版Volley库的API。

已经把项目托管于Jcenter仓库，Android项目的话可以用以下语句在Gradle 中引用该项目库。

	compile 'site.okhttp.codeyel:HandyVolley:1.0.3'

### 第二类：更新细节


##### 1.0.0
Fork基于2016年12月29号的原版Volley库
##### 1.0.1
更新compileSdkVersion为23
##### 1.0.2
1. 增加初始化时的磁盘缓存大小设置
2. 增加磁盘缓存控制策略：自定义网络响应的Ttl和softTtl。
3. 增加自定义的缓存设置相对于服务器的缓存策略的优先级控制
4. 对用户设置不缓存的请求，跳过响应解析操作，微小提升性能。

##### 1.0.3（Stable）	
1. 将1.0.2的缓存策略应用于对ImageLoader类
2. 修复缓存控制Bug

### 使用样例
经介绍区别于原本Volley的使用方法。

1. 增加初始化时的磁盘缓存大小设置

- 修改方式:Volley.java文件中增加newRequestQueue方法的重载方法
				
		public static RequestQueue newRequestQueue(Context context, HttpStack stack, int maxCacheSizeInBytes) 
		
- 使用方法（其中第三个参数表示瓷片缓存大小为30MB）
		mRequestQueue = Volley.newRequestQueue(mCtx.getApplicationContext(), null, 30 * 1024 * 1024);
		
		

2.增加磁盘缓存控制策略：自定义网络响应的Ttl和softTtl


- Ttl和softTtl说明
	Ttl和softTtl用来用户自定义缓存时间，通常softTtl <= Ttl。
	
	- 当一个请求的过期时间 > Ttl, 则重新请求服务器。
	
	- 当一个请求的过期时间 > softTtl && < Ttl, 则先使用缓存数据做出响应，并同时将该请求发送服务器。（也就是说，响应回调函数会触发两次）
	
	- 当一个请求的过期时间 > softTtl，则直接使用本地缓存。
	
	根据以上特性，通过设置恰当的Ttl和softTtl，APP可以实现数据的及时展现和刷新。

- 修改细节
	1. 在Request.java 增加以下三个属性值和相应的getter：
			
			/** this request's default Soft time limit(unit m		illisecond) if no cache control set by web severs*/
    		private int mDefaultSoftTtl = 0;

    		/** this request's default time limit(unit millisecond) if no cache control set by web severs*/
    		private int mDefaultTtl = 0;

    		/**
     		* Returns true means use the default TTL and soft TTL regardless of the server's cache control.
     		* Returns false means server's cache control has higher priority.
     		*/
    		private boolean localCacheControl = false;
    	
    	---
    	 	/**
     		* Returns true if responses to this request should be cached.
     		*/
    		public final boolean shouldCache() {
	        return mShouldCache;
   		 	}

    		/**
     		* Return this request's soft time limit (unit seconds).
     		*/
	    	public int getDefaultSoftTtl() {
   		   	  return mDefaultSoftTtl;
    		}

    		/**
     		* Returns this request's the time limit (unit seconds).
    		 */
    		public int getDefaultTtl() {
        	return mDefaultTtl;
		    }
		    
    	
    其中mDefaultSoftTtl表示默认的软缓存时间(单位毫秒)，mDefaultTtl表示默认的缓存时间(单位毫秒)，localCacheControl表示当mDefaultSoftTtl、mDefaultTtl的值和服务器返回的缓存策略冲突时，应该采用哪个数值。（怕大家看不懂我写的蹩脚英文，翻译一下）
    
    2.同理，在ImageLoader中也增加了以上三个属性和相应的getter。
    
	方法原型如下：	
			
			/**
     		* Constructs a new ImageLoader.
	     	* @param queue The RequestQueue to use for making image requests.
	        * @param imageCache The cache to use as an L1 cache.
   	  		* @param  defaultSoftTtl this request's time limit (unit seconds).
   	 		* @param  defaultTtl this request's soft time limit (unit seconds).
   	  		* @param useLocalCacheControl Returns this request's cache control priority.
   	  		*/
    		public ImageLoader(RequestQueue queue, ImageCache imageCache, int defaultSoftTtl, int defaultTtl, boolean useLocalCacheControl) {
        		mRequestQueue = queue;
	        	mCache = imageCache;
   	   		  	mDefaultSoftTtl = defaultSoftTtl;
   		     	mDefaultTtl = defaultTtl;
	      		localCacheControl = useLocalCacheControl;
   			}
    
	
- 使用方法
	- 对于不经过ImageLoader的Request，如StringRequest，直接覆写Request<T>属性的getter方法。

			new StringRequest(Request.Method.GET, url, responseListener, new DefaultErrorListener(context)) {
          		@Override
	          	public int getDefaultTtl() {
   	          		return 15 * 24 * 3600 * 1000;
   		       	}
       		   	@Override
	          	public int getDefaultSoftTtl() {
		       		return 1 * 60 * 1000;
          		}
          		@Override
          		public boolean shouldLocalCacheControl() {
            	    return true;
          		}
      		  };
    		}
    		
    	以上StringRequest表示默认缓存时间15天，默认软缓存时间1分钟，当与服务器的cache-control冲突时采用此数据为准。
	
	
	- 对于经过ImageLoader的Request，直接调用ImageLoader的另一个构造方法。

		mImageLoader = new ImageLoader(mRequestQueue, new MyLrnCache(mCtx), 1 * 3600 * 1000, 15 * 24 * 3600 * 1000, true);
		
		以上ImageLoader表示默认缓存时间15天，默认软缓存时间1小时，当与服务器的cache-control冲突时采用此数据为准。
		
- 实际案例

	[Netease--仿网易新闻Android端APP](https://github.com/freedomofme/Netease)

	
		
		
		
		



	



 