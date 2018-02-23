---
title: Android 学习笔记二：代码组织
updated: 2017-12-12 21:32
---

### 问题
在进行模块化的时候，试图将诸如`SearchListener`, `CallManager` 的模块从`MainActivity`中拆出来，然而在响应事件的时候，不可避免的需要改变其他资源状态，那么就需要获取其句柄。由此还需要把`MainActivity`作为句柄传入代码。

更令人纠结的是，由于存在一些异步事件（如申请权限），其触发在模块（`CallManager`）中，其接受在主体视图（`MainActivity`）中，又需要将保有的参数重新传回`MainActivity`，如此绕来绕去，让我不禁反思我模块化时候存在的问题。

### 细节

```Java
CallManager.java

public boolean playCall(String phoneNumber) {
    if (ContextCompat.checkSelfPermission(activity,
            Manifest.permission.CALL_PHONE)
            != PackageManager.PERMISSION_GRANTED) {
        ActivityCompat.requestPermissions(activity,
                new String[]{Manifest.permission.CALL_PHONE},
                CALL_PHONE_REQUEST);

        MainActivity ma = (MainActivity) activity;
        **ma.setPhoneNumber(phoneNumber);**

        return false;
...
}

MainActivity.java

private String phoneNumber = null;
public void setPhoneNumber(String number) {this.phoneNumber = number;}
public void onRequestPermissionsResult(int requestCode,
                                       String permissions[], int[] grantResults) {
    switch (requestCode) {
        case CallManager.CALL_PHONE_REQUEST:
            if (grantResults.length > 0
                    && grantResults[0] == PackageManager.PERMISSION_GRANTED) {

                if (phoneNumber != null ) {
                    CallManager callManager = new CallManager(this);
                    callManager.playCall(phoneNumber);

                }
            }
            break;

        ...
    }
}

```
在`Activity`中接收到call event调用`CallManager`，然后在其中请求权限，利用Set方法，将`PhoneNumber`传回`Activity`。初步想法是可以将请求权限、被授予权限后执行回调函数这种和`Activity`紧耦合的代码方法写在`MainActivity`中，然后`CallManager`只处理call逻辑。但仍需要一个类变量来保存电话号码，以进行从请求权限到执行回调之前的参数传递工作。











