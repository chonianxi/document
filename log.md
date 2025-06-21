![image](https://github.com/user-attachments/assets/2ee481db-ae81-498d-896c-207355f77c7f)log配置


package com.sports;

import ch.qos.logback.classic.LoggerContext;
import com.sports.log.lark.LarkAppender;
import org.apache.rocketmq.spring.core.RocketMQTemplate;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.ApplicationRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.env.Environment;

@Configuration
public class LarkLogConfig {
    @Autowired
    private Environment environment;


    @Bean
    public ApplicationRunner larkAppenderRegister(RocketMQTemplate rocketMQTemplate) {
        return args -> {
            LoggerContext context = (LoggerContext) LoggerFactory.getILoggerFactory();

            LarkAppender appender = new LarkAppender();
            appender.setName("LARK");
            appender.setContext(context);
            LarkAppender.setRocketMQTemplate(rocketMQTemplate);
            LarkAppender.setAppName(environment.getProperty("spring.application.name", "UnknownApp"));
            LarkAppender.setActive(environment.getProperty("spring.profiles.active", "UnknownActive"));
            appender.start();

            context.getLogger("ROOT").addAppender(appender);

            System.out.println("LarkAppender registered successfully.");
        };
    }
}

package com.sports.log.lark;

import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.classic.spi.IThrowableProxy;
import ch.qos.logback.classic.spi.StackTraceElementProxy;
import ch.qos.logback.core.AppenderBase;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.rocketmq.spring.core.RocketMQTemplate;
import org.apache.skywalking.apm.toolkit.trace.TraceContext;
import org.springframework.messaging.support.MessageBuilder;

import java.util.Date;
import java.util.HashMap;
import java.util.Map;

public class LarkAppender  extends AppenderBase<ILoggingEvent> {

    private static RocketMQTemplate rocketMQTemplate;

    private static String appName;

    private static String active;

    private static volatile boolean ready = false;

    private final String topic = "log-error-topic";

    public static void setRocketMQTemplate(RocketMQTemplate template) {
        LarkAppender.rocketMQTemplate = template;
        ready = true;
    }

    public static void setAppName(String appName) {
        LarkAppender.appName = appName;
    }

    public static void setActive(String active) {
        LarkAppender.active = active;
    }

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void start() {
        super.start();
    }

    @Override
    protected void append(ILoggingEvent event) {
        if (!event.getLevel().isGreaterOrEqual(Level.ERROR)) return;

        if (!ready || rocketMQTemplate == null) {
            System.err.println("[LarkAppender] RocketMQTemplate not set yet. Drop this log.");
            return;
        }

        String traceId = TraceContext.traceId();
        if (traceId.isEmpty()) {
            traceId = event.getMDCPropertyMap().get("tid");
        }
        if (traceId == null || traceId.isEmpty()) {
            traceId = event.getMDCPropertyMap().get("traceId");
        }
        if (traceId == null || traceId.isEmpty()) {
            traceId = "-";
        }

        Map<String, Object> logMsg = new HashMap<>();
        logMsg.put("app", appName+"-"+active);
        logMsg.put("timestamp", new Date(event.getTimeStamp()).toString());
        logMsg.put("traceId", traceId);
        logMsg.put("logger", event.getLoggerName());
        logMsg.put("message", event.getFormattedMessage());

        IThrowableProxy throwableProxy = event.getThrowableProxy();
        if (throwableProxy != null) {
            logMsg.put("exception", throwableProxy.getClassName() + ": " + throwableProxy.getMessage());
            StackTraceElementProxy[] traces = throwableProxy.getStackTraceElementProxyArray();
            if (traces != null && traces.length > 0) {
                StringBuilder stack = new StringBuilder();
                for (int i = 0; i < Math.min(5, traces.length); i++) {
                    stack.append("    at ").append(traces[i].getSTEAsString()).append("\n");
                }
                logMsg.put("stack", stack.toString());
            }
        }

        try {
            String json = objectMapper.writeValueAsString(logMsg);
            rocketMQTemplate.send(topic, MessageBuilder.withPayload(json).build());
        } catch (Exception e) {
            System.err.println("[LarkAppender] Failed to send to RocketMQ: " + e.getMessage());
        }
    }
}



