---
title: ImageLoder源码阅读笔记
---

>开源框架ImageLoader的源码阅读笔记,重点集中在线程管理机制和网络请求和缓存管理。其它繁琐内容暂且不去管。基于最新的1.95版本。文中有错还请纠正。我也是个菜鸟。互相学习

首先通过源码来了解下 ImageLoader 中的displayImage方法的去了解ImageLoader的大概架构，因为这个方法是我们使用ImageLoader中调用最多的方法了。也是最重要的方法了。
```Java
public void displayImage(String uri, ImageAware imageAware, DisplayImageOptions options,  
        ImageSize targetSize, ImageLoadingListener listener, ImageLoadingProgressListener progressListener) {  
  
    //一些检查工作  
    checkConfiguration();  
    if (imageAware == null) {  
        throw new IllegalArgumentException(ERROR_WRONG_ARGUMENTS);  
    }  
    if (listener == null) {  
        listener = defaultListener;  
    }  
    if (options == null) {  
        options = configuration.defaultDisplayImageOptions;  
    }  
      
    if (TextUtils.isEmpty(uri)) {  
        //这里有个ImageLoaderEngine 对象engine，这是个很重要的类，下文详解  
        engine.cancelDisplayTaskFor(imageAware);  
        listener.onLoadingStarted(uri, imageAware.getWrappedView());  
        if (options.shouldShowImageForEmptyUri()) {  
            imageAware.setImageDrawable(options.getImageForEmptyUri(configuration.resources));  
        } else {  
            imageAware.setImageDrawable(null);  
        }  
        listener.onLoadingComplete(uri, imageAware.getWrappedView(), null);  
        return;  
    }  
  
    if (targetSize == null) {  
        targetSize = ImageSizeUtils.defineTargetSizeForView(imageAware, configuration.getMaxImageSize());  
    }  
    //生成存储键值  
    String memoryCacheKey = MemoryCacheUtils.generateKey(uri, targetSize);  
    engine.prepareDisplayTaskFor(imageAware, memoryCacheKey);  
  
    listener.onLoadingStarted(uri, imageAware.getWrappedView());  
    //从一级缓存中去取缓存的bitmap   
    Bitmap bmp = configuration.memoryCache.get(memoryCacheKey);  
    if (bmp != null && !bmp.isRecycled()) {  
        //存在即加载喽......  
        L.d(LOG_LOAD_IMAGE_FROM_MEMORY_CACHE, memoryCacheKey);  
        //如果需要在源bitmap上有一些额外操作  
        if (options.shouldPostProcess()) {  
            ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,  
                    options, listener, progressListener, engine.getLockForUri(uri));  
            //这个Task可以在显示前由 程序员自定义对原bitmap的操作，（通过实现 bitmapProcessor接口）！  
            ProcessAndDisplayImageTask displayTask = new ProcessAndDisplayImageTask(engine, bmp, imageLoadingInfo,  
                    defineHandler(options));  
            if (options.isSyncLoading()) {  
                //如果是同步显示，则run开始走  
                displayTask.run();  
            } else {  
                //若是异步，那就就提交到线程池中执行  
                engine.submit(displayTask);  
            }  
        } else {  
            //没有的话，就去显示bitmap  
            options.getDisplayer().display(bmp, imageAware, LoadedFrom.MEMORY_CACHE);  
            listener.onLoadingComplete(uri, imageAware.getWrappedView(), bmp);  
        }  
    } else {  
        //加载时显示的图片  
        if (options.shouldShowImageOnLoading()) {  
            imageAware.setImageDrawable(options.getImageOnLoading(configuration.resources));  
        } else if (options.isResetViewBeforeLoading()) {  
            imageAware.setImageDrawable(null);  
        }  
  
        ImageLoadingInfo imageLoadingInfo = new ImageLoadingInfo(uri, imageAware, targetSize, memoryCacheKey,  
                options, listener, progressListener, engine.getLockForUri(uri));  
        LoadAndDisplayImageTask displayTask = new LoadAndDisplayImageTask(engine, imageLoadingInfo,  
                defineHandler(options));  
        if (options.isSyncLoading()) {  
            displayTask.run();  
        } else {  
            engine.submit(displayTask);  
        }  
    }  
}  
```

**从上面的代码可以看到displayImage这个方法中比较核心的几个类依次是ImageLoaderConfigura**

```Java	
	void submit(final LoadAndDisplayImageTask task) {
		taskDistributor.execute(new Runnable() {
			@Override
			public void run() {
				//先从本地缓存中读取已有的缓存
				File image = configuration.diskCache.get(task.getLoadingUri());
				boolean isImageCachedOnDisk = image != null && image.exists();
				initExecutorsIfNeed();
				//若有的话就从另外的线程去读
				if (isImageCachedOnDisk) {
					taskExecutorForCachedImages.execute(task);
				} else {
					taskExecutor.execute(task);
				}
			}
		});
	}
```

从engine的submit方法中看出，最终的加载显示的过程还是在LoadAndDisplayImageTask这个类中（废话。。）。下面我们来整体的看看这个Task类。首先来看它的run方法
```Java

	@Override
	public void run() {
		/*
			最主要的就是通过engine类检查 
			是否有线程池是否处于pause状态，若处于，则wait/
			另外这个方法还检查了 当前持有的view对象是非被GC 
			是否被用于其他image等 若是的话侧return
		*/
		if (waitIfPaused()) return;
		//你在options 中设置的delay
		if (delayIfNeed()) return;

		ReentrantLock loadFromUriLock = imageLoadingInfo.loadFromUriLock;
		L.d(LOG_START_DISPLAY_IMAGE_TASK, memoryCacheKey);
		if (loadFromUriLock.isLocked()) {
			L.d(LOG_WAITING_FOR_IMAGE_LOADED, memoryCacheKey);
		}

		loadFromUriLock.lock();
		Bitmap bmp;
		try {
			checkTaskNotActual();
				
			bmp = configuration.memoryCache.get(memoryCacheKey);
			if (bmp == null || bmp.isRecycled()) {
				bmp = tryLoadBitmap();
				if (bmp == null) return; // listener callback already was fired

				checkTaskNotActual();
				checkTaskInterrupted();
				
				if (options.shouldPreProcess()) {
					L.d(LOG_PREPROCESS_IMAGE, memoryCacheKey);
					//处理bitmap
					bmp = options.getPreProcessor().process(bmp);
					if (bmp == null) {
						L.e(ERROR_PRE_PROCESSOR_NULL, memoryCacheKey);
					}
				}

				if (bmp != null && options.isCacheInMemory()) {
					L.d(LOG_CACHE_IMAGE_IN_MEMORY, memoryCacheKey);
					configuration.memoryCache.put(memoryCacheKey, bmp);
				}
			} else {
				loadedFrom = LoadedFrom.MEMORY_CACHE;
				L.d(LOG_GET_IMAGE_FROM_MEMORY_CACHE_AFTER_WAITING, memoryCacheKey);
			}

			if (bmp != null && options.shouldPostProcess()) {
				L.d(LOG_POSTPROCESS_IMAGE, memoryCacheKey);
				bmp = options.getPostProcessor().process(bmp);
				if (bmp == null) {
					L.e(ERROR_POST_PROCESSOR_NULL, memoryCacheKey);
				}
			}
			checkTaskNotActual();
			checkTaskInterrupted();
		} catch (TaskCancelledException e) {
			fireCancelEvent();
			return;
		} finally {
			loadFromUriLock.unlock();
		}
		//最后，downBitmap完成，就要去显示了，又是交到另一个bitmap中去了
		DisplayBitmapTask displayBitmapTask = new DisplayBitmapTask(bmp, imageLoadingInfo, engine, loadedFrom);
		runTask(displayBitmapTask, syncLoading, handler, engine);
	}
从上面的代码可以看到 run方法的逻辑还是比较清晰的，下面看一下 tryLoadBitmap方法
	private Bitmap tryLoadBitmap() throws TaskCancelledException {
		Bitmap bitmap = null;
		try {
			//又是检查缓存..
			File imageFile = configuration.diskCache.get(uri);
			if (imageFile != null && imageFile.exists() && imageFile.length() > 0) {
				L.d(LOG_LOAD_IMAGE_FROM_DISK_CACHE, memoryCacheKey);
				loadedFrom = LoadedFrom.DISC_CACHE;
				
				checkTaskNotActual();
				//包裹成 “file://”形式，吐槽：真心不明白为什么要多次一举，因为在读取文件的时候又把这个scheme给去掉了
				bitmap = decodeImage(Scheme.FILE.wrap(imageFile.getAbsolutePath()));
			}
			if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
				L.d(LOG_LOAD_IMAGE_FROM_NETWORK, memoryCacheKey);

				loadedFrom = LoadedFrom.NETWORK;
				
				String imageUriForDecoding = uri;
				/*
					这一块的逻辑就是 若果有配置了缓存的逻辑，那么就从网络上拿下Bitmap对象，然后保存到本地
					保存到本地后。ImageUri就是本地的Uri 即以 “file://”开头的Sceme.
				*/
				if (options.isCacheOnDisk() && tryCacheImageOnDisk()) {
					imageFile = configuration.diskCache.get(uri);
					if (imageFile != null) {
						imageUriForDecoding = Scheme.FILE.wrap(imageFile.getAbsolutePath());
					}
				}
				
				checkTaskNotActual();
				//很明显的,最终无论走的 从网络取bitmap还是本地，都是走的这个方法
				bitmap = decodeImage(imageUriForDecoding);

				if (bitmap == null || bitmap.getWidth() <= 0 || bitmap.getHeight() <= 0) {
					fireFailEvent(FailType.DECODING_ERROR, null);
				}
			}
		} catch (IllegalStateException e) {
			fireFailEvent(FailType.NETWORK_DENIED, null);
		} catch (TaskCancelledException e) {
			throw e;
		} catch (IOException e) {
			L.e(e);
			fireFailEvent(FailType.IO_ERROR, e);
		} catch (OutOfMemoryError e) {
			L.e(e);
			fireFailEvent(FailType.OUT_OF_MEMORY, e);
		} catch (Throwable e) {
			L.e(e);
			fireFailEvent(FailType.UNKNOWN, e);
		}
		return bitmap;
	}

可以看的出来，最后的 拿bitmap都是交给 的decoder.decode方法，这个decoder的类是怎样的哪？ 我们可以招到 ImageDecoder这个和其实现 BaseImageDecoder类，这个BaseImageDecoder的类，就是默认的decode实现。 看代码
	

	public Bitmap decode(ImageDecodingInfo decodingInfo) throws IOException {
		Bitmap decodedBitmap;
		ImageFileInfo imageInfo;
		/*
			拿到InputStream 流，有各种得到iS的方法。基本用法全部在BaseImageDownLoader这里面。
			当然你也可以通过实现ImageDownLoader来定义自己的方法
		*/
		InputStream imageStream = getImageStream(decodingInfo);
		if (imageStream == null) {
			L.e(ERROR_NO_IMAGE_STREAM, decodingInfo.getImageKey());
			return null;
		}
		try {
			//下面就是通过你自动定义的Options 来定义解析Bitmap的options。最后解析出来 就OK了
			imageInfo = defineImageSizeAndRotation(imageStream, decodingInfo);
			imageStream = resetStream(imageStream, decodingInfo);
			Options decodingOptions = prepareDecodingOptions(imageInfo.imageSize, decodingInfo);
			decodedBitmap = BitmapFactory.decodeStream(imageStream, null, decodingOptions);
		} finally {
			IoUtils.closeSilently(imageStream);
		}

		if (decodedBitmap == null) {
			L.e(ERROR_CANT_DECODE_IMAGE, decodingInfo.getImageKey());
		} else {
			decodedBitmap = considerExactScaleAndOrientatiton(decodedBitmap, decodingInfo, imageInfo.exif.rotation,
					imageInfo.exif.flipHorizontal);
		}
		return decodedBitmap;
	}

 看上面的getImageStream方法，我们知道，ImageLoader不仅可以加载网络图片。还可以加载本地图片，甚至视频文件的一帧作为等等。最终的就是通过这个getStream调用的ImageDownLoader实现的。其基本实现是BaseImageDownLoader，我们可以看一下其里面的几个方法
	
	
	public InputStream getStream(String imageUri, Object extra) throws IOException {
		switch (Scheme.ofUri(imageUri)) {
			case HTTP:
			case HTTPS:
				return getStreamFromNetwork(imageUri, extra);
			case FILE:
				return getStreamFromFile(imageUri, extra);
			case CONTENT:
				return getStreamFromContent(imageUri, extra);
			case ASSETS:
				return getStreamFromAssets(imageUri, extra);
			case DRAWABLE:
				return getStreamFromDrawable(imageUri, extra);
			case UNKNOWN:
			default:
				return getStreamFromOtherSource(imageUri, extra);
		}
	}

上面的代码，就是通过Sceme 头来判断我到底改怎么去获取图片。网络，本地。Content，Assert文件还是...... 下面就是拿 NetWork 和本地File作为的源码看一下，其他的请读者自己研究
从网络获取

	protected InputStream getStreamFromNetwork(String imageUri, Object extra) throws IOException {
		//创建 UrlConnection
		HttpURLConnection conn = createConnection(imageUri, extra);

		int redirectCount = 0;
		//如果遇到重定向，最大重定向数目为5，
		while (conn.getResponseCode() / 100 == 3 && redirectCount < MAX_REDIRECT_COUNT) {
			conn = createConnection(conn.getHeaderField("Location"), extra);
			redirectCount++;
		}
		
		InputStream imageStream;
		try {
			imageStream = conn.getInputStream();
		} catch (IOException e) {
			// Read all data to allow reuse connection (http://bit.ly/1ad35PY)
			IoUtils.readAndCloseStream(conn.getErrorStream());
			throw e;
		}
		// ResponseCode 不是200 ，肯定有错
		if (!shouldBeProcessed(conn)) {
			IoUtils.closeSilently(imageStream);
			throw new IOException("Image request failed with response code " + conn.getResponseCode());
		}
		//生成buffer，供BitmapFactory 使用
		return new ContentLengthInputStream(new BufferedInputStream(imageStream, BUFFER_SIZE), conn.getContentLength());
	}

从本地读的代码比较简单，这里就不加注释了

	protected InputStream getStreamFromFile(String imageUri, Object extra) throws IOException {
		String filePath = Scheme.FILE.crop(imageUri);
		if (isVideoFileUri(imageUri)) {
			return getVideoThumbnailStream(filePath);
		} else {
			BufferedInputStream imageStream = new BufferedInputStream(new FileInputStream(filePath), BUFFER_SIZE);
			return new ContentLengthInputStream(imageStream, (int) new File(filePath).length());
		}
	}
```

