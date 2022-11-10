# spring 扩展点在业务中的使用

## RocketMq 消费者组 修改
### 问题背景
开发环境有多套，我们内部称之为动态环境，之前一个消费者写好之后，consumer group都是死的，不会和具体的分支名关联起来
在我自己的开发环境 中 我发一条消息 需要被我自己的环境消费 而不是 其他同事的环境消费掉。
公司内部ROcketMQ使用的是集群模式，一个消息只能被一个消费者组消费，为了解决问题，需要动态的改造消费者组名，不能通过应变码的形式。
因次可以使用spring 的后置处理器，对我定义后的bean进行加工处理

```java
package cn.yizhoucp.ms.core.base.env;

import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.apache.rocketmq.spring.annotation.RocketMQMessageListener;
import org.apache.rocketmq.spring.core.RocketMQListener;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor;
import org.springframework.core.env.Environment;

import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.Map;

/**
 * 描述：
 * 修改动态环境 mq 消费者组名
 *
 */
@Slf4j
public class DynamicRocketMQPostProcessor implements InstantiationAwareBeanPostProcessor {

    @Value("${spring.profiles.active}")
    String envProfile;

    @Autowired
    Environment env;

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if (!DynamicEnvManager.DEV_ENV.equals(envProfile) && !DynamicEnvManager.TEST_ENV.equals(envProfile)) {
            return bean;
        }
        if (bean instanceof RocketMQListener) {
            try {
                log.info("postProcessProperties bean -> {} beanName -> {}", bean, beanName);
                RocketMQListener<?> rocketMQListener = (RocketMQListener<?>) bean;
                Class<? extends RocketMQListener> rocketmqListener = rocketMQListener.getClass();

                RocketMQMessageListener annotation = rocketmqListener.getAnnotation(RocketMQMessageListener.class);

                //获取 这个代理实例所持有的 InvocationHandler
                InvocationHandler invocationHandler = Proxy.getInvocationHandler(annotation);

                // 获取 AnnotationInvocationHandler 的 memberValues 字段
                Field declaredField = invocationHandler.getClass().getDeclaredField("memberValues");

                // 因为这个字段事 private final 修饰，所以要打开权限
                declaredField.setAccessible(true);

                // 获取 memberValues
                Map memberValues = (Map) declaredField.get(invocationHandler);

                // 修改 value 属性值
                String oldConsumer = annotation.consumerGroup();

                String property = env.getProperty(DynamicEnvManager.GLOBAL_ENV_NAME);
                if (StringUtils.isBlank(property)) {
                    property = "";
                    log.error("postProcessBeforeInitialization env error property -> {}", property);
                }
                memberValues.put("consumerGroup", oldConsumer + property);

                log.info("postProcessBeforeInitialization annotation -> {}", annotation);
            } catch (Exception e) {
                log.error("postProcessBeforeInitialization consumerGroup replace error : ", e);
            }
        }
        return bean;
    }

}

```

## 在事务提交之后 发送mq
### 问题背景
先看一段代码
```java
@Tranicational
public void doBiz(){
    doSomething();
    sendMQ();
    doOthers();
}

```
通过这一段代码，可以发现问题：事务提交失败后，消息不能进行回滚
### 解决方案 
可以机遇Spring 事务的扩展点 实现一个工具类 在 事务成功提交之后 再执行额外的业务逻辑

```java
package cn.yizhoucp.ms.core.base.env;

import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.support.TransactionSynchronization;
import org.springframework.transaction.support.TransactionSynchronizationManager;

/**
 * @Author 六月
 * @Date 2022/11/10 23:58
 * @Version 1.0
 */
public class TxUtils {

    public static void doAfterTransaction(DoTransactionComplete doTransactionComplete) {
        if (TransactionSynchronizationManager.isActualTransactionActive()) {
            TransactionSynchronizationManager.registerSynchronization(doTransactionComplete);
        }
    }

    @Transactional
    public void doTx() {
        // start tx

        TxUtils.doAfterTransaction(new DoTransactionComplete(() -> {

        }));

        // end tx
    }

}

class DoTransactionComplete implements TransactionSynchronization {

    private Runnable run;

    public DoTransactionComplete(Runnable run) {
        this.run = run;
    }

    /**
     * Invoked after transaction commit/rollback.
     *
     * @param status
     */
    @Override
    public void afterCompletion(int status) {
        if (status == TransactionSynchronization.STATUS_COMMITTED) {
            // 执行事务提交后的业务逻辑
            this.run.run();
        }
    }
}

```