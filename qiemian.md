@Around("@annotation(printMethodExecTimeout)")
    public Object printMethodExecTimeout(ProceedingJoinPoint joinPoint, PrintMethodExecTimeout printMethodExecTimeout) throws Throwable {
        long threshold = printMethodExecTimeout.threshold();
        long start = System.currentTimeMillis();
        Object proceed = joinPoint.proceed();
        long executionTime = System.currentTimeMillis() - start;
        if (executionTime> threshold){
            // 获取方法签名
            MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
            // 方法名
            String methodName = methodSignature.getName();
            // 类名
            String className = joinPoint.getTarget().getClass().getName();
            log.warn("Class: {}，方法执行超时被记录了-Method: {},executed in {}",className, methodName,  executionTime);
            /* 参数值数组 先不打印，后续需要再打印
            Object[] args = joinPoint.getArgs();
            StringBuilder params = new StringBuilder();
            for (int i = 0; i < args.length; i++) {
                params.append("arg[").append(i).append("=").append(args[i]).append(", ");
            }*/

        }

        return proceed;
    }
