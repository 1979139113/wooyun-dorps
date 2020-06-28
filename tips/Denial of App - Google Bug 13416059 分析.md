# Denial of App - Google Bug 13416059 分析

0x01 背景
-------

* * *

Soot作者Eric Bodden所在的实验室, Secure Software Engineering最近宣布他们将在SPSM'14上讲述名为Denial-of-App-Attack的Android系统漏洞，影响4.4.3之前的机型，并给出了poc和对应的google commit id.

这个在googlecode上对应的链接是[https://code.google.com/p/android/issues/detail?id=65790](https://code.google.com/p/android/issues/detail?id=65790)

POC：[https://github.com/secure-software-engineering/denial-of-app-attack](https://github.com/secure-software-engineering/denial-of-app-attack)

该问题可以导致攻击者可以指定应用使其无法安装在手机上，除非有root权限或者factory reset手机。可以被木马用来占位拒绝杀毒软件的安装，或者占位拒绝竞品安装。下面是根据commit diff和poc给出的漏洞具体分析。

0x02 问题现象：
----------

* * *

下载安装这个POC，可以看到其实就是指定一个packagename，例如com.taobao.taobao，然后生成了一个malformed的APK并执行安装，由于该APK的dex是非法的，安装的时候会报告INSTALL_FAILED_DEXOPT并安装失败。但如果随后安装真正的com.taobao.taobao时，即使指定了重新安装选项（pm install -r），却会报INSTALL_FAILED_UID_CHANGED，导致后续安装失败，而在被占位的手机上已安装应用中却找不到com.taobao.taobao，自然也无法清除掉占位的幽灵，造成真正的淘宝应用完全无法安装，推而广之可以用在360等杀毒软件上。

![POC](http://drops.javaweb.org/uploads/images/f1f6e2a12532421ceb796ecac60a3a0c4a927e9a.jpg)

![安装之后](http://drops.javaweb.org/uploads/images/5fe07651249dc2984793e51f63239c9322ddd536.jpg)

![正常应用无法安装](http://drops.javaweb.org/uploads/images/4ef3d93b35ab93b140549106e1754ef6dc7d01bf.jpg)

![install -r](http://drops.javaweb.org/uploads/images/56866b63885135cf8d2a3ffeb0c8d48a4fe94a8f.jpg)

0x03 问题本质：
----------

Google的diff对此问题的描述是：

> We'd otherwise leave the data dirs & native libraries lying around. This will leave the app permanently broken because the next install of the app will fail with INSTALL_FAILED_UID_CHANGED.
> 
> Also remove an unnecessary instance variable.
> 
> Cherry-pick from master Bug 13416059

通过观察可以发现，第一次安装（所谓“占位”）结束的时候，在/data/data/目录下已经有了com.taobao.taobao目录并分配了一个uid，例如u70(10070)，但第二次安装的时候，PackageManager却出现了UID_CHANGED的error，而没有复用u70，这是为什么？

![uid](http://drops.javaweb.org/uploads/images/d2a4b93711851208c2fefd6cd110cd00778093e4.jpg)

INSTALL_FAILED_DEXOPT和UID_CHANGED是在如下代码块中：

```
3622    private PackageParser.Package scanPackageLI(PackageParser.Package pkg,
3623            int parseFlags, int scanMode, long currentTime, UserHandle user) {
//....
4141        if ((scanMode&SCAN_NO_DEX) == 0) {
4142            if (performDexOptLI(pkg, forceDex, (scanMode&SCAN_DEFER_DEX) != 0)
4143                    == DEX_OPT_FAILED) {
4144                mLastScanError = PackageManager.INSTALL_FAILED_DEXOPT;
4145                return null;
4146            }
4147        }

```

scanPackageLI函数流程大概如下：

```
/**/
//检查是否系统应用
/**/
//检查Package是否重复，否则抛出PackageManager.INSTALL_FAILED_DUPLICATE_PACKAGE
  // Initialize package source and resource directories
3686        File destCodeFile = new File(pkg.applicationInfo.sourceDir);
3687        File destResourceFile = new File(pkg.applicationInfo.publicSourceDir);
//...
 // Just create the setting, don't add it yet. For already existing packages
3812            // the PkgSetting exists already and doesn't have to be created.
3813            pkgSetting = mSettings.getPackageLPw(pkg, origPackage, realName, suid, destCodeFile,
3814                    destResourceFile, pkg.applicationInfo.nativeLibraryDir,
3815                    pkg.applicationInfo.flags, user, false);
//在这之后uid已经被指定了
/**/
//检查签名
//检查Provider权限

//开始创建目录
   final long scanFileTime = scanFile.lastModified();
3926        final boolean forceDex = (scanMode&SCAN_FORCE_DEX) != 0;
3927        pkg.applicationInfo.processName = fixProcessName(
3928                pkg.applicationInfo.packageName,
3929                pkg.applicationInfo.processName,
3930                pkg.applicationInfo.uid);
3931
3932        File dataPath;
3933        if (mPlatformPackage == pkg) {
//omit
3937        } else {
3938            // This is a normal package, need to make its data directory.
3939            dataPath = getDataPathForPackage(pkg.packageName, 0);
3940
3941            boolean uidError = false;
3942
3943            if (dataPath.exists()) {
3944                int currentUid = 0;
3945                try {
3946                    StructStat stat = Libcore.os.stat(dataPath.getPath());
3947                    currentUid = stat.st_uid;
3948                } catch (ErrnoException e) {
3949                    Slog.e(TAG, "Couldn't stat path " + dataPath.getPath(), e);
3950                }
3951
3952                // If we have mismatched owners for the data path, we have a problem.
3953                if (currentUid != pkg.applicationInfo.uid) {
3954                    boolean recovered = false;
3955                    if (currentUid == 0) {
3956                     //omit...
3969                    }
3970                    if (!recovered && ((parseFlags&PackageParser.PARSE_IS_SYSTEM) != 0
3971                            || (scanMode&SCAN_BOOTING) != 0)) {
3972                        // If this is a system app, we can at least delete its
3973                        // current data so the application will still work.
3974                        //omit...
4001                    } else if (!recovered) {
4002                        // If we allow this install to proceed, we will be broken.
4003                        // Abort, abort!
4004                        mLastScanError = PackageManager.INSTALL_FAILED_UID_CHANGED;
4005                        return null;
4006                    }
                 } else {//目录不存在，新建立
4029                if (DEBUG_PACKAGE_SCANNING) {
4030                    if ((parseFlags & PackageParser.PARSE_CHATTY) != 0)
4031                        Log.v(TAG, "Want this data dir: " + dataPath);
4032                }
4033                //invoke installer to do the actual installation
4034                int ret = createDataDirsLI(pkgName, pkg.applicationInfo.uid);//建立目录
4035                if (ret < 0) {
4036                    // Error from installer
4037                    mLastScanError = PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
4038                    return null;
4039                }
4040
4041                if (dataPath.exists()) {
4042                    pkg.applicationInfo.dataDir = dataPath.getPath();
4043                } else {
4044                    Slog.w(TAG, "Unable to create data directory: " + dataPath);
4045                    pkg.applicationInfo.dataDir = null;
4046                }
4047            }
//omit...
//拷贝nativeLibrary
//omit...
//进行DexOpt
4141        if ((scanMode&SCAN_NO_DEX) == 0) {
4142            if (performDexOptLI(pkg, forceDex, (scanMode&SCAN_DEFER_DEX) != 0)
4143                    == DEX_OPT_FAILED) {
4144                mLastScanError = PackageManager.INSTALL_FAILED_DEXOPT;
4145                return null;
4146            }
4147        }

```

那么漏洞的原理就很清楚了，第一次占位安装时，故意让PMS在数据目录已分配uid并写入了/data/data/下之后走到dexopt时使其报错，导致安装异常终止，此时已放置的数据目录却没有被清除掉。第二次安装的时候package被分配了新的的uid，但此时已有同名却不同uid的数据目录存在，导致uid_changed错误，安装失败。

为什么第二次安装的时候就会被分配不同的uid?关键在于 mSettings.getPackageLPw,辗转ref到/frameworks/base/services/java/com/android/server/pm/Settings.java

```
private PackageSetting getPackageLPw(String name, PackageSetting origPackage,
359            String realName, SharedUserSetting sharedUser, File codePath, File resourcePath,
360            String nativeLibraryPathString, int vc, int pkgFlags,
361            UserHandle installUser, boolean add, boolean allowInstall) {
//omit...
    } else {
423                p = new PackageSetting(name, realName, codePath, resourcePath,
424                        nativeLibraryPathString, vc, pkgFlags);
425                p.setTimeStamp(codePath.lastModified());
426                p.sharedUser = sharedUser;
427                // If this is not a system app, it starts out stopped.
428                if ((pkgFlags&ApplicationInfo.FLAG_SYSTEM) == 0) {
429                    if (DEBUG_STOPPED) {
430                        RuntimeException e = new RuntimeException("here");
431                        e.fillInStackTrace();
432                        Slog.i(PackageManagerService.TAG, "Stopping package " + name, e);
433                    }
434                    List<UserInfo> users = getAllUsers();
435                    if (users != null && allowInstall) {
436                        for (UserInfo user : users) {
437                            // By default we consider this app to be installed
438                            // for the user if no user has been specified (which
439                            // means to leave it at its original value, and the
440                            // original default value is true), or we are being
441                            // asked to install for all users, or this is the
442                            // user we are installing for.
443                            final boolean installed = installUser == null
444                                    || installUser.getIdentifier() == UserHandle.USER_ALL
445                                    || installUser.getIdentifier() == user.id;
446                            p.setUserState(user.id, COMPONENT_ENABLED_STATE_DEFAULT,
447                                    installed,
448                                    true, // stopped,
449                                    true, // notLaunched
450                                    null, null);
451                            writePackageRestrictionsLPr(user.id);
452                        }
453                    }
454                }
455                if (sharedUser != null) {
456                    p.appId = sharedUser.userId;
457                } else {
458                    // Clone the setting here for disabled system packages
459                    PackageSetting dis = mDisabledSysPackages.get(name);
460                    if (dis != null) {
//omit..
484                    } else {
485                        // Assign new user id
486                        p.appId = newUserIdLPw(p);//关键点
487                    }
488                }

```

继续查看newUserIdLPw

```
private int newUserIdLPw(Object obj) {
2360        // Let's be stupidly inefficient for now...
2361        final int N = mUserIds.size();
2362        for (int i = 0; i < N; i++) {
2363            if (mUserIds.get(i) == null) {//检查空位
2364                mUserIds.set(i, obj);
2365                return Process.FIRST_APPLICATION_UID + i;
2366            }
2367        }
2368
2369        // None left?
2370        if (N > (Process.LAST_APPLICATION_UID-Process.FIRST_APPLICATION_UID)) {
2371            return -1;
2372        }
2373
2374        mUserIds.add(obj);
2375        return Process.FIRST_APPLICATION_UID + N;
2376    }

```

mUserIds是一个PackageSettings的数组状结构，维护了当前的userid，并在安装时遍历进行分配。在第一次恶意的占位安装中， mUserIds这个array状结构已经被添加了一个PackageSettings进去，形成类似于

```
[PackageSetting{(10001, bla)}, ..., PackageSetting{(10070, com.taobao.taobao)}]

```

的结构，但在dexopt failed的时候最末尾一项没有被移除。随后再安装时，newUserIdLPw会遍历mUserIds，发现没有空位，就会在末尾重新添加一个，就会形成

```
[PackageSetting{(10001, bla)},...,PackageSetting{(10070, com.taobao.taobao)},PackageSetting{(10071, com.taobao.taobao)}]

```

的结构，导致两次安装分配的UID不同，触发INSTALL_FAILED_UID_CHANGED。

但值得注意的是，这时候mUserIds并没有被固化在packages.xml和packages.list中。

0x04 进一步思考
----------

* * *

那么这样肯定会想到，如果杀掉system_server（软重启），让其重新扫描并建立mUserIds数组不就能修复这个问题了？

理论上来说，如果在重启前没有安装过其他应用的话，那么这还真是可行的。因为重启后重新建立的uid数组是[(10001, bla),...,()10069, haha)]，那么重新安装的com.taobao.taobao刚好能占到10070的位置，皆大欢喜。

但如果在重启后又安装了其他应用，那么其就会占掉10070的位置，导致taobao再安装的时候以10071及之后的uid就拿不回原来应该属于它的/data/data/com.taoba.taobao了... what a pity.

以上在stock rom(Genymotion, SDK)和小米2、Nexus等上验证通过。

所以现在看来，原作者说只有root或者reset才能清除这个问题的说法似乎不准确，至少从给出的poc和google的diff来看实验结果某些情况下重启就能fix。具体还有什么细节就只能等待SPSM的paper了。总体来说，这是一个比较好玩的trick类漏洞，而且从issuelink来看，应该还有一些其他类型的同样效果的漏洞存在。

0x05 修复
-------

* * *

Google对此的修复：

Google的diff主要是添加了SCAN_DELETE_DATA_ON_FAILURES的flag，在设置了该flag的时候安装失败时会删除遗留掉的文件。

```
@@ -4644,6 +4643,10 @@
         if ((scanMode&SCAN_NO_DEX) == 0) {
             if (performDexOptLI(pkg, forceDex, (scanMode&SCAN_DEFER_DEX) != 0, false)
                     == DEX_OPT_FAILED) {
+                if ((scanMode & SCAN_DELETE_DATA_ON_FAILURES) != 0) {
+                    removeDataDirsLI(pkg.packageName);
+                }
+
                 mLastScanError = PackageManager.INSTALL_FAILED_DEXOPT;
                 return null;
             }
@@ -4721,6 +4724,10 @@
                     PackageParser.Package clientPkg = clientLibPkgs.get(i);
                     if (performDexOptLI(clientPkg, forceDex, (scanMode&SCAN_DEFER_DEX) != 0, false)
                             == DEX_OPT_FAILED) {
+                        if ((scanMode & SCAN_DELETE_DATA_ON_FAILURES) != 0) {
+                            removeDataDirsLI(pkg.packageName);
+                        }
+
                         mLastScanError = PackageManager.INSTALL_FAILED_DEXOPT;
                         return null;
                     }

```

如何fix某个占位攻击：

root下删除该数据目录即可，非root。。。那只能reset了。