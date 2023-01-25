# Java 日志切面模版

```java

import cn.yizhoucp.ms.core.base.ErrorCode;
import cn.yizhoucp.ms.core.base.Result;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.annotation.*;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.context.support.ResourceBundleMessageSource;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestAttributes;
import org.springframework.web.context.request.RequestContextHolder;

import javax.annotation.Resource;
import javax.servlet.http.HttpServletRequest;
import java.util.HashMap;
import java.util.Locale;
import java.util.Map;

/**
 * 日志
 */
@Aspect
@Component
@Slf4j
public class WebLogAspect {

    /** 返回字符串最大长度 */
    private static final Integer MAX_RESULT_STR_LENGTH = 2000;

    /** 国际化处理工具 */
    @Resource
    private ResourceBundleMessageSource resourceBundleMessageSource;

    @Pointcut("execution(public * cn.yizhoucp.ump.biz.project.web.rest.controller..*.*(..))")
    public void umpControllerLog() {
    }

    @Around("umpControllerLog()")
    public Object aroundHandle(ProceedingJoinPoint joinPoint) throws Throwable {
        long ts = System.currentTimeMillis();
        log.info("http request method:{}, param:{}", getMethodName(joinPoint), JSONObject.toJSONString(getParams(joinPoint)));
        Object proceed = internationalHandle(joinPoint.proceed());
        log.info("http request result:{}, time:{}ms", getResultStr(proceed), System.currentTimeMillis() - ts);
        return proceed;
    }

    private String getMethodName(ProceedingJoinPoint joinPoint) {
        return joinPoint.getTarget().getClass().getSimpleName() + "." + joinPoint.getSignature().getName();
    }

    private String getResultStr(Object proceed) {
        String result = "";
        if (proceed != null) {
            result = JSONObject.toJSONString(proceed);
            if (result.length() > MAX_RESULT_STR_LENGTH) {
                result = result.substring(0, MAX_RESULT_STR_LENGTH);
            }
        }
        return result;
    }


    private Map<String, String> getParams(JoinPoint joinPoint) {
        Signature signature = joinPoint.getSignature();
        MethodSignature methodSignature = (MethodSignature) signature;
        String[] parameterNames = methodSignature.getParameterNames();
        Object[] paramValues = joinPoint.getArgs();
        Map<String, String> params = new HashMap<>();
        if (parameterNames.length > 0) {
            for (int i = 0; i < parameterNames.length; i++) {
                String paramName = "";
                String paramValue = "";
                if (parameterNames[i] != null) {
                    paramName = parameterNames[i];
                }
                if (paramValues[i] != null) {
                    paramValue = paramValues[i].toString();
                } else {
                    paramValue = "null";
                }
                params.put(paramName, paramValue);
            }
        }
        return params;
    }

    public Object internationalHandle(Object result) {
        if (result instanceof Result) {
            try {
                RequestAttributes requestAttributes = RequestContextHolder.getRequestAttributes();
                HttpServletRequest request = (HttpServletRequest) requestAttributes.resolveReference(RequestAttributes.REFERENCE_REQUEST);
                String language = request.getHeader("inner-user-language-in-header-gateway");
                if (language == null) {
                    language = "zh";
                }
                String country = request.getHeader("inner-user-country-in-header-gateway");
                if (country == null) {
                    // 因为端上 用户国家的信息不好拿 临时进行处理
                    if ("en".equals(language)) {
                        country = "US";
                    } else if ("in".equals(language)) {
                        country = "ID";
                    } else {
                        country = "CN";
                    }
                }
                log.debug("afterInternationController language -> {} country -> {}", language, country);
                String code = ((Result) result).getCode();
                if (code == null || ErrorCode.SUCCESS.getCode().equals(code)) {
                    return result;
                }
                String message = ((Result) result).getMessage();
                if (StringUtils.isEmpty(message) || ErrorCode.SUCCESS.getCode().equals(message)) {
                    return result;
                }
                String description = resourceBundleMessageSource.getMessage(message, null, new Locale(language, country));
                ((Result) result).setMessage(description);
                log.debug("afterInternationController controller aop log result 返回 {}", JSON.toJSONString(result));
            } catch (Exception e) {
                log.info("afterInternationController get international wrong result -> {}", result);
            }
        }
        return result;
    }

}


```