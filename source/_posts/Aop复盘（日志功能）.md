---
title: Aop复盘（日志功能）
date: 2023-10-20 13:52:21
tags: 
    - Spring
    - Aop
description: Aop 实现日志功能 ## 页面描述
keywords: aop  ## 关键字
comments: true  ## 评论模块
cover: /top_img/Aop日志复盘top_img.jpeg  ## 文章缩略图
# top_img:  ## 顶部图片
aside: false  ## 显示侧边栏（默认 true）
---

### Aop 实现+自定义注解实现 异步日志记录

##### 设计思路
通过自定义注解及 aop 切面 + huto 工具类实现异步记录操作日志

---
1. 定义日志对象 实体类 （用来保存、查询 等操作）
```
/**
 * 日志对象
 * cloud api 用到的接口传输对象
 */
@Data
public class LogDTO implements Serializable {

    private static final long serialVersionUID = 8482720462943906924L;

    /**内容*/
    private String logContent;

    /**日志类型(0:操作日志;1:登录日志;2:定时任务)  */
    private Integer logType;

    /**操作类型(1:添加;2:修改;3:删除;) */
    private Integer operateType;

    /**登录用户 */
    private LoginUser loginUser;

    private String id;
    private String createBy;
    private Date createTime;
    private Long costTime;
    private String ip;

    /**请求参数 */
    private String requestParam;

    /**请求类型*/
    private String requestType;

    /**请求路径*/
    private String requestUrl;

    /**请求方法 */
    private String method;

    /**操作人用户名称*/
    private String username;

    /**操作人用户账户*/
    private String userid;


    .....(省略构造器)
}
```
2. 创建日志切面处理类
```
/**
 * 系统日志，切面处理类
 */
@Aspect
@Component
public class AutoLogAspect {

    @Resource
    private BaseCommonService baseCommonService;

    // 切入点
    @Pointcut("@annotation(org.jeecg.common.aspect.annotation.AutoLog)")
    public void logPointCut() {

    }

    ... 省去 前置通知、后置通知、环绕通知


    // @Around注解可以用来在调用一个具体方法前和调用后来完成一些具体的任务
    @Around("logPointCut()")
    public Object around(ProceedingJoinPoint point) throws Throwable {
        long beginTime = System.currentTimeMillis();
        //执行方法
        Object result = point.proceed();
        //执行时长(毫秒)
        long time = System.currentTimeMillis() - beginTime;

        //保存日志
        saveSysLog(point, time, result);

        return result;
    }

    private void saveSysLog(ProceedingJoinPoint joinPoint, long time, Object obj) {
        // JoinPoint类 用来获取代理类和被代理类的信息 [后期学习](http://baidu.physton.com/?q=JoinPoint%E7%B1%BB%E7%9A%84%E4%BD%9C%E7%94%A8)
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();


        LogDTO dto = new LogDTO();
        // 通过反射获取对象
        AutoLog syslog = method.getAnnotation(AutoLog.class);

        if(syslog != null){
            //update-begin-author:taoyan date:
            String content = syslog.value();
            if(syslog.module()== ModuleType.ONLINE){
                // 记录操作状态
                content = getOnlineLogContent(obj, content);
            }
            //注解上的描述,操作日志内容
            dto.setLogType(syslog.logType());
            dto.setLogContent(content);
        }

        // 通过反射获取请求的类名和方法名
        String className = joinPoint.getTarget().getClass().getName();
        String methodName = signature.getName();
        dto.setMethod(className + "." + methodName + "()");

        //  设置操作类型
        if (CommonConstant.LOG_TYPE_2 == dto.getLogType()) {
            dto.setOperateType(getOperateType(methodName, syslog.operateType()));
        }

        //获取request
        HttpServletRequest request = SpringContextUtils.getHttpServletRequest();
        //请求的参数
        dto.setRequestParam(getReqestParams(request,joinPoint));
        //设置IP地址
        dto.setIp(IPUtils.getIpAddr(request));

        //获取登录用户信息
        LoginUser sysUser = (LoginUser) SecurityUtils.getSubject().getPrincipal();
        if(sysUser!=null){
            dto.setUserid(sysUser.getUsername());
            dto.setUsername(sysUser.getRealname());

        }
        //耗时
        dto.setCostTime(time);
        dto.setCreateTime(new Date());

        //保存系统日志 (注入serverImp 层实现异步保存日志)
        baseCommonService.addLog(dto);
    }
``` 

```
/**
 * online日志内容拼接
 */
private String getOnlineLogContent(Object obj, String content){
    if (Result.class.isInstance(obj)){
        Result res = (Result)obj;
        String msg = res.getMessage();
        if(res.isSuccess()){
            content+= ","+(oConvertUtils.isEmpty(msg)?"操作成功":msg);
        }else{
            content+= ","+(oConvertUtils.isEmpty(msg)?"操作失败":msg);
        }
    }
    return content;
}
```

```
/**
 * 获取操作类型(查询、添加、删除、更新...)
 */
private int getOperateType(String methodName,int operateType) {
    if (operateType > 0) {
        return operateType;
    }
    if (methodName.startsWith("list")) {
        return CommonConstant.OPERATE_TYPE_1;
    }
    if (methodName.startsWith("add")) {
        return CommonConstant.OPERATE_TYPE_2;
    }
    if (methodName.startsWith("edit")) {
        return CommonConstant.OPERATE_TYPE_3;
    }
    if (methodName.startsWith("delete")) {
        return CommonConstant.OPERATE_TYPE_4;
    }
    if (methodName.startsWith("import")) {
        return CommonConstant.OPERATE_TYPE_5;
    }
    if (methodName.startsWith("export")) {
        return CommonConstant.OPERATE_TYPE_6;
    }
    return CommonConstant.OPERATE_TYPE_1;
}
```
3. 异步保存日志
```
...... (实现类)
@Override
public void addLog(LogDTO logDTO) {
    if(oConvertUtils.isEmpty(logDTO.getId())){
        logDTO.setId(String.valueOf(IdWorker.getId()));
    }
    //保存日志（ 通过huto 工具包的异步工具类捕获，防止获处理数据太大存储失败，导致业务失败）
    try {
        ThreadUtil.execAsync(() -> {
            baseCommonMapper.saveLog(logDTO);
        });
    } catch (Exception e) {
        log.warn(" LogContent length : "+logDTO.getLogContent().length());
        log.warn(e.getMessage());
    }
}
```

