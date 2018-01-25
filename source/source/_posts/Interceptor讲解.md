---
title: Interceptor讲解
date: 2017-07-10 13:31:51
tags: java
---

#### 拦截器接口
```java
public interface HandlerInterceptor {
    boolean preHandle(HttpServletRequest var1, HttpServletResponse var2, Object var3) throws Exception;

    void postHandle(HttpServletRequest var1, HttpServletResponse var2, Object var3, ModelAndView var4) throws Exception;

    void afterCompletion(HttpServletRequest var1, HttpServletResponse var2, Object var3, Exception var4) throws Exception;
}
```
- preHandle: 预处理回调方法，实现处理器的预处理(如登录检查)，第三个参数为响应的处理器。返回`TRUE`，继续执行；返回`FALSE`，中断流程。

- postHandle: 实现处理器的后处理，但在渲染视图之前。

- afterCompletion: 整个请求处理完毕回调方法，即在视图渲染完毕时回调

<!-- more -->

#### 拦截器适配器（缺省适配器实现）
在实现自定义的拦截器时，无需实现HandlerInterceptor，只需继承HandlerInterceptorAdapter，并实现所需功能即可。

```java
//异步拦截器，继承自HandlerInterceptor
public interface AsyncHandlerInterceptor extends HandlerInterceptor {
    void afterConcurrentHandlingStarted(HttpServletRequest var1, HttpServletResponse var2, Object var3) throws Exception;
}

//拦截器适配器，提供各方法默认实现
public abstract class HandlerInterceptorAdapter implements AsyncHandlerInterceptor {
    public HandlerInterceptorAdapter() {
    }

    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        return true;
    }

    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
    }

    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
    }

    public void afterConcurrentHandlingStarted(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    }
}
```

#### 拦截器执行流程
拦截器的执行流程，可以通过DispatcherServlet的源码进行分析。

```java
// DispatcherServlet类中doDispatch方法部分源码解析
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        HttpServletRequest processedRequest = request;
        HandlerExecutionChain mappedHandler = null;
        boolean multipartRequestParsed = false;
        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

        try {
            try {
                ModelAndView err = null;
                Object dispatchException = null;

                try {
                    //判断请求是否为MultipartHttpServletRequest，是，转换为MultipartHttpServletRequest请求实体
                    processedRequest = this.checkMultipart(request);
                    multipartRequestParsed = processedRequest != request;
                    
                    //通过HandlerMapping获取HandlerExecutionChain，HandlerExecutionChain包含拦截器列表和处理器
                    mappedHandler = this.getHandler(processedRequest);
                    if(mappedHandler == null || mappedHandler.getHandler() == null) {
                        this.noHandlerFound(processedRequest, response);
                        return;
                    }

                    //获取处理器适配器
                    HandlerAdapter err1 = this.getHandlerAdapter(mappedHandler.getHandler());
                    String method = request.getMethod();
                    boolean isGet = "GET".equals(method);
                    if(isGet || "HEAD".equals(method)) {
                        long lastModified = err1.getLastModified(request, mappedHandler.getHandler());
                        if(this.logger.isDebugEnabled()) {
                            this.logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
                        }

                        if((new ServletWebRequest(request, response)).checkNotModified(lastModified) && isGet) {
                            return;
                        }
                    }

                   
                   //执行拦截器的preHandle()方法。如果返回FALSE，则调用所有已经执行过拦截器的afterCompletion方法。(参考代码1.1，代码1.2)
                   if(!mappedHandler.applyPreHandle(processedRequest, response)) {
                        return;
                    }

                    //请求处理
                    err = err1.handle(processedRequest, response, mappedHandler.getHandler());
                    if(asyncManager.isConcurrentHandlingStarted()) {
                        return;
                    }

                   //设置视图名
                   this.applyDefaultViewName(processedRequest, err);
                   
                   //执行拦截器的postHandle()方法
                   mappedHandler.applyPostHandle(processedRequest, response, err);
                } catch (Exception var20) {
                    dispatchException = var20;
                } catch (Throwable var21) {
                    dispatchException = new NestedServletException("Handler dispatch failed", var21);
                }

               //视图渲染，并执行拦截器的afterCompletion()方法（参考代码1.3）
               this.processDispatchResult(processedRequest, response, mappedHandler, err, (Exception)dispatchException);
            } catch (Exception var22) {
                this.triggerAfterCompletion(processedRequest, response, mappedHandler, var22);
            } catch (Throwable var23) {
                this.triggerAfterCompletion(processedRequest, response, mappedHandler, new NestedServletException("Handler processing failed", var23));
            }

        } finally {
            if(asyncManager.isConcurrentHandlingStarted()) {
                if(mappedHandler != null) {
                    mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
                }
            } else if(multipartRequestParsed) {
                this.cleanupMultipart(processedRequest);
            }

        }
    }

```
---
代码1.1
```java
//代码1.1
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
        HandlerInterceptor[] interceptors = this.getInterceptors();
        if(!ObjectUtils.isEmpty(interceptors)) {
            //每次循环，设置interceptorIndex的值，triggerAfterCompletion()方法执行使用此字段
            for(int i = 0; i < interceptors.length; this.interceptorIndex = i++) {
                HandlerInterceptor interceptor = interceptors[i];
                if(!interceptor.preHandle(request, response, this.handler)) {
                    this.triggerAfterCompletion(request, response, (Exception)null);
                    return false;
                }
            }
        }

        return true;
    }
```
---
代码1.2

```java
//代码1.2
void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, Exception ex) throws Exception {
        HandlerInterceptor[] interceptors = this.getInterceptors();
        if(!ObjectUtils.isEmpty(interceptors)) {
            //使用interceptorIndex字段，该字段在applyPreHandle()方法中进行设值
            for(int i = this.interceptorIndex; i >= 0; --i) {
                HandlerInterceptor interceptor = interceptors[i];

                try {
                    interceptor.afterCompletion(request, response, this.handler, ex);
                } catch (Throwable var8) {
                    logger.error("HandlerInterceptor.afterCompletion threw exception", var8);
                }
            }
        }

    }
```

---
代码1.3
```java
//代码1.3
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response, HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {
        boolean errorView = false;
        if(exception != null) {
            if(exception instanceof ModelAndViewDefiningException) {
                this.logger.debug("ModelAndViewDefiningException encountered", exception);
                mv = ((ModelAndViewDefiningException)exception).getModelAndView();
            } else {
                Object handler = mappedHandler != null?mappedHandler.getHandler():null;
                mv = this.processHandlerException(request, response, handler, exception);
                errorView = mv != null;
            }
        }

        if(mv != null && !mv.wasCleared()) {
            //视图渲染
            this.render(mv, request, response);
            if(errorView) {
                WebUtils.clearErrorRequestAttributes(request);
            }
        } else if(this.logger.isDebugEnabled()) {
            this.logger.debug("Null ModelAndView returned to DispatcherServlet with name \'" + this.getServletName() + "\': assuming HandlerAdapter completed request handling");
        }

        if(!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
            if(mappedHandler != null) {
               
               //执行拦截器的afterCompletion()方法
               mappedHandler.triggerAfterCompletion(request, response, (Exception)null);
            }

        }
    }
```
总结：

- 正常流程

intercepor.preHandle ->  获取HandleAdapter -> intercepor.postHandle -> View 渲染 ->  intercepor.afterCompletion

- 异常流程

intercepor.preHandle -> 部分intercepor.afterCompletion

#### 应用

- 性能监控
- 登录检测
- 日志记录  ...等

性能监控实现

```java
public class StopWatchHandlerInterceptor extends HandlerInterceptorAdapter {
    //使用 ThreadLocal，是为了解决线程不安全问题。它是线程绑定的变量，提供线程局部变量(一个线程一个 ThreadLocal，A 线程的 ThreadLocal 只能看到 A 线程的 ThreadLocal，不能看到 B 线程的 ThreadLocal)。
        private NamedThreadLocal<Long> startTimeThreadLocal =
                new NamedThreadLocal<Long>("StopWatch-StartTime");

        @Override
        public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
            long beginTime = System.currentTimeMillis();//1、开始时间
            startTimeThreadLocal.set(beginTime);//线程绑定变量(该数据只有当前请求的线程可见)
            return true;//继续流程
        }

        @Override
        public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
            long endTime = System.currentTimeMillis();//2、结束时间
            System.out.println("方法执行时间为:" + (endTime - startTimeThreadLocal.get()));
        }
    }

```
