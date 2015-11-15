---
layout: post
title:  "UserHandler类分析"
date:   2015-11-15 16:46:51 +0800
categories: android update
---
##概述  
>此类代表设备上的一个用户。在Android中，开始支持多用户。
其中
```
public static final int PER_USER_RANGE = 100000;
```
代表一个用户可以开启多少个进程，对这些进程，也进行了一些区分：  

|进程ID类型|进程ID范围  
|--------|-----------  
|系统进程|从SYSTEM_UID开始  
|应用进程|FIRST_APPLICATION_UID ~ LAST_APPLICATION_UID  
|隔离进程（没有自己的权限）|FIRST_ISOLATED_UID ~ LAST_ISOLATED_UID  
|共享资源的GID|FIRST_SHARED_APPLICATION_GID ~ FIRST_SHARED_APPLICATION_GID  

此类提供了获取用户ID等一些方法。
```java

    /**
     * Returns the user id for a given uid.
     * @hide
     */
    public static final int getUserId(int uid) {
        if (MU_ENABLED) {
            return uid / PER_USER_RANGE;
        } else {
            return 0;
        }
    }

    /** @hide */
    public static final int getCallingUserId() {
        return getUserId(Binder.getCallingUid());
    }

    /**
     * Returns the uid that is composed from the userId and the appId.
     * @hide
     */
    public static final int getUid(int userId, int appId) {
        if (MU_ENABLED) {
            return userId * PER_USER_RANGE + (appId % PER_USER_RANGE);
        } else {
            return appId;
        }
    }

    /**
     * Returns the app id (or base uid) for a given uid, stripping out the user id from it.
     * @hide
     */
    public static final int getAppId(int uid) {
        return uid % PER_USER_RANGE;
    }

    /**
     * Returns the shared app gid for a given uid or appId.
     * @hide
     */
    public static final int getSharedAppGid(int id) {
        return Process.FIRST_SHARED_APPLICATION_GID + (id % PER_USER_RANGE)
                - Process.FIRST_APPLICATION_UID;
    }
```

##判断一个应用是否foreground
背景：某些API的调用只能由当前处于Active状态的应用调用。
摘自BluetoothManagerService: 
```java
    private boolean checkIfCallerIsForegroundUser() {
        int foregroundUser;
        int callingUser = UserHandle.getCallingUserId();
        int callingUid = Binder.getCallingUid();
        long callingIdentity = Binder.clearCallingIdentity();
        int callingAppId = UserHandle.getAppId(callingUid);
        boolean valid = false;
        try {
            foregroundUser = ActivityManager.getCurrentUser();
            valid = (callingUser == foregroundUser) ||
                    callingAppId == Process.NFC_UID;
            if (DBG) {
                Log.d(TAG, "checkIfCallerIsForegroundUser: valid=" + valid
                    + " callingUser=" + callingUser
                    + " foregroundUser=" + foregroundUser);
            }
        } finally {
            Binder.restoreCallingIdentity(callingIdentity);
        }
        return valid;
    }


 public static boolean checkCaller() {
        boolean ok;
        // Get the caller's user id then clear the calling identity
        // which will be restored in the finally clause.
        int callingUser = UserHandle.getCallingUserId();
        long ident = Binder.clearCallingIdentity();

        try {
            // With calling identity cleared the current user is the foreground user.
            int foregroundUser = ActivityManager.getCurrentUser();
            ok = (foregroundUser == callingUser);
        } catch (Exception ex) {
            Log.e(TAG,"checkIfCallerIsSelfOrForegroundUser: Exception ex=" + ex);
            ok = false;
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
        return ok;
    }

```
