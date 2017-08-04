---
title: Google Analytics自定义异常格式
date: 2016-01-08 21:39:46
categories: Google Analytics
tags: [Android, Google Analytics]
---

## 自动配置
可以在xml配置文件中直接配置以下字段，即可开启异常捕获与上传：
```xml
<bool name="ga_reportUncaughtExceptions">true</bool>
```


## 问题来了
用系统的默认`ExceptionParser`会有下面2个问题：
    1. 格式是按照GA默认的样式
    2. 如果开启了混淆，就无法知道那个类文件出问题了

默认格式如下：异常名 (@类名:方法名:行数) {所在线程}
```
NullPointerException (@c:run:35) {main}
```
GA默认的异常解析类是`StandardExceptionParser`，大家可以查看其中的`getDescription(Throwable cause, StackTraceElement element, String threadName)`方法实现，里面定义了异常的格式。


## 解决方法
当然就是实现自己的`ExceptionParser`，并设置为`ExceptionReporter`内置的解析器，步骤如下：
    
- 继承`StandardExceptionParser`或者自己实现`ExceptionParser`接口。这里选用继承`StandardExceptionParser`，因为这个类里面为我们做了很多操作，例如其中里面有个getBestStackTraceElement()函数，他会从`Throwable`对象中取出最关键的`StackTraceElement`对象，方便后续进行处理。（注意下面重写了getDescription()3参数的方法，因为2参数那个方法也是调用它）
```java
public class GAExceptionParser extends StandardExceptionParser {
    public GAExceptionParser(Context context) {
        this(context, new ArrayList<String>());
    }
    public GAExceptionParser(Context context, Collection<String> additionalPackages) {
        super(context, additionalPackages);
    }
    @Override
    protected String getDescription(Throwable cause, StackTraceElement element, String threadName) {
        StringBuilder sb = new StringBuilder();
        sb.append(cause.getClass().getSimpleName());
        if (element != null) {
            sb.append(" ");
            sb.append(element.toString());
        }

        if (threadName != null) {
            sb.append(String.format(" {%s}", new Object[]{threadName}));
        }

        return sb.toString();
    }
}
```

- 配置`ExceptionReporter`并将上述`GAExceptionParser `设置为内置的解析器：
```java
ExceptionReporter gaHandler = new ExceptionReporter(
    mTracker,                                     // Currently used Tracker.
    Thread.getDefaultUncaughtExceptionHandler(),  // Current default uncaught exception handler.
    context);                                    // Context of the application.
gaHandler.setExceptionParser(new GAExceptionParser(context));
// Make our gaHandler to the new default uncaught exception handler.
Thread.setDefaultUncaughtExceptionHandler(gaHandler);
```

- 通过上述设置后，奔溃的时候就会按照我们自定义的格式发送到后台服务器中了，修改后在后台看到的格式如下：
```
NullPointerException com.codezjx.googleanalyticstest.c.run(SecondActivity.java:35) {main}
```


## 参考
https://developers.google.com/analytics/devguides/collection/android/v4/exceptions