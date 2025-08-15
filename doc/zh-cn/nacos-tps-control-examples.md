# Nacos TPS 控制实战示例

## 1. 基础使用示例

### 1.1 在 Controller 中使用 TPS 控制

```java
@RestController
@RequestMapping("/v1/config")
public class ConfigController {
    
    @PostMapping("/publish")
    @TpsControl(pointName = "ConfigController.publishConfig") 
    public Result<Boolean> publishConfig(
            @RequestParam("dataId") String dataId,
            @RequestParam("group") String group,
            @RequestParam("content") String content) {
        
        // 发布配置的具体逻辑
        return Result.success(configService.publishConfig(dataId, group, content));
    }
    
    @GetMapping("/get")
    @TpsControl(pointName = "ConfigController.getConfig")
    public Result<String> getConfig(
            @RequestParam("dataId") String dataId,
            @RequestParam("group") String group) {
        
        // 获取配置的具体逻辑  
        return Result.success(configService.getConfig(dataId, group));
    }
}
```

### 1.2 配置文件设置

```properties
# application.properties

# 启用 TPS 控制
nacos.core.control.tps.enabled=true

# 使用 nacos 控制插件
nacos.core.control.manager.type=nacos

# 本地规则存储目录
nacos.core.control.rule.local.basedir=/data/nacos/control

# 启用外部规则存储
nacos.core.control.rule.external.storage=mysql
```

### 1.3 TPS 规则配置

创建 TPS 控制规则文件：`/data/nacos/control/tps/ConfigController.publishConfig.json`

```json
{
  "pointName": "ConfigController.publishConfig",
  "pointRule": {
    "maxCount": 100,
    "period": 1,
    "monitorType": "intercept"
  }
}
```

## 2. 代码深度解析

### 2.1 HTTP 过滤器工作流程

```java
// NacosHttpTpsFilter.java 核心逻辑
@Override
public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain)
        throws IOException, ServletException {
    
    final HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequest;
    final HttpServletResponse response = (HttpServletResponse) servletResponse;
    
    // 1. 获取控制器方法
    Method method = controllerMethodsCache.getMethod(httpServletRequest);
    
    try {
        // 2. 检查是否需要 TPS 控制
        if (method != null && method.isAnnotationPresent(TpsControl.class)
                && TpsControlConfig.isTpsControlEnabled()) {
            
            // 3. 获取控制点信息
            TpsControl tpsControl = method.getAnnotation(TpsControl.class);
            String pointName = tpsControl.pointName();
            
            // 4. 初始化 TPS 控制管理器
            initTpsControlManager();
            
            // 5. 注册控制点
            tpsControlManager.registerTpsPoint(pointName);
            
            // 6. 构建检查请求
            TpsCheckRequest tpsCheckRequest = buildTpsCheckRequest(httpServletRequest, pointName);
            
            // 7. 执行 TPS 检查
            TpsCheckResponse tpsCheckResponse = tpsControlManager.check(tpsCheckRequest);
            
            // 8. 处理检查结果
            if (!tpsCheckResponse.isSuccess()) {
                // 限流触发，返回 503 错误
                generate503Response(httpServletRequest, response, 
                    tpsCheckResponse.getMessage(), null);
                return;
            }
        }
        
        // 9. 继续处理请求
        filterChain.doFilter(servletRequest, servletResponse);
        
    } catch (Exception e) {
        Loggers.TPS.warn("Error occurred while processing TPS control", e);
        filterChain.doFilter(servletRequest, servletResponse);
    }
}
```

### 2.2 TPS 屏障实现详解

```java
// SimpleCountRuleBarrier.java 核心限流逻辑
@Override
public TpsCheckResponse applyTps(BarrierCheckRequest barrierCheckRequest) {
    
    // 1. 判断监控类型
    if (MonitorType.INTERCEPT.getType().equals(getMonitorType())) {
        
        // 2. 获取最大允许计数
        long maxCount = getMaxCount();
        
        // 3. 尝试增加计数（原子操作）
        boolean accepted = rateCounter.tryAdd(
            barrierCheckRequest.getTimestamp(), 
            barrierCheckRequest.getCount(), 
            maxCount
        );
        
        // 4. 返回检查结果
        return accepted ? 
            new TpsCheckResponse(true, TpsResultCode.PASS_BY_POINT, "success") :
            new TpsCheckResponse(false, TpsResultCode.DENY_BY_POINT, 
                "tps over limit :" + maxCount);
                
    } else {
        // 仅监控模式，不拦截
        rateCounter.add(barrierCheckRequest.getTimestamp(), barrierCheckRequest.getCount());
        return new TpsCheckResponse(true, TpsResultCode.PASS_BY_POINT, "success");
    }
}
```

### 2.3 计数器实现原理

```java
// LocalSimpleCountRateCounter.java 滑动窗口计数
public class LocalSimpleCountRateCounter implements RateCounter {
    
    private final AtomicLong count = new AtomicLong(0);
    private volatile long lastResetTime = System.currentTimeMillis();
    private final long periodMs;
    
    @Override
    public boolean tryAdd(long timeStamp, int count, long maxCount) {
        // 1. 检查是否需要重置计数器
        checkAndResetIfNeed(timeStamp);
        
        // 2. 原子性地尝试增加计数
        long currentCount = this.count.get();
        if (currentCount + count <= maxCount) {
            long newCount = this.count.addAndGet(count);
            return newCount <= maxCount;
        }
        
        return false;
    }
    
    private void checkAndResetIfNeed(long timeStamp) {
        long timePassed = timeStamp - lastResetTime;
        if (timePassed >= periodMs) {
            synchronized (this) {
                if (timeStamp - lastResetTime >= periodMs) {
                    count.set(0);
                    lastResetTime = timeStamp;
                }
            }
        }
    }
}
```

## 3. 自定义扩展示例

### 3.1 自定义 TPS 控制管理器

```java
@Component
public class EnhancedTpsControlManager extends TpsControlManager {
    
    private final Map<String, TpsBarrier> points = new ConcurrentHashMap<>();
    private final Map<String, TpsControlRule> rules = new ConcurrentHashMap<>();
    private final RedisTemplate<String, String> redisTemplate;
    
    public EnhancedTpsControlManager(RedisTemplate<String, String> redisTemplate) {
        super();
        this.redisTemplate = redisTemplate;
    }
    
    @Override
    public void registerTpsPoint(String pointName) {
        if (!points.containsKey(pointName)) {
            // 创建增强的 TPS 屏障
            points.put(pointName, new DistributedTpsBarrier(pointName, redisTemplate));
            
            // 从 Redis 加载规则
            loadRuleFromRedis(pointName);
        }
    }
    
    @Override
    public TpsCheckResponse check(TpsCheckRequest tpsRequest) {
        String pointName = tpsRequest.getPointName();
        TpsBarrier barrier = points.get(pointName);
        
        if (barrier == null) {
            return new TpsCheckResponse(true, TpsResultCode.PASS_BY_POINT, "no barrier found");
        }
        
        return barrier.applyTps(tpsRequest);
    }
    
    @Override
    public void applyTpsRule(String pointName, TpsControlRule rule) {
        rules.put(pointName, rule);
        
        // 同步规则到 Redis
        saveRuleToRedis(pointName, rule);
        
        // 应用到本地屏障
        TpsBarrier barrier = points.get(pointName);
        if (barrier != null) {
            barrier.applyRule(rule);
        }
    }
    
    private void loadRuleFromRedis(String pointName) {
        String ruleJson = redisTemplate.opsForValue().get("tps:rule:" + pointName);
        if (StringUtils.isNotBlank(ruleJson)) {
            TpsControlRule rule = JsonUtils.parseObject(ruleJson, TpsControlRule.class);
            applyTpsRule(pointName, rule);
        }
    }
    
    private void saveRuleToRedis(String pointName, TpsControlRule rule) {
        String ruleJson = JsonUtils.toJSONString(rule);
        redisTemplate.opsForValue().set("tps:rule:" + pointName, ruleJson);
    }
    
    @Override
    public String getName() {
        return "enhanced";
    }
}
```

### 3.2 分布式 TPS 屏障

```java
public class DistributedTpsBarrier extends TpsBarrier {
    
    private final RedisTemplate<String, String> redisTemplate;
    private final String pointName;
    private final String redisKey;
    
    public DistributedTpsBarrier(String pointName, RedisTemplate<String, String> redisTemplate) {
        super(pointName);
        this.pointName = pointName;
        this.redisTemplate = redisTemplate;
        this.redisKey = "tps:count:" + pointName;
    }
    
    @Override
    public TpsCheckResponse applyTps(TpsCheckRequest tpsCheckRequest) {
        long currentSecond = tpsCheckRequest.getTimestamp() / 1000;
        String secondKey = redisKey + ":" + currentSecond;
        
        try {
            // 使用 Redis Lua 脚本保证原子性
            String luaScript = 
                "local current = redis.call('get', KEYS[1]) " +
                "if current == false then current = 0 else current = tonumber(current) end " +
                "if current + ARGV[1] <= ARGV[2] then " +
                "    redis.call('incrby', KEYS[1], ARGV[1]) " +
                "    redis.call('expire', KEYS[1], 2) " +
                "    return 1 " +
                "else " +
                "    return 0 " +
                "end";
            
            Long result = redisTemplate.execute(
                new DefaultRedisScript<>(luaScript, Long.class),
                Collections.singletonList(secondKey),
                String.valueOf(tpsCheckRequest.getCount()),
                String.valueOf(getMaxCount())
            );
            
            boolean accepted = result != null && result == 1;
            
            return accepted ?
                new TpsCheckResponse(true, TpsResultCode.PASS_BY_POINT, "success") :
                new TpsCheckResponse(false, TpsResultCode.DENY_BY_POINT, 
                    "distributed tps over limit");
                    
        } catch (Exception e) {
            Loggers.CONTROL.error("Error in distributed TPS check", e);
            // 降级到通过
            return new TpsCheckResponse(true, TpsResultCode.PASS_BY_POINT, "fallback");
        }
    }
}
```

### 3.3 动态规则管理

```java
@RestController
@RequestMapping("/tps/admin")
public class TpsAdminController {
    
    @Autowired
    private TpsControlManager tpsControlManager;
    
    @PostMapping("/rule")
    public Result<Void> updateTpsRule(@RequestBody TpsRuleUpdateRequest request) {
        
        // 验证规则
        validateRule(request);
        
        // 构建控制规则
        TpsControlRule rule = new TpsControlRule();
        rule.setPointName(request.getPointName());
        
        RuleDetail ruleDetail = new RuleDetail();
        ruleDetail.setMaxCount(request.getMaxCount());
        ruleDetail.setPeriod(request.getPeriod());
        ruleDetail.setMonitorType(request.getMonitorType());
        rule.setPointRule(ruleDetail);
        
        // 应用规则
        tpsControlManager.applyTpsRule(request.getPointName(), rule);
        
        // 触发规则变更事件
        ControlManagerCenter.getInstance()
            .reloadTpsControlRule(request.getPointName(), false);
        
        return Result.success();
    }
    
    @GetMapping("/rule/{pointName}")
    public Result<TpsControlRule> getTpsRule(@PathVariable String pointName) {
        Map<String, TpsControlRule> rules = tpsControlManager.getRules();
        TpsControlRule rule = rules.get(pointName);
        return Result.success(rule);
    }
    
    @GetMapping("/metrics/{pointName}")
    public Result<TpsMetrics> getTpsMetrics(@PathVariable String pointName) {
        Map<String, TpsBarrier> points = tpsControlManager.getPoints();
        TpsBarrier barrier = points.get(pointName);
        
        if (barrier != null) {
            TpsMetrics metrics = barrier.getMetrics(System.currentTimeMillis());
            return Result.success(metrics);
        }
        
        return Result.error("Point not found");
    }
}
```

## 4. 监控和告警

### 4.1 TPS 指标收集

```java
@Component
public class TpsMetricsCollector {
    
    private final MeterRegistry meterRegistry;
    private final TpsControlManager tpsControlManager;
    
    public TpsMetricsCollector(MeterRegistry meterRegistry, TpsControlManager tpsControlManager) {
        this.meterRegistry = meterRegistry;
        this.tpsControlManager = tpsControlManager;
    }
    
    @Scheduled(fixedRate = 10000) // 每10秒收集一次
    public void collectMetrics() {
        Map<String, TpsBarrier> points = tpsControlManager.getPoints();
        
        for (Map.Entry<String, TpsBarrier> entry : points.entrySet()) {
            String pointName = entry.getKey();
            TpsBarrier barrier = entry.getValue();
            
            TpsMetrics metrics = barrier.getMetrics(System.currentTimeMillis());
            if (metrics != null) {
                // 记录通过数
                meterRegistry.gauge("nacos.tps.pass.count", 
                    Tags.of("point", pointName), 
                    metrics.getCounter().getPassCount());
                
                // 记录拒绝数  
                meterRegistry.gauge("nacos.tps.denied.count",
                    Tags.of("point", pointName),
                    metrics.getCounter().getDeniedCount());
                
                // 计算拒绝率
                long total = metrics.getCounter().getPassCount() + metrics.getCounter().getDeniedCount();
                double deniedRate = total > 0 ? 
                    (double) metrics.getCounter().getDeniedCount() / total : 0;
                    
                meterRegistry.gauge("nacos.tps.denied.rate",
                    Tags.of("point", pointName),
                    deniedRate);
            }
        }
    }
}
```

### 4.2 告警配置

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: metrics, health, prometheus
  metrics:
    export:
      prometheus:
        enabled: true

# 告警规则 (Prometheus)
groups:
  - name: nacos-tps
    rules:
      - alert: NacosTpsHighDeniedRate
        expr: nacos_tps_denied_rate > 0.1
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Nacos TPS denied rate is high"
          description: "Point {{ $labels.point }} has denied rate {{ $value }}"
          
      - alert: NacosTpsHighLoad  
        expr: nacos_tps_pass_count > 1000
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Nacos TPS load is very high"
          description: "Point {{ $labels.point }} has {{ $value }} requests per period"
```

## 5. 测试示例

### 5.1 单元测试

```java
@SpringBootTest
class TpsControlTest {
    
    @Autowired
    private TpsControlManager tpsControlManager;
    
    @Test
    void testTpsControl() {
        String pointName = "test.point";
        
        // 1. 注册控制点
        tpsControlManager.registerTpsPoint(pointName);
        
        // 2. 设置规则：每秒最多10个请求
        TpsControlRule rule = createTpsRule(pointName, 10, 1, "intercept");
        tpsControlManager.applyTpsRule(pointName, rule);
        
        // 3. 测试正常情况
        for (int i = 0; i < 10; i++) {
            TpsCheckRequest request = createTpsRequest(pointName, 1);
            TpsCheckResponse response = tpsControlManager.check(request);
            Assertions.assertTrue(response.isSuccess());
        }
        
        // 4. 测试超限情况
        TpsCheckRequest request = createTpsRequest(pointName, 1);
        TpsCheckResponse response = tpsControlManager.check(request);
        Assertions.assertFalse(response.isSuccess());
        Assertions.assertEquals(TpsResultCode.DENY_BY_POINT, response.getCode());
    }
    
    private TpsControlRule createTpsRule(String pointName, long maxCount, int period, String monitorType) {
        TpsControlRule rule = new TpsControlRule();
        rule.setPointName(pointName);
        
        RuleDetail ruleDetail = new RuleDetail();
        ruleDetail.setMaxCount(maxCount);
        ruleDetail.setPeriod(period);
        ruleDetail.setMonitorType(monitorType);
        rule.setPointRule(ruleDetail);
        
        return rule;
    }
    
    private TpsCheckRequest createTpsRequest(String pointName, int count) {
        TpsCheckRequest request = new TpsCheckRequest();
        request.setPointName(pointName);
        request.setCount(count);
        request.setTimestamp(System.currentTimeMillis());
        return request;
    }
}
```

### 5.2 压力测试

```java
@Test
void stressTest() throws InterruptedException {
    String pointName = "stress.test";
    tpsControlManager.registerTpsPoint(pointName);
    
    // 设置每秒100个请求的限制
    TpsControlRule rule = createTpsRule(pointName, 100, 1, "intercept");
    tpsControlManager.applyTpsRule(pointName, rule);
    
    CountDownLatch latch = new CountDownLatch(200);
    AtomicInteger passCount = new AtomicInteger(0);
    AtomicInteger denyCount = new AtomicInteger(0);
    
    // 启动200个并发线程
    for (int i = 0; i < 200; i++) {
        CompletableFuture.runAsync(() -> {
            try {
                TpsCheckRequest request = createTpsRequest(pointName, 1);
                TpsCheckResponse response = tpsControlManager.check(request);
                
                if (response.isSuccess()) {
                    passCount.incrementAndGet();
                } else {
                    denyCount.incrementAndGet();
                }
            } finally {
                latch.countDown();
            }
        });
    }
    
    latch.await(10, TimeUnit.SECONDS);
    
    // 验证结果
    System.out.println("Pass: " + passCount.get() + ", Deny: " + denyCount.get());
    Assertions.assertTrue(passCount.get() <= 100);
    Assertions.assertTrue(denyCount.get() >= 100);
}
```

## 6. 生产环境建议

### 6.1 性能优化

1. **合理设置限流阈值**
   - 基于实际业务容量和响应时间设置
   - 考虑突发流量的缓冲空间

2. **使用异步处理**
   - 对于非关键路径，考虑异步处理被限流的请求
   - 实现请求排队和延迟处理机制

3. **缓存优化**
   - 对频繁访问的规则进行本地缓存
   - 减少规则查询的开销

### 6.2 故障处理

1. **降级策略**
   - 当 TPS 控制组件异常时，自动降级为通过模式
   - 记录异常日志但不影响正常业务流程

2. **监控告警**
   - 设置合理的告警阈值
   - 建立完善的告警通知机制

3. **快速恢复**
   - 提供管理接口快速调整限流规则
   - 支持热配置更新，无需重启服务

这个实战示例展示了 Nacos TPS 控制系统的完整使用方法，从基础配置到高级扩展，从单元测试到生产部署，为实际项目提供了全面的参考。