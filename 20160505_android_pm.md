## 说明

Android系统在启动的过程中，会启动一个应用程序管理服务PackageManagerService，这个服务负责扫描系统中特定的目录，找到里面的应用程序文件，即以Apk为后缀的文件，然后对这些文件进解析，得到应用程序的相关信息，完成应用程序的安装过程

应用程序管理服务PackageManagerService安装应用程序的过程，其实就是解析析应用程序配置文件AndroidManifest.xml的过程，并从里面得到得到应用程序的相关信息，例如得到应用程序的组件Activity、Service、Broadcast Receiver和Content Provider等信息，有了这些信息后，通过ActivityManagerService这个服务，我们就可以在系统中正常地使用这些应用程序了。

应用程序管理服务PackageManagerService是系统启动的时候由SystemServer组件启动的，启后它就会执行应用程序安装的过程，因此，本文将从SystemServer启动PackageManagerService服务的过程开始分析系统中的应用程序安装的过程。

## PackageManagerService启动

SystemServer组件是由Zygote进程负责启动的，启动的时候就会调用它的main函数：

services/core/java/com/android/server/pm/PackageManagerService.java

main()

   --> init1(args): 初始化SurfaceFlinger、SensorService、AudioFlinger、MediaPlayerService、CameraService和AudioPolicyService这几个服务
   
   --> init2() : 创建了一个ServerThread线程
   
   这个函数创建了一个ServerThread线程，通过PackageManagerService.main初始化PackageManagerService，还启动了其它很多的服务，如ActivityManagerService。
   
    class PackageManagerService extends IPackageManager.Stub {
    	......
    
    	public static final IPackageManager main(Context context, boolean factoryTest) {
    		PackageManagerService m = new PackageManagerService(context, factoryTest);
    		ServiceManager.addService("package", m);
    		return m;
    	}
    
    	......
    }
    
   PackageManagerService服务被添加到ServiceManager中去，ServiceManager是Android系统Binder进程间通信机制的守护进程，负责管理系统中的Binder对象。
   
   
## PackageManagerService安装APK

在创建这个PackageManagerService服务实例时，会在PackageManagerService类的构造函数中开始执行安装应用程序的过程

    class PackageManagerService extends IPackageManager.Stub {
    	......
    
    	public PackageManagerService(Context context, boolean factoryTest) {
    		......
    
    		synchronized (mInstallLock) {
    			synchronized (mPackages) {
    				......
    
    				File dataDir = Environment.getDataDirectory();
    				mAppDataDir = new File(dataDir, "data");
    				mSecureAppDataDir = new File(dataDir, "secure/data");
    				mDrmAppPrivateInstallDir = new File(dataDir, "app-private");
    
    				......
    
    				mFrameworkDir = new File(Environment.getRootDirectory(), "framework");
    				mDalvikCacheDir = new File(dataDir, "dalvik-cache");
    
    				......
    
    				// Find base frameworks (resource packages without code).
    				mFrameworkInstallObserver = new AppDirObserver(
    				mFrameworkDir.getPath(), OBSERVER_EVENTS, true);
    				mFrameworkInstallObserver.startWatching();
    				scanDirLI(mFrameworkDir, PackageParser.PARSE_IS_SYSTEM
    					| PackageParser.PARSE_IS_SYSTEM_DIR,
    					scanMode | SCAN_NO_DEX, 0);
    
    				// Collect all system packages.
    				mSystemAppDir = new File(Environment.getRootDirectory(), "app");
    				mSystemInstallObserver = new AppDirObserver(
    					mSystemAppDir.getPath(), OBSERVER_EVENTS, true);
    				mSystemInstallObserver.startWatching();
    				scanDirLI(mSystemAppDir, PackageParser.PARSE_IS_SYSTEM
    					| PackageParser.PARSE_IS_SYSTEM_DIR, scanMode, 0);
    
    				// Collect all vendor packages.
    				mVendorAppDir = new File("/vendor/app");
    				mVendorInstallObserver = new AppDirObserver(
    					mVendorAppDir.getPath(), OBSERVER_EVENTS, true);
    				mVendorInstallObserver.startWatching();
    				scanDirLI(mVendorAppDir, PackageParser.PARSE_IS_SYSTEM
    					| PackageParser.PARSE_IS_SYSTEM_DIR, scanMode, 0);
    
    
    				mAppInstallObserver = new AppDirObserver(
    					mAppInstallDir.getPath(), OBSERVER_EVENTS, false);
    				mAppInstallObserver.startWatching();
    				scanDirLI(mAppInstallDir, 0, scanMode, 0);
    
    				mDrmAppInstallObserver = new AppDirObserver(
    					mDrmAppPrivateInstallDir.getPath(), OBSERVER_EVENTS, false);
    				mDrmAppInstallObserver.startWatching();
    				scanDirLI(mDrmAppPrivateInstallDir, PackageParser.PARSE_FORWARD_LOCK,
    					scanMode, 0);
    
    				......
    			}
    		}
    	}
    
    	......
    }
    
这里会调用scanDirLI函数来扫描移动设备上的下面这五个目录中的Apk文件：

    /system/framework

    /system/app

    /vendor/app

    /data/app

    /data/app-private
    
## scanDirLI

    class PackageManagerService extends IPackageManager.Stub {
    	......
    
    	private void scanDirLI(File dir, int flags, int scanMode, long currentTime) {
    		String[] files = dir.list();
    		......
    
    		int i;
    		for (i=0; i<files.length; i++) {
    			File file = new File(dir, files[i]);
    			if (!isPackageFilename(files[i])) {
    				// Ignore entries which are not apk's
    				continue;
    			}
    			PackageParser.Package pkg = scanPackageLI(file,
    				flags|PackageParser.PARSE_MUST_BE_APK, scanMode, currentTime);
    			// Don't mess around with apps in system partition.
    			if (pkg == null && (flags & PackageParser.PARSE_IS_SYSTEM) == 0 &&
    				mLastScanError == PackageManager.INSTALL_FAILED_INVALID_APK) {
    					// Delete the apk
    					Slog.w(TAG, "Cleaning up failed install of " + file);
    					file.delete();
    			}
    		}
    	}
    
    
    	......
    }
    
对于目录中的每一个文件，如果是以后Apk作为后缀名，那么就调用scanPackageLI函数来对它进行解析和安装。

## scanPackageLI

    class PackageManagerService extends IPackageManager.Stub {
    	......
    
    	private PackageParser.Package scanPackageLI(File scanFile,
    			int parseFlags, int scanMode, long currentTime) {
    		......
    
    		String scanPath = scanFile.getPath();
    		parseFlags |= mDefParseFlags;
    		PackageParser pp = new PackageParser(scanPath);
    		
    		......
    
    		final PackageParser.Package pkg = pp.parsePackage(scanFile,
    			scanPath, mMetrics, parseFlags);
    
    		......
    
    		return scanPackageLI(pkg, parseFlags, scanMode | SCAN_UPDATE_SIGNATURE, currentTime);
    	}
    
    	......
    }
    
这个函数首先会为这个Apk文件创建一个PackageParser实例，接着调用这个实例的parsePackage函数来对这个Apk文件进行解析。这个函数最后还会调用另外一个版本的scanPackageLI函数把来解析后得到的应用程序的package、provider、service、receiver和activity等信息保存在PackageManagerService中

## PackageParser.parsePackage

    public class PackageParser {
    	......
    
    	public Package parsePackage(File sourceFile, String destCodePath,
    			DisplayMetrics metrics, int flags) {
    		......
    
    		mArchiveSourcePath = sourceFile.getPath();
    
    		......
    
    		XmlResourceParser parser = null;
    		AssetManager assmgr = null;
    		boolean assetError = true;
    		try {
    			assmgr = new AssetManager();
    			int cookie = assmgr.addAssetPath(mArchiveSourcePath);
    			if(cookie != 0) {
    				parser = assmgr.openXmlResourceParser(cookie, "AndroidManifest.xml");
    				assetError = false;
    			} else {
    				......
    			}
    		} catch (Exception e) {
    			......
    		}
    
    		......
    
    		String[] errorText = new String[1];
    		Package pkg = null;
    		Exception errorException = null;
    		try {
    			// XXXX todo: need to figure out correct configuration.
    			Resources res = new Resources(assmgr, metrics, null);
    			pkg = parsePackage(res, parser, flags, errorText);
    		} catch (Exception e) {
    			......
    		}
    
    		......
    
    		parser.close();
    		assmgr.close();
    
    		// Set code and resource paths
    		pkg.mPath = destCodePath;
    		pkg.mScanPath = mArchiveSourcePath;
    		//pkg.applicationInfo.sourceDir = destCodePath;
    		//pkg.applicationInfo.publicSourceDir = destRes;
    		pkg.mSignatures = null;
    
    		return pkg;
    	}
    
    	......
    }
    
 每一个Apk文件都是一个归档文件，它里面包含了Android应用程序的配置文件AndroidManifest.xml，这里主要就是要对这个配置文件就行解析了，从Apk归档文件中得到这个配置文件后，就调用另一外版本的parsePackage函数对AndroidManifest.xml文件中的各个标签进行解析。
 
 