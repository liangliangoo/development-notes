# nacos 服务注册 & 心跳检测

## 临时节点的注册
注册源码
```java
@Override
public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {
    String groupedServiceName = NamingUtils.getGroupedName(serviceName, groupName);
    //临时实例
    if (instance.isEphemeral()) {
        //创建BeatTask心跳线程
        BeatInfo beatInfo = beatReactor.buildBeatInfo(groupedServiceName, instance);
        beatReactor.addBeatInfo(groupedServiceName, beatInfo);
    }
    //向Nacos注册实例
    serverProxy.registerService(groupedServiceName, groupName, instance);
}
```

BeatTask 心跳线程源码分析
```java
@Override
public void run() {

    if (beatInfo.isStopped()) {
        return;
    }
    //下一次心跳时间
    long nextTime = beatInfo.getPeriod();
    try {
        //发送心跳
        JsonNode result = serverProxy.sendBeat(beatInfo, BeatReactor.this.lightBeatEnabled);
        long interval = result.get("clientBeatInterval").asLong();
        boolean lightBeatEnabled = false;
        if (result.has(CommonParams.LIGHT_BEAT_ENABLED)) {
            lightBeatEnabled = result.get(CommonParams.LIGHT_BEAT_ENABLED).asBoolean();
        }
        BeatReactor.this.lightBeatEnabled = lightBeatEnabled;
        //使用服务器返回的心跳时间
        if (interval > 0) {
            nextTime = interval;
        }
        int code = NamingResponseCode.OK;
        if (result.has(CommonParams.CODE)) {
            code = result.get(CommonParams.CODE).asInt();
        }
        //服务器返回实例不存在，重新注册实例
        if (code == NamingResponseCode.RESOURCE_NOT_FOUND) {
            Instance instance = new Instance();
            instance.setPort(beatInfo.getPort());
            instance.setIp(beatInfo.getIp());
            instance.setWeight(beatInfo.getWeight());
            instance.setMetadata(beatInfo.getMetadata());
            instance.setClusterName(beatInfo.getCluster());
            instance.setServiceName(beatInfo.getServiceName());
            instance.setInstanceId(instance.getInstanceId());
            instance.setEphemeral(true);
            try {
                //重新注册实例
                serverProxy.registerService(beatInfo.getServiceName(),
                                            NamingUtils.getGroupName(beatInfo.getServiceName()), instance);
            } catch (Exception ignore) {
            }
        }
    } catch (NacosException ex) {
        NAMING_LOGGER.error("[CLIENT-BEAT] failed to send beat: {}, code: {}, msg: {}",
                            JacksonUtils.toJson(beatInfo), ex.getErrCode(), ex.getErrMsg());

    }
    //开启下一次心跳
    executorService.schedule(new BeatTask(beatInfo), nextTime, TimeUnit.MILLISECONDS);
}
```

## 持久化节点
服务的探活主要是通过nacos服务端主动探测，探活支持
1. HTTP：HTTP通过检测返回200状态码标记是否健康
2. TCP：TPC通过Channel连接方式标记是否健康
3. Mysql：Mysql则保证当前节点为主节点，可用于主从切换场景

## 服务发现整体流程
NacosNamingService.getAllInstances 方法中 如果subscribe（表示是否需要订阅节点的变化） 参数为空false，
那么就会调用getServiceInfoDirectlyFromServer 从nacos 拉去服务列表 加载到本地内存当中</br>
如果 subscribe 参数为true，调用getServiceInfo ，如果开启了 failover 模式，则从本地文件中读取实例列表，否则先从本地serviceInfoMap 中获取服务列表，如果内存中不存在 就发起http请求（GET /instance/list ）去nacos 服务器中获取，放入到内存中，然后开启一个定时任务UpdateTask 定时的拉去nacos中服务列表信息，更新到缓存当中，当serviceInfoMap中的数据发生变化，向外广播 InstancesChangeEvent 事件并最终执行 EventListener 的 onEvent 方法将数据通知给订阅方。

```java
@Override
public List<Instance> getAllInstances(String serviceName, String groupName, List<String> clusters,
                                      boolean subscribe) throws NacosException {

    ServiceInfo serviceInfo;
    if (subscribe) {
        // 调用 getServiceInfo 方法获取实例列表，并定时拉取最新的实例列表
        serviceInfo = hostReactor.getServiceInfo(NamingUtils.getGroupedName(serviceName, groupName),StringUtils.join(clusters, ","));
    } else {
        //调用 getServiceInfoDirectlyFromServer 方法从Nacos服务器中获取实例列表并更新到本地内存中
        serviceInfo = hostReactor
            .getServiceInfoDirectlyFromServer(NamingUtils.getGroupedName(serviceName, groupName),StringUtils.join(clusters, ","));
    }
    List<Instance> list;
    if (serviceInfo == null || CollectionUtils.isEmpty(list = serviceInfo.getHosts())) {
        return new ArrayList<Instance>();
    }
    return list;
}
```

```java
 public ServiceInfo getServiceInfo(final String serviceName, final String clusters) {
     NAMING_LOGGER.debug("failover-mode: " + failoverReactor.isFailoverSwitch());
     String key = ServiceInfo.getKey(serviceName, clusters);
     //failover-mode开关打开
     if (failoverReactor.isFailoverSwitch()) {
         //返回本地文件中的实例列表
         return failoverReactor.getService(key);
     }
     //从内存serviceInfoMap中获取实例
     ServiceInfo serviceObj = getServiceInfo0(serviceName, clusters);
     if (null == serviceObj) {
         serviceObj = new ServiceInfo(serviceName, clusters);
         serviceInfoMap.put(serviceObj.getKey(), serviceObj);
         //用于并发控制
         updatingMap.put(serviceName, new Object());
         //调用Nacos服务器获取实例
         updateServiceNow(serviceName, clusters);
         updatingMap.remove(serviceName);

     }
     //控制并发请求
     else if (updatingMap.containsKey(serviceName)) {
         if (UPDATE_HOLD_INTERVAL > 0) {
             synchronized (serviceObj) {
                 try {
                     //当前线程等待5秒，上一个线程完成更新后会调用 serviceObj.notifyAll() 方法唤醒线程
                     serviceObj.wait(UPDATE_HOLD_INTERVAL);
                 } catch (InterruptedException e) {
                     NAMING_LOGGER
                         .error("[getServiceInfo] serviceName:" + serviceName + ", clusters:" + clusters, e);
                 }
             }
         }
     }
     //开启 updateTask 线程定时更新 serviceInfoMap 中的实例
     scheduleUpdateIfAbsent(serviceName, clusters);
     return serviceInfoMap.get(serviceObj.getKey());
 }
```

UpdateTask 线程
```java
@Override
public void run() {
    //pull执行间隔时间，默认为1秒
    long delayTime = DEFAULT_DELAY;

    try {
        //如果内存中没有 serviceName 对应的实例，就马上调用 updateService 方法进行从Nacos查询数据
        ServiceInfo serviceObj = serviceInfoMap.get(ServiceInfo.getKey(serviceName, clusters));
        if (serviceObj == null) {
            updateService(serviceName, clusters);
            return;
        }

        // 如果 serviceName 的数据已经通过服务端的 push 更新了，那么就只需要刷新下本地数据，否则从Nacos查询数据并更新数据
        // updateService 会做3件事情：
		// 1. 从Nacos拉取最新的数据
        // 2. 将数据更新到 serviceInfoMap 中
        // 3. 向外广播 InstancesChangeEvent 事件
        if (serviceObj.getLastRefTime() <= lastRefTime) {
            updateService(serviceName, clusters);
            serviceObj = serviceInfoMap.get(ServiceInfo.getKey(serviceName, clusters));
        } else {
            refreshOnly(serviceName, clusters);
        }
        lastRefTime = serviceObj.getLastRefTime();

        if (!notifier.isSubscribed(serviceName, clusters) && !futureMap
            .containsKey(ServiceInfo.getKey(serviceName, clusters))) {
            NAMING_LOGGER.info("update task is stopped, service:" + serviceName + ", clusters:" + clusters);
            return;
        }
        if (CollectionUtils.isEmpty(serviceObj.getHosts())) {
            incFailCount();
            return;
        }
        delayTime = serviceObj.getCacheMillis();
        resetFailCount();
    } catch (Throwable e) {
        incFailCount();
        NAMING_LOGGER.warn("[NA] failed to update serviceName: " + serviceName, e);
    } finally {

        // 开启下一次 pull
        executor.schedule(this, Math.min(delayTime << failCount, DEFAULT_DELAY * 60), TimeUnit.MILLISECONDS);
    }
}
```