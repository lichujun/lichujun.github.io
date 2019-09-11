---
layout: post
title: Spring AOP统一打印日志
categories: Spring
tags: log
---


### 一、定义注解

#### 1. 创建注解，注解作用域只在方法上
```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ServerLog {
    // 方法入参描述
    String reqDesc() default "";
    // 方法出参描述
    String resDesc() default "";
    // 日志打印范围，默认全打印
    LogRange logRange() default LogRange.ALL;
}
```



#### 2. 定义LogRange枚举类，定义打印日志的范围
```
@Getter
@AllArgsConstructor
public enum LogRange {
    // 出入参都不打印日志
    NONE(false, false),
    // 入参打印日志
    REQ(true, false),
    // 出参打印日志
    RES(false, true),
    // 出入参都打印日志
    ALL(true, true),
    ;
    private boolean printReq;
    private boolean printRes;
}
```

### 二、AOP拦截日志注解

#### 1. 拦截日志注解

```
@Around(value = "@annotation(serverLog)", argNames = "joinPoint,serverLog")
public Object printLog(ProceedingJoinPoint joinPoint, ServerLog serverLog) throws Throwable {
    LogRange logRange = serverLog.logRange();
    if (logRange.isPrintReq()) {
        String reqDesc = serverLog.reqDesc();
      	// 通过CodeSignature获取参数名称
        CodeSignature codeSignature = (CodeSignature) joinPoint.getSignature();
        String[] parameterNames = codeSignature.getParameterNames();
      	// 如果参数数组不为空，则打印参数名称和参数实体
        if (ArrayUtils.isNotEmpty(parameterNames)) {
            StringBuilder logMessage = new StringBuilder(reqDesc);
            Object[] parameterMessages = new Object[parameterNames.length * 2];
            Object[] parameterObjs = joinPoint.getArgs();
            for (int i = 0; i < parameterNames.length; i++) {
                logMessage.append(LOG_BLOCK);
                parameterMessages[2 * i] = parameterNames[i];
                parameterMessages[2 * i + 1] = JSON.toJSONString(parameterObjs[i]);
            }
            log.info(logMessage.toString(), parameterMessages);
        } else {
            log.info(reqDesc);
        }
    }
    Object ret = joinPoint.proceed();
    if (logRange.isPrintRes()) {
        String resDesc = serverLog.resDesc() + " {}";
        log.info(resDesc, JSON.toJSONString(ret));
    }
    return ret;
}
```

