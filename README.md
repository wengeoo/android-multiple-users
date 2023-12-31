# android-multiple-users
Android的这个“用户”并不是等同于linux下的用户概念。Android是基于Linux的OS，Linux下有一套自己的账户管理体系，而Android对此有一些封装和改动。同时，Android也引进了自己的多用户功能。

 Android与Linux的用户概念异同


先以第三方应用为例做说明:


一个应用被安装后,系统给分配唯一的"Application ID", 简称 =="AppId"== . 同时系统中会有多个用户(User),每个用户也有一个唯一的ID值,称为 =="UserId"== 。



Android这里的"UserId"跟Linux的UserId完全不是同一个东西。UserId *10000 + appId 才等于Linux下的UserId, 即进程所属用户的概念, 在Android我们通常记做 =="uid"==，以下以微信为例作为说明：



// 查看微信进程信息
$ adb shell ps | grep tencent
USER      PID   PPID  VSIZE   RSS    WCHAN                PC  NAME
u0_a110   26250 416   2144140 203204 SyS_epoll_ 00e7b40428 S com.tencent.mm
u0_a110   26338 416   1794628 123868 SyS_epoll_ 00e7b40428 S com.tencent.mm:push


可以看到微信创建了2个进程，其第一列USER字段均为u0_a110. ==这个u0_a110就是uid== .


这个字段这样拆解成int值:



以"_"为界线, 前一部分是UserId, 后一部分是ApplicationId. 转为int值即为:
u0_a110 == 0 * 10000 + (10110) == 10110 == uid;
         (u0)*(十万)   a110

1> u0即表示userId = 0;
2> a110中的"a"永远翻译为10000(一万)
2> "userId * 100000 + appId = uid"是代码中写死的规则, 全系统通用.


==uid就对应Linux系统里"进程所属用户的概念".==
在Android系统里, 我们可以很容易发现:



同个应用创建的多个进程, 进程uid相同.

不同应用创建的进程, 其uid一定不同.(除非设置了shareUid)



从这里我们看到Android跟Linux处理"用户"这一概念的不同了：



Android基于Linux，Android进程的底层实际上就是对应的Linux进程；

在Linux系统中的"用户"概念，在android这里叫做"uid"；

而android的"用户"概念是一个更高层次封装, 可以说是跟linux毫无关系的一个概念。



Android的用户概念(UserId)



从Android L（5.0）开始引入多用户API。先不论权限问题，我们可以创建多个用户。Android为什么引入这样的多用户概念? 有至少以下2个意图:



实现"访客模式"这样的通常PC端支持的多用户模式。其特点设备可以支持多个人在不同时段登陆, 独享设备。比如上班时登陆A用户,回家给小孩玩,登陆B用户. A/B用户数据完全独立, 互不影响。



单纯为了"应用多开"，原生的计划基于多用户做"工作模式"(Android For work, 简"AFW")。所谓工作模式就是可在手机上打开一个应用的多个实例,同时运行与同一个桌面下,而又数据相互独立：一个用于工作, 一个用于生活. 简单的说,基本就是我们要的"应用多开"。


以上2点的区别是, 前一种多用户是必须有一个明显的用户切换过程,一个时段内只有一个用户下的进程可以在前台显示和交互；后一种是在主用户的基础上创建一个依附于主用户的"影子用户"(Managed Profile)。



前一种运行情况可以用我们flyme-设置-访客模式 来体验，进入访客模式就是进入B用户。



后一种影子用户与主用户运行在同一个桌面下 效果如下图:

图片


图1:桌面上有2个qq, 一个是主用户的, 一个是"影子用户的"



图片

图2:多任务里, 显示2个qq, 一个是主用户下的qq进程组, 一个是影子用户下的qq进程组。



所谓 =="影子用户"== 是我个人的翻译. 谷歌官方把这种用户一直称为 =="Managed Profile"== , 字面翻译不能表达其真实含义, 所以我将其意译为"影子用户"。



使用Android原生多用户 -- "普通用户"



可以使用adb命令来模拟:



// 第一步,创新新用户:
$ adb shell pm craete-user 'test-user'

// 第二步,得道新用户的userId:
$ adb shell dumpsys user 
  ...
  UserInfo{10:test-user:0} serialNo=1001
  ...
// 得道UserId = 10

// 第三步, 启动新用户:
$ adb shell am start-user 10

// 第四步, 将新用户切到前台来:
$ adb shell am switch-user 10

// 第五步, 校验切换用户成功:
$ adb shell dumpsys activity a | grep 'Hist '
      * Hist #0: ActivityRecord{8636857 u10 com.meizu.flyme.launcher/.Launcher t1100001}

// 看到桌面"com.meizu.flyme.launcher/.Launcher"运行在"u10"即运行在userId=10的用户下, 说明新用户正处与前台.

// ps: 如切回主用户不赘述, 可自行查询 "adb shell pm / am"命令.


使用Android原生多用户 -- "影子用户"



以微信为例, 我们测试创建新用户,并尝试双开微信。



第一步,创建一个android:sharedUserId="android.uid.system"的应用, 调用如下代码,创建ManagedProfileUser(影子用户), 并启动该用户。（ps:创建"影子用户必须使用代码, 没有对应的adb命令）虽然创建影子用户跟普通用户走一样的代码, 只是flags的不同.但是adb命令中无法指定这个flash,故而创建影子用户需使用代码调用API创建；



 int FLAG_MANAGED_PROFILE = 0x00000020; // 创建影子用户必要的flag
    private static final String FLYME_PARALLEL_SPACE_USER_NAME = "FlymeParallelSpace";// 指定影子用户UserName
    private UserManager mUserManager;
    private Object mFlymeParallelSpaceUserInfo = null; // multi-open UserInfo

    static int getUserIdFromUserInfo(Object userInfo) {
        int userId = -1;
        try {
            Field field_id = userInfo.getClass().getDeclaredField("id");
            field_id.setAccessible(true);
            userId = (Integer)field_id.get(userInfo);
        } catch (Exception e) {
            Log.d(TAG, "getUserIdFromUserInfo() E:" + e.toString(), e);
        }
        return userId;
    }

    public void openFlymeParallelSpace() {
        mUserManager = (UserManager) mContext.getSystemService(Context.USER_SERVICE);

        // step 1 : call UMS.createProfileForUser() to create Managed Profile User
        UserHandle userHandle = android.os.Process.myUserHandle();
        int getIdentifier = ReflectCache.on(userHandle, "getIdentifier").invoke();
        mFlymeParallelSpaceUserInfo = ReflectCache.on(mUserManager,
                "createProfileForUser",
                String.class, int.class, int.class)
                .invoke(FLYME_PARALLEL_SPACE_USER_NAME,
                        FLAG_MANAGED_PROFILE /*| FLAG_DISABLED*/,
                        getIdentifier);


        // step 2 call AMS.startUserInBackground() to start the new user.
        int userId = getUserIdFromUserInfo(mFlymeParallelSpaceUserInfo);
        Object iActivityManager = ReflectCache.on("android.app.ActivityManagerNative",
                    "getDefault").invoke();
            boolean isOk = ReflectCache.on(iActivityManager, "startUserInBackground",
                    int.class).invoke(userId);
            Log.d(TAG, "startUserInBackground() userId = " + userId + " | isOk = " + isOk);
    }
 

第二步,安装应用到影子用户, 如果事先已安装了微信,则可是使用如下命令将微信额外安装到影子用户:



// 首先要获取影子用户的userID:
$ adb shell dumpsys user
  ...
  UserInfo{10:FlymeParallelSpace:30} serialNo=10
  ...

// UserInfo{10:FlymeParallelSpace:30}表示:
//  userId=10, 
//  userName="FlymeParallelSpace", 
//  flags=0x30
于是我们得到影子用户的UserId


// 如微信已经安装了, 使用adb命令重安装到影子用户:
$ adb shell pm install -r --user 10 `adb shell pm path com.tencent.mm | awk -F':' '{print $2}'`
// "--user 10" 指定安装userId为10.


// 或者调用API:
//   PMS.installExistingPackageAsUser()
//   PMS.installPackageAsUser()

public void install(Context context, String packageName, int userId) {
	try {
		PackageManager pm = context.getPackageManager();
		Method installExistingPackageAsUser = PackageManager.class.getMethod("installExistingPackageAsUser",
				String.class, int.class);
		Object object = installExistingPackageAsUser.invoke(pm, packageName, userId);          
	} catch (NoSuchMethodException | IllegalAccessException | InvocationTargetException e) {
		e.printStackTrace();        
	}
}


第三步, 分别启动主用户和影子用户下的微信；



// 首先找到微信的首页Activity:
$ adb shell dumpsys package com.tencent.mm | grep "android.intent.action.MAIN:" -A 5 
      android.intent.action.MAIN:
        8f8990c com.tencent.mm/.ui.LauncherUI filter 3d82835
          Action: "android.intent.action.MAIN"
          Category: "android.intent.category.LAUNCHER"
          Category: "android.intent.category.MULTIWINDOW_LAUNCHER"
          AutoVerify=false
 // 得到首页Activity为"com.tencent.mm/.ui.LauncherUI"


 //于是启动影子用户下的微信为:
 $ adb shell am start --user 10 com.tencent.mm/.ui.LauncherUI
 // "--user 10"为指定userId为10, 不指定则默认为主用户, 即userId=0为默认.
 // ps: 并不是所有可指定userId的命令都这样设定.
 //     如 "am force-stop"命令是默认情况下杀所有用户下进程, 而非仅杀主用户下进程.



// 启动主用户下微信:
$ adb shell am start --user 0 com.tencent.mm/.ui.LauncherUI
 或:
$ adb shell am start com.tencent.mm/.ui.LauncherUI


检查微信进程:
$ adb shell ps | grep com.tencent.mm
u10_a110 19794 11620 2157444 185912 SyS_epoll_ 00eb0ce428 S com.tencent.mm
u10_a110 19882 11620 1832116 121932 SyS_epoll_ 00eb0ce428 S com.tencent.mm:push
u0_a110   19989 11620 2151704 194924 SyS_epoll_ 00eb0ce428 S com.tencent.mm
u0_a110   20072 11620 1830048 122600 SyS_epoll_ 00eb0ce428 S com.tencent.mm:push


可以看到,微信出现两组进程组, 一组在u0_a110用户下, 一组在u10_a110用户下。且观察界面可以看到他们同时运行在同一个桌面下。基于多用户, 我们很容易将创建了任意应用(微信)的分身乃至多开(多创建几个影子用户即可)。



Android原生多用户总结



1. 原生Android多用户在linux看来是不可见的上次应用行为. Android多用户的多用户与Linux的多用户是完全不同的概念；



2. 原生多用户可以做到2种型式的多开：



一种是显式的切换用户后,应用运行在不同用户下的多开, 跟传统PC上的多用户类似;

另一种是不用切换用户,当前用户下运行N个影子用户,一个主用户跟N个影子用户共同运行于同一桌面下.



3. 上面忽略了一些信息, 如：



原生提供了DevicePolicyManager接口, 用于控制新创建用户的权限,比如可以控制新用户是否可以访问网络等;

创建用户时flags参数可以控制新用户的一些属性, 比如此用户是"访客用户",则该用户退出时完全清理数据。这些细节对我们系统开发人员来说并不重要, 我们很容易可以按需定制, 稍做了解知道有这么回事就好。
