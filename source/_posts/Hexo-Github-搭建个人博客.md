---
title: Hexo Github 搭建个人博客
date: 2018-11-11 23:39:25
tags: dubbo
---

## dubbo群容错与负载均衡策略 ##

### 一、dubbo容错策略 ###

### 1、概述

>   进行系统设计时，不仅要考虑正常逻辑下代码如何执行，还要考虑异常情况下代码逻辑是什么样的。当服务消费方调用服务提供方出现错误时该如何处理，dubbo提供了很多容错方案，缺省模式为 **failover**

### 2、容错模式

- 2.1、**Failover Cluster：失败重试**

>   当服务消费方调用服务提供方的接口失败后自动切换到其他服务提供者的服务器重试。这个模式通常适用于读操作或者具有幂等的写操作

>      <dubbo:reference retries="2" />

>   如上配置，如果第一个调用失败后会重试两次，所有共有3次调用

```
 public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    	List<Invoker<T>> copyinvokers = invokers;
    	checkInvokers(copyinvokers, invocation);
    	// 重试次数
        int len = getUrl().getMethodParameter(invocation.getMethodName(), Constants.RETRIES_KEY, Constants.DEFAULT_RETRIES) + 1;
        if (len <= 0) {
            len = 1;
        }
        // retry loop.
        RpcException le = null; // last exception.
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyinvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
        for (int i = 0; i < len; i++) {
        	//重试时，进行重新选择，避免重试时invoker列表已发生变化.
        	//注意：如果列表发生了变化，那么invoked判断会失效，因为invoker示例已经改变
        	if (i > 0) {
        		checkWheatherDestoried();
        		copyinvokers = list(invocation);
        		//重新检查一下
        		checkInvokers(copyinvokers, invocation);
        	}
            Invoker<T> invoker = select(loadbalance, invocation, copyinvokers, invoked);
            invoked.add(invoker);
            RpcContext.getContext().setInvokers((List)invoked);
            try {
                Result result = invoker.invoke(invocation);
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
    }

```


- 2.2、**Failfast Cluster：快速失败**
 
>  当服务消费方调用服务提供者失败后，立即报错，也就是只调用一次。通常这种模式用于非幂等性的写操作。

```
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        try {
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            if (e instanceof RpcException && ((RpcException)e).isBiz()) { // biz exception.
                throw (RpcException) e;
            }
            throw new RpcException(e instanceof RpcException ? ((RpcException)e).getCode() : 0, "Failfast invoke providers " + invoker.getUrl() + " " + loadbalance.getClass().getSimpleName() + " select from all providers " + invokers + " for service " + getInterface().getName() + " method " + invocation.getMethodName() + " on consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e.getCause() != null ? e.getCause() : e);
        }
    }
```

- 2.3、**Failsafe Cluster：失败安全**

>  失败安全，出现异常时，直接忽略。通常用于写入审计日志等操作。

```
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        try {
            checkInvokers(invokers, invocation);
            Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            logger.error("Failsafe ignore exception: " + e.getMessage(), e);
            return new RpcResult(); // ignore
        }
    }
```

- 2.4、**Failback Cluster**

>  失败自动恢复，后台记录失败请求，定时重发。通常用于消息通知操作。

```
    protected Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        try {
            checkInvokers(invokers, invocation);
            Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            logger.error("Failback to invoke method " + invocation.getMethodName() + ", wait for retry in background. Ignored exception: "
                                 + e.getMessage() + ", ", e);
            addFailed(invocation, this);
            return new RpcResult(); // ignore
        }
    }
```

```
 // 创建一个定长线程池，支持定时及周期性任务执行
 private final ScheduledExecutorService scheduledExecutorService = Executors.newScheduledThreadPool(2, new NamedThreadFactory("failback-cluster-timer", true));
 
 private void addFailed(Invocation invocation, AbstractClusterInvoker<?> router) {
        if (retryFuture == null) {
            synchronized (this) {
                if (retryFuture == null) {
                    retryFuture = scheduledExecutorService.scheduleWithFixedDelay(new Runnable() {

                        public void run() {
                            // 收集统计信息
                            try {
                                retryFailed();
                            } catch (Throwable t) { // 防御性容错
                                logger.error("Unexpected error occur at collect statistic", t);
                            }
                        }
                    }, RETRY_FAILED_PERIOD, RETRY_FAILED_PERIOD, TimeUnit.MILLISECONDS);
                }
            }
        }
        failed.put(invocation, router);
    }
```

```
    void retryFailed() {
        if (failed.size() == 0) {
            return;
        }
        for (Map.Entry<Invocation, AbstractClusterInvoker<?>> entry : new HashMap<Invocation, AbstractClusterInvoker<?>>(
                                                                                                                         failed).entrySet()) {
            Invocation invocation = entry.getKey();
            Invoker<?> invoker = entry.getValue();
            try {
                invoker.invoke(invocation);
                failed.remove(invocation);
            } catch (Throwable e) {
                logger.error("Failed retry to invoke method " + invocation.getMethodName() + ", waiting again.", e);
            }
        }
    }
```

- 2.5、**Forking Cluster**

>  并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 forks="2" 来设置最大并行数。

```
 @SuppressWarnings({ "unchecked", "rawtypes" })
    public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        final List<Invoker<T>> selected;
        final int forks = getUrl().getParameter(Constants.FORKS_KEY, Constants.DEFAULT_FORKS);
        final int timeout = getUrl().getParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
        if (forks <= 0 || forks >= invokers.size()) {
            selected = invokers;
        } else {
            selected = new ArrayList<Invoker<T>>();
            for (int i = 0; i < forks; i++) {
                //在invoker列表(排除selected)后,如果没有选够,则存在重复循环问题.见select实现.
                Invoker<T> invoker = select(loadbalance, invocation, invokers, selected);
                if(!selected.contains(invoker)){//防止重复添加invoker
                    selected.add(invoker);
                }
            }
        }
        RpcContext.getContext().setInvokers((List)selected);
        final AtomicInteger count = new AtomicInteger();
        final BlockingQueue<Object> ref = new LinkedBlockingQueue<Object>();
        for (final Invoker<T> invoker : selected) {
            executor.execute(new Runnable() {
                public void run() {
                    try {
                        Result result = invoker.invoke(invocation);
                        ref.offer(result);
                    } catch(Throwable e) {
                        int value = count.incrementAndGet();
                        if (value >= selected.size()) {
                            ref.offer(e);
                        }
                    }
                }
            });
        }
        try {
            Object ret = ref.poll(timeout, TimeUnit.MILLISECONDS);
            if (ret instanceof Throwable) {
                Throwable e = (Throwable) ret;
                throw new RpcException(e instanceof RpcException ? ((RpcException)e).getCode() : 0, "Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e.getCause() != null ? e.getCause() : e);
            }
            return (Result) ret;
        } catch (InterruptedException e) {
            throw new RpcException("Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e);
        }
    }
```

- 2.6、**Broadcast Cluster**

>  广播调用所有提供者，逐个调用，任意一台报错则报错。通常用于通知所有提供者更新缓存或日志等本地资源信息。

```
  @SuppressWarnings({ "unchecked", "rawtypes" })
    public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        RpcContext.getContext().setInvokers((List)invokers);
        RpcException exception = null;
        Result result = null;
        for (Invoker<T> invoker: invokers) {
            try {
                result = invoker.invoke(invocation);
            } catch (RpcException e) {
                exception = e;
                logger.warn(e.getMessage(), e);
            } catch (Throwable e) {
                exception = new RpcException(e.getMessage(), e);
                logger.warn(e.getMessage(), e);
            }
        }
        if (exception != null) {
            throw exception;
        }
        return result;
    }
```