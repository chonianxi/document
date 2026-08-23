# bitgame-sign-spring-boot-starter 完整技术文档

> 本文档面向 AI 和开发者，详细描述组件的每一个模块、类、方法、数据流和设计决策。

---

## 1. 项目概述

### 1a. 定位

`bitgame-sign-spring-boot-starter` 是一个 **数据防篡改自动签名组件**，基于 Spring Boot Starter 机制，以 **零代码侵入** 的方式为 MyBatis Plus 应用的数据库表行数据自动计算并填充 SHA-256 签名。

### 1b. 核心能力

| 能力 | 说明 |
|---|---|
| **自动签名** | INSERT/UPDATE 时自动计算签名并写入 `data_hash` 字段 |
| **自动时间戳填充** | INSERT 时填充 `created_at`，UPDATE 时刷新 `updated_at`（字段名可配置） |
| **真签名 vs 假签名** | 在配置中的表生成真签名（SHA-256+盐值）；不在配置但有 hash 字段的表生成假签名（随机 64 位 hex） |
| **多表支持** | YAML 配置多张表，每表独立签名字段和盐值 |
| **AWS Secrets Manager** | 启动时从 AWS 获取盐值，支持全局和表级独立 Secret |
| **验签工具** | `SignVerifier` 支持实体对象验签、Map 验签、字段列表验签三种模式 |
| **异常隔离** | 任何签名异常不影响业务执行，仅记录日志 |
| **配置驱动 + 注解驱动** | YAML 优先于注解，两种方式可共存 |

### 1c. 技术栈

| 技术 | 版本 | 用途 |
|---|---|---|
| Java | 1.8 | 运行环境 |
| Spring Boot | 2.5.15 | 自动装配框架 |
| MyBatis Plus | 3.5.3.1 | ORM 框架（provided 依赖） |
| AWS SDK v2 | 2.20.162 | Secrets Manager 盐值获取 |
| Jackson | 2.13.5 | JSON 解析（provided 依赖） |
| SLF4J | 1.7.36 | 日志接口（provided 依赖） |

### 1d. 依赖关系

```
业务项目
  └── bitgame-sign-spring-boot-starter (jar)
        ├── spring-boot-starter (provided)
        ├── spring-boot-autoconfigure (provided)
        ├── spring-boot-configuration-processor (optional, 编译期元数据生成)
        ├── mybatis-plus-boot-starter (provided)
        ├── slf4j-api (provided)
        ├── jackson-databind (provided)
        └── aws-secretsmanager (compile, 传递给业务方)
```

> AWS SDK 是 compile scope，会传递给业务方。其余依赖为 provided，由业务项目提供。
> `spring-boot-configuration-processor` 为 optional，仅在编译期生成配置元数据（IDE 自动补全提示），运行时不需要。

---

## 2. 项目结构

```
src/main/java/com/bitgame/sign/
├── annotation/
│   ├── SignTable.java          # 类级注解，标注实体为签名表
│   └── SignField.java          # 字段级注解，标注字段参与签名
├── config/
│   ├── SignAutoConfiguration.java   # Spring Boot 自动装配入口
│   ├── SignProperties.java          # 配置属性绑定 (bitgame.sign.*)
│   ├── SignTableConfig.java         # 单表签名配置 POJO
│   ├── SignFieldConfig.java         # 签名字段配置 POJO
│   ├── SignTableResolver.java       # 配置解析器（YAML > 注解，带缓存）
│   ├── AwsSaltInitializer.java      # AWS Secrets Manager 盐值初始化
│   └── WarnPojo.java                # 告警日志 JSON POJO
├── handler/
│   └── AutoSignHandler.java         # MyBatis Plus MetaObjectHandler 实现
├── interceptor/
│   └── SignMybatisInterceptor.java  # MyBatis 拦截器（兜底签名）
└── util/
    ├── SignUtils.java               # SHA-256、hex、脱敏、时间恒定比较
    └── SignVerifier.java            # 验签工具类（3 种验签模式）

src/main/resources/
└── META-INF/
    └── spring.factories             # 自动装配注册文件
```

---

## 3. 自动装配机制

### 3a. spring.factories

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.bitgame.sign.config.SignAutoConfiguration
```

Spring Boot 启动时扫描 `spring.factories`，自动注册 `SignAutoConfiguration`。

### 3b. SignAutoConfiguration

**文件**: `config/SignAutoConfiguration.java`

**装配条件**:
- `@ConditionalOnClass("com.baomidou.mybatisplus.core.handlers.MetaObjectHandler")` — classpath 中存在 MyBatis Plus
- `@ConditionalOnProperty(prefix = "bitgame.sign", name = "enabled", havingValue = "true", matchIfMissing = true)` — 默认启用
- `@EnableConfigurationProperties(SignProperties.class)` — 激活 `@ConfigurationProperties` 绑定，将 `bitgame.sign.*` 配置注入 `SignProperties`

**注册的 Bean**:

| Bean | 类型 | 条件 | 作用 | 启动日志 |
|---|---|---|---|---|
| `SignTableResolver` | 普通 Bean | 无额外条件 | 解析表签名配置（YAML > 注解） | 无 |
| `AutoSignHandler` | `MetaObjectHandler` | `@ConditionalOnMissingBean(MetaObjectHandler.class)` | MyBatis Plus 自动填充签名 | `INFO: 自动签名组件已启用, hashField={}, 配置表数={}, 表级AWS={}` |
| `SignMybatisInterceptor` | MyBatis `Interceptor` | 无额外条件 | 拦截器兜底签名 | `INFO: MyBatis 拦截器已注册` |
| `AwsSaltInitializer` | `InitializingBean` | 无额外条件 | 启动时从 AWS 获取盐值 | 无（初始化后按结果输出，见 6d 节） |

> **注意**: 如果业务方已有自定义 `MetaObjectHandler`，`AutoSignHandler` 不会注册。此时 `SignMybatisInterceptor` 作为兜底仍然生效。
>
> **启动日志中的表级 AWS 统计**: `autoSignHandler` 创建时遍历 `properties.getTables()` 统计 `hasAwsConfig()` 的表数，仅统计配置了 `secret-name` 的表，不代表 AWS 获取成功。

---

## 4. 配置体系

### 4a. SignProperties

**文件**: `config/SignProperties.java`
**前缀**: `bitgame.sign`

| 属性 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `enabled` | `boolean` | `true` | 全局开关 |
| `salt` | `String` | `""` | 全局签名盐值 |
| `hash-field-name` | `String` | `"data_hash"` | 签名存储列名（所有表统一） |
| `timestamp-fields` | `List<String>` | `["created_at", "updated_at"]` | 自动填充的时间戳列名列表 |
| `debug` | `boolean` | `false` | debug 日志开关（假签名、Resolver 日志受此控制） |
| `tables` | `Map<String, SignTableConfig>` | `{}` | 签名表配置，key = 表名 |
| `aws` | `AwsSecretConfig` | `new AwsSecretConfig()` | AWS Secrets Manager 配置 |

### 4b. AwsSecretConfig（内部类）

| 属性 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `secret-name` | `String` | `null` | AWS Secret 名称，非空时启用 |
| `secret-key` | `String` | `"WITHDRAW_SIGN_SALT"` | Secret JSON 中的 key |
| `region` | `String` | `null` | AWS Region，null 用 SDK 默认 |

**`isConfigured()`**: `secretName != null && !secretName.isEmpty()`

### 4c. SignTableConfig

**文件**: `config/SignTableConfig.java`

| 属性 | 类型 | 说明 |
|---|---|---|
| `sign-fields` | `List<SignFieldConfig>` | 参与签名的字段列表 |
| `insert-timestamp-field` | `String` | INSERT 时间戳列名 |
| `update-timestamp-field` | `String` | UPDATE 时间戳列名 |
| `salt` | `String` | 表级盐值（运行时由 AwsSaltInitializer 填充） |
| `secret-name` | `String` | 表级 AWS Secret 名称 |
| `secret-key` | `String` | 表级 Secret JSON key（类中无默认值，`AwsSaltInitializer` 获取时若为空则回退 `"WITHDRAW_SIGN_SALT"`） |
| `region` | `String` | 表级 AWS Region |

**关键方法**:
- `getEffectiveSalt(String globalSalt)`: 表级 salt 非 null 且非空则用表级，否则回退全局（`salt != null && !salt.isEmpty() ? salt : globalSalt`）
- `hasAwsConfig()`: `secretName != null && !secretName.isEmpty()`

> **secretKey 默认值差异**: 全局 `AwsSecretConfig.secretKey` 在类中默认 `"WITHDRAW_SIGN_SALT"`；表级 `SignTableConfig.secretKey` 在类中默认 `null`，但 `AwsSaltInitializer.initTableSalts()` 获取时若为空会回退到 `"WITHDRAW_SIGN_SALT"`。两者最终行为一致。

### 4d. SignFieldConfig

**文件**: `config/SignFieldConfig.java`

| 属性 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `name` | `String` | `null` | 数据库列名（下划线格式，如 `order_no`） |
| `order` | `int` | `0` | 签名串中的拼接顺序（升序），如 1, 2, 3。未指定时为 0，排在最前 |

> **YAML 配置方式**: `sign-fields` 列表中每项为 `{ name: 列名, order: 序号 }`。注解驱动时由 `SignTableResolver.doResolveFromAnnotation()` 自动构建，Java 属性名通过 `javaFieldToColumnName()` 转为下划线列名。

### 4e. YAML 配置示例

```yaml
bitgame:
  sign:
    enabled: true
    salt: ${WITHDRAW_SIGN_SALT:default_dev_salt}
    hash-field-name: data_hash
    debug: true
    timestamp-fields:
      - create_time
      - last_update_time
    aws:
      secret-name: prod/bitgame/sign-salt
      secret-key: WITHDRAW_SIGN_SALT
      region: ap-southeast-1
    tables:
      fund_withdrawl_order:
        sign-fields:
          - { name: order_no, order: 1 }
          - { name: user_id, order: 2 }
          - { name: amount, order: 3 }
        insert-timestamp-field: create_time
        update-timestamp-field: last_update_time
        secret-name: prod/bitgame/withdraw-salt
        secret-key: WITHDRAW_SIGN_SALT
        region: ap-southeast-1
```

---

## 5. 注解体系

### 5a. @SignTable

**文件**: `annotation/SignTable.java`
**目标**: `ElementType.TYPE`（类级别）
**保留**: `RUNTIME`
**`@Documented`**: 是

| 属性 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `insertTimestampField` | `String` | `""` | INSERT 时间戳列名（下划线格式） |
| `updateTimestampField` | `String` | `""` | UPDATE 时间戳列名（下划线格式） |

**优先级**: YAML 配置 > 注解配置。如果表在 YAML `tables` 中有配置，注解被忽略。

### 5b. @SignField

**文件**: `annotation/SignField.java`
**目标**: `ElementType.FIELD`（字段级别）
**保留**: `RUNTIME`
**`@Documented`**: 是

| 属性 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `order` | `int` | **必填**（无默认值） | 签名串中的拼接顺序（升序），一旦确定不可更改 |

> **注意**: `@SignField` 的 `order` 属性无默认值，使用注解时必须显式指定，否则编译报错。

### 5c. 注解使用示例

```java
@SignTable(insertTimestampField = "create_time", updateTimestampField = "last_update_time")
@TableName("fund_withdrawl_order")
public class FundWithdrawlOrder {

    @SignField(order = 1)
    private String orderNo;

    @SignField(order = 2)
    private Long userId;

    @SignField(order = 3)
    private BigDecimal amount;

    @TableField("data_hash")
    private String dataHash;

    // getter / setter ...
}
```

> 注解中的字段名使用 Java 属性名（驼峰），`SignTableResolver` 会自动转为下划线列名。

---

## 6. 核心类详解

### 6a. SignTableResolver — 配置解析器

**文件**: `config/SignTableResolver.java`

**职责**: 统一解析 YAML 配置和注解配置，返回 `SignTableConfig`。

**解析优先级**:
1. YAML 配置（`properties.getTables().get(tableName)`）— 仅当 `signFields != null && !isEmpty` 时才返回
2. 注解配置（`@SignTable` + `@SignField`）
3. 都没有 → 返回 `null`（假签名或跳过）

> **YAML 空配置回退**: 如果 YAML 中表存在但 `signFields` 为 null 或空列表，不会返回该 YAML 配置，而是继续尝试注解解析。这确保了配置不完整时不会误判为"已配置"。

**缓存机制**:
- `annotationCache`: `ConcurrentHashMap<Class<?>, SignTableConfig>` — 注解解析正缓存
- `noAnnotationCache`: `ConcurrentHashMap<Class<?>, Boolean>` — 负缓存（无注解的类快速跳过）

> **`putIfAbsent` + `get` 模式**: `resolveFromAnnotation` 在解析成功后执行 `putIfAbsent` + `get`（而非直接返回本地变量），确保并发时返回缓存中的首个值，保证所有线程获得同一实例。负缓存同样使用 `putIfAbsent`。

**核心方法**:

| 方法 | 说明 |
|---|---|
| `resolve(String tableName, Class<?> clazz)` | 按优先级解析配置，YAML > 注解 > null。`clazz == null` 时仅查 YAML，不解析注解 |
| `resolve(String tableName)` | 仅查 YAML（不传 Class），等价于 `resolve(tableName, null)` |
| `resolveFromAnnotation(Class<?>)` | 从注解解析（带缓存）。先查 `noAnnotationCache` 负缓存，再查 `annotationCache` 正缓存，最后反射解析 |
| `doResolveFromAnnotation(Class<?>)` | 实际反射解析注解（无缓存，由 `resolveFromAnnotation` 调用） |
| `javaFieldToColumnName(String)` | 驼峰转下划线（`orderNo` → `order_no`） |

**注解解析逻辑** (`doResolveFromAnnotation`):
1. 检查 `@SignTable` 注解是否存在，不存在返回 null
2. 遍历类及父类（`current != null && current != Object.class`）的所有 `getDeclaredFields()`，收集带 `@SignField` 的字段
3. 将 Java 属性名通过 `javaFieldToColumnName()` 转为数据库列名（驼峰转下划线）
4. 如果没有 `@SignField` 字段，记录 WARN 日志并返回 null
5. 构建 `SignTableConfig`，设置 `signFields`（`Collections.unmodifiableList` 不可变列表）、时间戳字段
6. 首次解析时如 `debug=true` 输出 INFO 日志（class, signFields 数, insertTs, updateTs）

**`resolve(String tableName)` 方法**: 仅查 YAML 配置，不传 Class，不解析注解。适用于无法获取实体类的场景。

**`javaFieldToColumnName(String)` 方法**: 驼峰转下划线，`orderNo` → `order_no`，`GlWithdraw` → `gl_withdraw`。遇到大写字母且非首字符时，前面插入 `_` 再转小写。

### 6b. AutoSignHandler — MyBatis Plus 自动填充

**文件**: `handler/AutoSignHandler.java`
**实现**: `MetaObjectHandler`

**触发时机**: MyBatis Plus 的 `insert()` / `update()` / `saveBatch()` 等方法调用时。

**核心流程** (`doFill`):

```
insertFill / updateFill (MetaObjectHandler 接口方法)
  └── try { doFill(metaObject, isUpdate) }
      catch (Throwable) { 记录日志，不影响业务 }

doFill:
  1. checkSaltOnce() — 首次使用时检查盐值是否为空，空则 WARN
  2. 检查实体是否有 hashFieldName 对应的属性，没有则跳过
  3. fillTimestamps() — 自动填充时间戳
  4. resolveTableName() — 获取表名
  5. resolver.resolve(tableName, clazz) — 获取签名配置
  6. tableConfig != null && signFields != null && !isEmpty → computeRealHash() → 真签名
     否则 → generateFakeHash() → 假签名
     (UPDATE 时检测签名字段全空 → WARN)
  7. metaObject.setValue(hashProp, hash) — 写入签名字段
```

**时间戳填充逻辑** (`fillTimestamps`):

| 字段类型 | INSERT | UPDATE |
|---|---|---|
| 含 "create" 的字段 | null 时填充当前时间 | 不修改 |
| 含 "update" 的字段 | null 时填充当前时间 | 总是刷新为当前时间 |
| 其他时间字段 | null 时填充当前时间 | 不修改 |

> 时间戳类型兼容: `LocalDateTime` / `Date` / `Long` 或 `long`(epoch millis) / `String`(yyyy-MM-dd HH:mm:ss)

**真签名计算** (`computeRealHash`):

```
1. 从 sortedFieldsCache 获取排序后的签名字段（带缓存）
   - 缓存未命中: 复制 signFields → 按 order 排序 → unmodifiableList → putIfAbsent + get（保证并发时返回同一实例）
2. 按顺序拼接字段值: field1|field2|field3|...
   - 每个字段: toCamelCase(列名) → hasProp 检查 → metaObject.getValue 取值 → formatValue
3. 追加时间戳: |create_time|last_update_time
4. 追加盐值（表级优先，回退全局）: salt
5. SHA-256 → 64 位 hex
```

**签名串格式**:
```
order_no|user_id|amount|create_time|last_update_timesalt
                                                    ^ 无分隔符
```

**假签名** (`generateFakeHash`):
- 使用 `ThreadLocalRandom` 生成 64 位随机 hex
- 格式与真 SHA-256 完全一致
- 防止攻击者通过 hash 格式判断哪些表有真签名

**缓存**:
- `sortedFieldsCache`: `ConcurrentHashMap<String, List<SignFieldConfig>>` — 排序后字段缓存
- `camelCaseCache`: `ConcurrentHashMap<String, String>` — 下划线转驼峰缓存

**工具方法**:

| 方法 | 说明 |
|---|---|
| `checkSaltOnce()` | 首次调用时检查 `properties.getSalt()` 是否为空，空则 WARN（`WarnPojo`）。`volatile saltChecked` 保证可见性，并发时可能输出多条 WARN（可接受，仅日志冗余） |
| `resolveTableName(MetaObject)` | 优先读 `@TableName` 注解值，无注解则 `camelToSnake(class.getSimpleName())` |
| `hasProp(MetaObject, prop)` | 检查 `metaObject.hasSetter(prop) && metaObject.hasGetter(prop)`，两者都满足才返回 true |
| `getFieldType(MetaObject, prop)` | `metaObject.getGetterType(prop)`，异常返回 null |
| `toCamelCase(snake)` | 下划线转驼峰（`created_at` → `createdAt`），带缓存 |
| `camelToSnake(camel)` | 驼峰转下划线（`FundWithdraw` → `fund_withdraw`） |
| `formatValue(val)` | 值格式化（见下表） |
| `appendTimestamp(sb, metaObject, tsField)` | tsField 非空时，`toCamelCase` 转驼峰，`hasProp` 检查属性存在后取值，追加 `|` + `formatValue`。属性不存在时取 null → `formatValue(null)` → `""` |
| `setTimestampValue(metaObject, prop, now)` | 根据字段类型设置时间值（见下方） |

**formatValue 类型处理顺序**（必须按此顺序，`Timestamp extends Date`）:

| 检查顺序 | 类型 | 格式化结果 |
|---|---|---|
| 1 | `null` | `""` |
| 2 | `LocalDateTime` | `yyyy-MM-dd HH:mm:ss` |
| 3 | `java.sql.Timestamp` | `toLocalDateTime().format(FMT)` |
| 4 | `Date` | `new Timestamp(date.getTime()).toLocalDateTime().format(FMT)` |
| 5 | `BigDecimal` | `toPlainString()`（不使用科学计数法） |
| 6 | 其他 | `String.valueOf(val)` |

> **重要**: `Timestamp` 继承自 `Date`，必须先检查 `Timestamp` 再检查 `Date`，否则 `Timestamp` 会被当作 `Date` 处理，虽然结果相同但多一次转换开销。

**setTimestampValue 类型兼容**: `LocalDateTime` / `Date`（写入 `java.sql.Timestamp`）/ `Long` 或 `long`（epoch millis，使用系统时区转换）/ `String`（`yyyy-MM-dd HH:mm:ss`）。`getFieldType` 返回 null 或其他类型时不写入，避免类型不匹配污染字段。

**异常处理**:
- 外层 `catch (Throwable)` 包裹整个 `doFill`
- 内层 `try-catch` 包裹日志调用，防止日志本身异常穿透

**日志策略**:
- 真签名: **始终输出 INFO**（`table, op, type=REAL, signFields, hash前8位`）
- 假签名: 仅 `debug=true` 时输出 INFO
- 盐值未配置: 首次 WARN（`WarnPojo` JSON 格式）
- 签名字段全空（UPDATE）: WARN（`WarnPojo` JSON 格式）

### 6c. SignMybatisInterceptor — MyBatis 拦截器（兜底）

**文件**: `interceptor/SignMybatisInterceptor.java`
**实现**: MyBatis `Interceptor`
**拦截**: `Executor.update(MappedStatement, Object)`

**为什么需要拦截器**:
`MetaObjectHandler` 仅在 MyBatis Plus 的 `insert`/`update` 方法中触发。如果业务使用了自定义 SQL 或 Wrapper update，`MetaObjectHandler` 不一定被调用。此拦截器作为兜底。

**核心流程** (`intercept`):

```
intercept(Invocation):
  1. 检查 enabled，未启用 → 直接 proceed()
  2. 获取 MappedStatement，判断 INSERT/UPDATE
  3. processParameter(parameter, isUpdate) — 先计算签名并写入实体
  4. invocation.proceed() — 后执行 SQL（签名在 SQL 执行前完成）
  注: try-catch 包裹步骤 2-3，异常不影响 proceed()
```

**参数处理** (`processParameter`):

| 参数类型 | 处理 |
|---|---|
| `Map` | 检查 key `et`（`isUserEntity` 检查后处理）、`param1`（`isUserEntity` + `param1 != entity` 去重）、`list`（`Collection` 类型时遍历，每个元素 `isUserEntity` 检查） |
| `Collection` | 遍历每个元素，`isUserEntity` 检查后处理 |
| 其他实体 | `isUserEntity` 检查后直接处理 |

> **Map 参数去重**: MyBatis Plus 可能将同一实体同时放入 `et` 和 `param1`，`processParameter` 通过 `param1 != entity` 引用比较避免对同一实体重复签名。
>
> **`isUserEntity` 过滤**: 所有参数（`et`、`param1`、`list` 元素、`Collection` 元素、直接实体）在 `fillEntitySafe` 前均经过 `isUserEntity` 检查，排除 `Map`、`Collection`、`String`、`Number`、`Boolean`、数组、枚举、MyBatis Plus Wrapper 类型（`com.baomidou.mybatisplus.core.conditions` 包前缀）。

**填充逻辑** (`fillEntity`):

```
1. 负缓存检查: skipClasses 中有此类 → 直接返回
2. 获取类字段映射（getFields，带缓存，最多 512 个类）
3. 无 hash 字段 → 加入负缓存，返回
4. hash 字段已有 64 位值 → MetaObjectHandler 已处理，跳过
5. fillTimestamps() — 填充时间戳
6. resolveTableName() + resolver.resolve() — 获取配置
7. tableConfig != null && signFields != null && !isEmpty → computeRealHash() → 真签名
   否则 → generateFakeHash() → 假签名
   (UPDATE 时检测签名字段全空 → WARN)
8. setFieldValue(entity, hashField, hash) — 写入
```

**与 AutoSignHandler 的关系**:
- `AutoSignHandler` 先执行（MetaObjectHandler 在拦截器之前）
- 拦截器检查 hash 是否已有 64 位值，有则跳过（避免重复签名）
- 如果 `AutoSignHandler` 未注册（业务方有自定义 Handler），拦截器独立完成签名

**缓存**:
- `fieldCache`: `ConcurrentHashMap<Class<?>, Map<String, Field>>` — 类字段反射缓存（上限 512）
- `sortedFieldsCache`: 排序后字段缓存（`putIfAbsent` + `get` 保证并发一致性，与 `AutoSignHandler` 相同模式）
- `camelCaseCache`: 驼峰转换缓存
- `skipClasses`: 负缓存（无 hash 字段的类）

**`getFields(Class<?>)` 方法详解**:
1. 检查 `fieldCache` 是否已有此类缓存，有则直接返回
2. 检查 `fieldCache.size() >= 512`，超限则 `clear()` 清空重建 + WARN 日志
3. 遍历类及父类（`current != null && current != Object.class`）的 `getDeclaredFields()`
4. 对每个字段 `f.setAccessible(true)` 后放入 `Map<fieldName, Field>`
5. `putIfAbsent` 写入缓存，返回结果（可能返回其他线程先写入的值）

**`isUserEntity(Object)` 排除规则**:

| 排除类型 | 原因 |
|---|---|
| `Map` | MyBatis 参数包装，非实体 |
| `Collection` | 批量参数集合，非实体 |
| `String` / `Number` / `Boolean` | 基本类型包装 |
| 数组 | 非实体 |
| 枚举 | 非实体 |
| `com.baomidou.mybatisplus.core.conditions.*` | MyBatis Plus Wrapper 类型 |

**MyBatis Interceptor 生命周期方法**:

| 方法 | 实现 |
|---|---|
| `plugin(Object target)` | `Plugin.wrap(target, this)` — 标准 MyBatis 插件包装 |
| `setProperties(Properties)` | no-op — 无额外属性配置 |

**工具方法**:

| 方法 | 说明 |
|---|---|
| `resolveTableName(Class<?>)` | 优先读 `@TableName` 注解值，无注解则 `camelToSnake(class.getSimpleName())` |
| `toCamelCase(snake)` | 下划线转驼峰（`created_at` → `createdAt`），带缓存 |
| `camelToSnake(camel)` | 驼峰转下划线（`FundWithdraw` → `fund_withdraw`） |
| `formatValue(val)` | 值格式化（与 AutoSignHandler.formatValue 完全一致，见下方） |
| `appendTimestamp(sb, entity, fields, tsField)` | tsField 非空时，`toCamelCase` 转驼峰，`fields.get(tsProp)` 检查字段存在后取值，追加 `|` + `formatValue`。字段不存在时取 null → `formatValue(null)` → `""` |
| `getFieldValue(entity, field)` | `field.get(entity)`，`catch(IllegalAccessException)` 返回 null |
| `setFieldValue(entity, field, value)` | `field.set(entity, value)`，`catch(IllegalAccessException)` + WARN 日志 |
| `setTimestampValue(entity, field, now)` | 根据字段类型设置时间值（见下方） |

**formatValue 类型处理顺序**（与 `AutoSignHandler.formatValue` 完全一致）:

| 检查顺序 | 类型 | 格式化结果 |
|---|---|---|
| 1 | `null` | `""` |
| 2 | `LocalDateTime` | `yyyy-MM-dd HH:mm:ss` |
| 3 | `java.sql.Timestamp` | `toLocalDateTime().format(FMT)` |
| 4 | `Date` | `new Timestamp(date.getTime()).toLocalDateTime().format(FMT)` |
| 5 | `BigDecimal` | `toPlainString()`（不使用科学计数法） |
| 6 | 其他 | `String.valueOf(val)` |

**setTimestampValue 类型兼容**: `LocalDateTime` / `Date`（写入 `java.sql.Timestamp`）/ `Long` 或 `long`（epoch millis，使用系统时区转换）/ `String`（`yyyy-MM-dd HH:mm:ss`）。其他类型不写入，避免类型不匹配。

> **与 AutoSignHandler.setTimestampValue 的差异**: `AutoSignHandler` 版本额外检查 `type == null`（`getFieldType` 返回 null 时跳过），拦截器版本使用 `field.getType()` 不会返回 null，无需此检查。`AutoSignHandler` 用 `long.class.isAssignableFrom(type)` 判断原始类型，拦截器用 `long.class == type`，两者效果一致。两者对不支持类型的处理结果一致：不写入。

**异常处理**:
- `intercept` 外层 `catch (Throwable)` — 签名异常不影响业务
- `fillEntitySafe` 外层 `catch (Throwable)` + 内层 `try-catch` 保护日志
- `getFieldValue` 捕获 `IllegalAccessException` 返回 null
- `setFieldValue` 捕获 `IllegalAccessException` + WARN 日志

**日志策略**:
- 真签名: **始终输出 INFO**（`table, op, class, type=REAL, signFields, hash前8位`）
- 假签名: 仅 `debug=true` 时输出 INFO
- 拦截到操作: 仅 `debug=true` 时输出 INFO
- Handler 已处理跳过: 仅 `debug=true` 时输出 INFO
- 签名字段全空（UPDATE）: WARN（`WarnPojo` JSON 格式）
- fieldCache 达上限: WARN 并清空重建（`WarnPojo` JSON 格式）
- 设置字段失败: WARN（`WarnPojo` JSON 格式）

> **与 AutoSignHandler 的差异**: 拦截器**无 `checkSaltOnce`** 盐值检查。设计原因：`AutoSignHandler` 先执行并已做盐值检查；若 `AutoSignHandler` 未注册（业务方有自定义 Handler），拦截器独立签名时盐值为空仅导致 hash 不匹配（无安全风险），不重复 WARN 避免日志冗余。

### 6d. AwsSaltInitializer — AWS 盐值初始化

**文件**: `config/AwsSaltInitializer.java`
**实现**: `InitializingBean`

**触发时机**: Spring Bean 初始化完成后（`afterPropertiesSet`）

**初始化流程**:

```
afterPropertiesSet:
  1. initGlobalSalt() — 全局盐值
     └── aws.isConfigured() → fetchSalt() → properties.setSalt()
  2. initTableSalts() — 表级盐值
     └── 遍历 tables，每表 hasAwsConfig() → fetchSalt() → tableConfig.setSalt()
  3. 汇总日志（仅 globalOk || tableCount > 0 时输出，无 AWS 配置时不产生日志）
```

**fetchSalt 逻辑**:
1. 构建 `SecretsManagerClient`（按 region 配置，region 为 null 时用 SDK 默认凭证链）
2. 调用 `GetSecretValue` API，获取 `secretString`
3. `secretString` 为 null 或空 → 返回 null
4. 解析 Secret：
   - 先尝试 JSON 解析：`OBJECT_MAPPER.readTree(secretString)` → 取 `secretKey` 对应的 `JsonNode`
   - `saltNode` 存在且非 `isNull()` → 返回 `saltNode.asText()`
   - `saltNode` 不存在或为 null，且 `secretString` 不以 `{` 开头 → 作为纯文本直接返回
   - `saltNode` 不存在但 `secretString` 以 `{` 开头（是 JSON 但 key 不匹配）→ 返回 null
5. `catch(Exception)` 捕获所有异常（含 `JsonProcessingException`、网络异常、权限异常），记录 ERROR 日志返回 null
6. `finally` 中关闭 client，关闭异常被 `catch(Exception ignored)` 吞掉

> **Secret 格式兼容**: 优先按 JSON 解析（`{"KEY":"value"}`），如果 Secret 内容不是 JSON（不以 `{` 开头），则作为纯文本盐值直接返回。注意：`readTree()` 对非 JSON 字符串可能抛 `JsonProcessingException`（被 catch 捕获返回 null），也可能不抛异常但 `saltNode` 为 null，此时再检查是否以 `{` 开头判断是否为纯文本。
>
> **凭证链**: `SecretsManagerClient.builder()` 不显式设置凭证，使用 AWS SDK 默认凭证链（环境变量 → 系统属性 → EC2/ECS/EKS IAM Role → ~/.aws/credentials）。
>
> **ObjectMapper**: `static final` 单例，避免每次 `fetchSalt` 调用重复创建。

**容错策略**:
- 无 AWS 配置 → 不产生 IO，直接跳过
- 获取失败 → 记录 ERROR 日志，回退到 YAML `salt` 配置
- Secret 中找不到 key → 记录 WARN 日志
- 表级获取失败 → 回退到全局盐值

**日志脱敏**:
- `secretName` 和 `secretKey` 在日志中使用 `SignUtils.mask()` 脱敏
- 不输出盐值明文

**性能优化**:
- `ObjectMapper` 为 `static final` 常量，避免重复创建
- `SecretsManagerClient` 在 `finally` 中关闭

### 6e. SignUtils — 签名工具类

**文件**: `util/SignUtils.java`
**类型**: `final class`，私有构造函数，全静态方法。不依赖外部库（如 commons-codec），减少 jar 包依赖。

| 方法 | 说明 |
|---|---|
| `sha256Hex(String)` | SHA-256 哈希，输入使用 `StandardCharsets.UTF_8` 编码，返回 64 位小写 hex |
| `bytesToHex(byte[])` | 字节数组转 hex 字符串（小写） |
| `mask(String)` | 敏感信息脱敏（中间 8 位替换为 `*`） |
| `verify(String, String)` | 时间恒定比较（防 timing attack） |

**SHA-256 性能优化**:
- 使用 `ThreadLocal<MessageDigest>` 缓存 `MessageDigest` 实例
- `MessageDigest` 非线程安全，`ThreadLocal` 保证每线程一个实例
- 避免每次调用 `MessageDigest.getInstance()` 的开销
- `ThreadLocal` 初始值：`MessageDigest.getInstance("SHA-256")`，异常时 `throw new RuntimeException(e)`
- 每次使用前 `md.reset()` 清除上次状态

**bytesToHex 实现**:
```java
private static final char[] HEX_CHARS = "0123456789abcdef".toCharArray();
// 预分配 char[] 数组，每个字节拆为高 4 位和低 4 位，分别查表
char[] hexChars = new char[bytes.length * 2];
for (int i = 0; i < bytes.length; i++) {
    int v = bytes[i] & 0xFF;           // 无符号化
    hexChars[i * 2] = HEX_CHARS[v >>> 4];     // 高 4 位
    hexChars[i * 2 + 1] = HEX_CHARS[v & 0x0F]; // 低 4 位
}
return new String(hexChars);
```

> **性能细节**: 使用预分配 `char[]` 而非 `StringBuilder`，避免动态扩容开销。`bytes[i] & 0xFF` 确保负字节值正确转换。`>>>` 无符号右移等价于 `>> 4` 后 `& 0x0F`，但更简洁。

**mask 脱敏规则**:

| 输入长度 | 规则 | 示例 |
|---|---|---|
| null/空 | `"***"` | → `***` |
| ≤ 4 | 全部 `*` | `abc` → `***` |
| 5~12 | 保留首尾各 1 位，中间 `*` | `WITHDRAW` → `W******W` |
| > 12 | 保留前后各 `(len-8)/2` 位，中间 8 个 `*` | `prod/bitgame/sign-salt` → `prod/bi********gn-salt` |

**verify 时间恒定比较**:
```java
if (expected == null || actual == null) {
    return false;
}
if (expected.length() != actual.length()) {
    return false;
}
int result = 0;
for (int i = 0; i < expected.length(); i++) {
    result |= expected.charAt(i) ^ actual.charAt(i);
}
return result == 0;
```
- 先检查 null（任一为 null 返回 false）
- 再比较长度（长度不同直接返回 false）
- 逐字符 XOR 比较，所有位都参与运算
- 防止攻击者通过响应时间推断正确字符

### 6f. SignVerifier — 验签工具类

**文件**: `util/SignVerifier.java`
**类型**: `final class`，私有构造函数，全静态方法

**三种验签模式**:

| 模式 | 方法 | 适用场景 |
|---|---|---|
| 实体对象验签 | `verifyEntity(entity, salt)` | 业务服务内，查出实体后校验 |
| 注解 Map 验签 | `verifyByAnnotation(columnValues, clazz, salt)` | binlog 监听，有实体类依赖 |
| 字段列表验签 | `verify(columnValues, fields, tsFields, hashField, salt)` | 独立部署，零依赖 |

**方法重载一览**:

| 方法签名 | 说明 |
|---|---|
| `verifyEntity(entity, salt)` | 默认 hash 属性名 `dataHash` |
| `verifyEntity(entity, hashProperty, salt)` | 自定义 hash 属性名（驼峰格式） |
| `verifyByAnnotation(columnValues, clazz, salt)` | 默认 hash 列名 `data_hash` |
| `verifyByAnnotation(columnValues, clazz, hashFieldName, salt)` | 自定义 hash 列名 |
| `verify(columnValues, orderedFields, tsFields, hashFieldName, salt)` | 字段列表驱动 |
| `computeHash(columnValues, orderedFields, tsFields, salt)` | 仅计算 hash，不做验证 |

**验签逻辑**:

`verifyEntity` (实体验签):
```
1. entity == null → return false
2. getFieldMetas(clazz) → 无 @SignTable 或无 @SignField → return false
3. actualHash = getStringFieldValue(entity, clazz, hashProperty)
4. actualHash == null 或 length != 64 → return false (DEBUG 日志)
5. 拼接签名串: field1|field2|...|ts1|ts2 + salt
   - salt == null → WARN 日志，不追加盐值（签名串末尾无盐值）
6. expectedHash = SHA-256(签名串)
7. SignUtils.verify(expectedHash, actualHash) — 时间恒定比较
8. 不匹配时输出 DEBUG 日志（hash 前 8 位对比）
```

`verifyByAnnotation` (注解 Map 验签):
```
1. getOrderedFields(clazz) → null 或 empty → return false (DEBUG 日志)
2. getTimestampFields(clazz) → [insertTs, updateTs]
3. 过滤 null/空 时间戳字段，构建 tsList
4. 委托 verify(columnValues, orderedFields, tsList, hashFieldName, salt)
```

`verify` (Map 验签):
```
1. columnValues == null 或 orderedFields == null/empty → return false
2. actualHash = columnValues.get(hashFieldName)
3. actualHash == null 或 isEmpty → return false (DEBUG 日志)
   注意: Map 验签不检查 length != 64，但 SignUtils.verify 会比较长度，
   长度不匹配时仍返回 false
4. computeHash() 计算预期 hash
5. SignUtils.verify(expectedHash, actualHash) — 时间恒定比较
6. 不匹配时输出 DEBUG 日志（hash 前 8 位对比）
```

> **差异注意**: `verifyEntity` 对 actualHash 做了 `length != 64` 的前置检查，而 `verify` (Map) 仅检查 `null/empty`。最终结果一致（长度不匹配时 `SignUtils.verify` 也会返回 false），但前置检查能更早返回并输出更精确的失败原因。
>
> **DEBUG 日志 substring 安全**: `verify` (Map) 的 DEBUG 日志中，`expectedHash` 使用 `Math.min(8, expectedHash.length())` 防止越界（`expectedHash` 始终为 64 位，此为防御性代码），`actualHash` 使用 `length >= 8 ? substring(0,8) : actualHash` 处理短 hash。`verifyEntity` 的 DEBUG 日志直接使用 `substring(0, 8)`，因为 `actualHash` 已通过 `length != 64` 前置检查。

**computeHash 方法**:
- 仅计算签名，不做验证
- 用于存量数据初始化、手动重新签名
- 不检查 actualHash，不输出验签失败日志
- `salt == null` 时不追加盐值（签名串末尾无盐值），与 `verifyEntity` 行为一致

**缓存**:
- `ANNOTATION_CACHE`: `Class → List<String>` — 注解解析的有序字段列表（列名），仅缓存正结果
- `TS_CACHE`: `Class → String[2]` — 时间戳字段名 `[insertTimestampField, updateTimestampField]`，有 `@SignTable` 时缓存（即使值为空 `""`），无 `@SignTable` 不缓存
- `FIELD_META_CACHE`: `Class → List<FieldMeta>` — 实体字段反射元信息（Field + order），仅缓存正结果

> **负缓存差异**: 与 `SignTableResolver` 的 `noAnnotationCache` 不同，`SignVerifier` 的三个缓存**不做负缓存**。无 `@SignTable` 或无 `@SignField` 时每次调用都会重新反射检查，不缓存 null 结果。这是因为验签调用频率远低于签名写入频率，且 `SignVerifier` 作为静态工具类无法依赖 Spring 生命周期管理缓存大小。

**注解解析方法**:

| 方法 | 说明 |
|---|---|
| `getOrderedFields(clazz)` | 从注解解析有序字段列表（列名），正结果缓存。`clazz == null` / 无 `@SignTable` / 无 `@SignField` 返回 null（不缓存负结果） |
| `getTimestampFields(clazz)` | 从 `@SignTable` 获取 `[insertTs, updateTs]`，有 `@SignTable` 时缓存（即使值为空 `""`）。`clazz == null` / 无 `@SignTable` 返回 `{"",""}`（不缓存） |
| `getFieldMetas(clazz)` | 从注解解析 `List<FieldMeta>`（Field + order），正结果缓存。无 `@SignTable` 或无 `@SignField` 返回 null（不缓存负结果） |

**反射工具方法**:

| 方法 | 说明 |
|---|---|
| `getFieldValueSafe(entity, field)` | `field.get(entity)`，`catch(Exception)` 返回 null |
| `getFieldValueByName(entity, clazz, prop)` | 遍历类及父类 `getDeclaredField(prop)`，`NoSuchFieldException` 继续向上找，其他异常返回 null |
| `getStringFieldValue(entity, clazz, prop)` | `getFieldValueByName` + `toString()`，null 返回 null |
| `appendEntityTimestamp(sb, entity, clazz, tsColumnName)` | tsColumnName 非空时，`columnToProperty` 转驼峰，`getFieldValueByName` 取值，追加 `|` + `formatValue` |
| `columnToProperty(column)` | 下划线转驼峰（`create_time` → `createTime`） |
| `javaFieldToColumnName(camel)` | 驼峰转下划线（`orderNo` → `order_no`） |

**`verifyByAnnotation` 委托逻辑**:
1. 调用 `getOrderedFields(clazz)` 获取有序字段列表（列名）
2. 调用 `getTimestampFields(clazz)` 获取 `[insertTs, updateTs]`
3. 过滤掉空/null 的时间戳字段，构建 `tsList`
4. 委托给 `verify(columnValues, orderedFields, tsList, hashFieldName, salt)`

> **注意**: `verifyByAnnotation` 会过滤掉空字符串时间戳字段（`@SignTable` 默认 `insertTimestampField=""` 时不参与签名），而 `verifyEntity` 的 `appendEntityTimestamp` 通过非空检查实现相同效果。

**`computeHash` null 安全**:
- `tsFields == null` → 不追加任何时间戳（`if (tsFields != null)` 守卫）
- `tsFields` 中单个 `tsField` 为 null 或空字符串 → 跳过该项（不追加 `|` 也不追加值）
- `salt == null` → 不追加盐值（`if (salt != null)` 守卫）。**注意**: 签名组件（`AutoSignHandler` / `SignMybatisInterceptor`）通过 `getEffectiveSalt` 获取盐值后直接 `sb.append(salt)`，null 时 `StringBuilder` 追加字符串 `"null"`。`SignVerifier` 用 `if (salt != null)` 守卫跳过。两者在 salt=null 时签名串不同，但 `SignProperties.salt` 默认 `""` 不会为 null，实际不会触发此差异
- `columnValues.get(field)` 返回 null → 追加空字符串 `""`

**内部类**:

| 类 | 字段 | 用途 |
|---|---|---|
| `FieldEntry` | `columnName, order` | Map 验签的排序辅助 |
| `FieldMeta` | `field, order` | 实体验签的字段元信息 |

**formatValue 规则**（与 AutoSignHandler 保持一致）:

| 类型 | 格式化结果 |
|---|---|
| `null` | `""` |
| `LocalDateTime` | `yyyy-MM-dd HH:mm:ss` |
| `java.sql.Timestamp` | 转 LocalDateTime 后格式化 |
| `Date` | 转 Timestamp → LocalDateTime 后格式化 |
| `BigDecimal` | `toPlainString()`（不使用科学计数法） |
| 其他 | `String.valueOf(val)` |

> **一致性保证**: `SignVerifier.formatValue` 与 `AutoSignHandler.formatValue` / `SignMybatisInterceptor.formatValue` 逻辑完全一致，确保验签时拼接的签名串与签名时一致。

### 6g. WarnPojo — 告警日志 POJO

**文件**: `config/WarnPojo.java`

**用途**: 将告警信息序列化为 JSON 格式输出，便于日志采集系统解析。

**字段**:

| 字段 | 类型 | 说明 |
|---|---|---|
| `bizType` | `String` | 业务类型标识，固定为 `"dbSignWarn"` |
| `warn` | `String` | 告警详细描述 |

**构造函数**: `WarnPojo(String bizType, String warn)`

**`toString()`**: 使用 `ObjectMapper` 序列化为 JSON，序列化失败时手动拼接 JSON 字符串。

```json
{"bizType":"dbSignWarn","warn":"表[xxx]是签名表，但更新时签名字段全为空..."}
```

> **fallback 细节**: 序列化失败时手动拼接 JSON，使用 `String.valueOf()` 处理 null 安全（`null` → `"null"`），`warn` 字段中的双引号 `"` 转义为 `\"`，`bizType` 字段不转义（固定值 `"dbSignWarn"` 不含特殊字符）。

**使用方式**: `log.warn("{}", new WarnPojo("dbSignWarn", "告警信息"))` — SLF4J 占位符调用 `toString()` 序列化。

**使用场景**:
- 盐值未配置告警
- UPDATE 时签名字段全为空告警
- fieldCache 达上限告警
- 设置字段失败告警

---

## 7. 签名计算流程

### 7a. 签名串拼接规则

```
签名串 = signField1|signField2|...|signFieldN|insertTimestamp|updateTimestamp + salt
```

- 字段间用 `|` 分隔
- 时间戳前有 `|` 分隔符
- **盐值直接拼接在最后一个时间戳之后，无 `|` 分隔符**
- 字段按 `order` 升序排列
- null 值格式化为空字符串 `""`
- **时间戳字段为 null 或空字符串时跳过**：`appendTimestamp` 方法检查 `tsField != null && !tsField.isEmpty()`，不满足时不追加任何内容（连 `|` 也不追加）
- **签名字段列表为空时**：`tableConfig.getSignFields()` 为 null 或空 → 生成假签名，不进入真签名计算

> **注意**: 如果 `insertTimestampField` 或 `updateTimestampField` 配置为空字符串 `""`，则该时间戳不参与签名串拼接。验签时也必须使用相同的配置，否则签名不匹配。

### 7b. 完整示例

```
字段配置: order_no(1), user_id(2), amount(3)
时间戳: create_time, last_update_time
盐值: mySecretSalt

INSERT 场景（组件填充两个时间戳为相同值）:
签名串 = DEP20260821001|1001|100.00|2026-08-21 21:00:00|2026-08-21 21:00:00mySecretSalt
→ SHA-256 → 64位hex

UPDATE 场景（create_time 不变，last_update_time 刷新）:
签名串 = DEP20260821001|1001|100.00|2026-08-21 21:00:00|2026-08-21 21:30:00mySecretSalt
→ SHA-256 → 64位hex
```

### 7c. 双层签名机制

```
INSERT/UPDATE 请求
  │
  ├── 1. AutoSignHandler (MetaObjectHandler)
  │     └── MyBatis Plus 方法触发
  │         └── 计算 hash → 写入 MetaObject
  │
  └── 2. SignMybatisInterceptor (Interceptor)
        └── Executor.update 拦截
            └── 检查 hash 是否已有 64 位值
                ├── 有 → 跳过（Handler 已处理）
                └── 无 → 计算 hash → 反射写入实体
```

**设计意图**: 确保所有 INSERT/UPDATE 路径都被覆盖，即使业务方使用了自定义 SQL 或 Wrapper。

---

## 8. 盐值管理体系

### 8a. 两级盐值架构

```
全局盐值 (properties.salt)
  ├── 来源: YAML bitgame.sign.salt 或 AWS 全局 Secret
  ├── 适用: 所有未配置表级盐值的表
  └── 覆盖: AwsSaltInitializer.initGlobalSalt() → properties.setSalt()

表级盐值 (tableConfig.salt)
  ├── 来源: YAML tables.xxx.salt 或 AWS 表级 Secret
  ├── 适用: 仅该表
  └── 覆盖: AwsSaltInitializer.initTableSalts() → tableConfig.setSalt()
```

### 8b. 盐值优先级

```
getEffectiveSalt(globalSalt):
  if (tableConfig.salt != null && !empty)
      return tableConfig.salt    // 表级优先
  else
      return globalSalt          // 回退全局
```

> **盐值追加行为**: 签名组件（`AutoSignHandler` / `SignMybatisInterceptor`）使用 `sb.append(config.getEffectiveSalt(properties.getSalt()))` 直接追加，`SignProperties.salt` 默认 `""` 确保不会追加 `"null"`。验签组件（`SignVerifier`）使用 `if (salt != null) { sb.append(salt); }` 守卫式追加。两者在 `salt == ""` 时行为一致（追加空字符串），在 `salt == null` 时有差异（签名追加 `"null"`，验签不追加），但 `SignProperties.salt` 默认 `""` 避免了此场景。

### 8c. AWS Secrets Manager 集成

**全局 Secret**:
```json
{"WITHDRAW_SIGN_SALT": "global_salt_value"}
```

**表级 Secret**:
```json
{"WITHDRAW_SIGN_SALT": "table_specific_salt_value"}
```

**Secret 格式兼容**:
- JSON 格式: 按 `secretKey` 取值
- 纯文本格式: 直接作为盐值

### 8d. 验签时获取盐值的正确方式

```java
// ✅ 正确：注入 SignProperties，使用 getEffectiveSalt
@Autowired
private SignProperties signProperties;

SignTableConfig tableConfig = signProperties.getTables().get("tableName");
String salt = tableConfig.getEffectiveSalt(signProperties.getSalt());

// ❌ 错误：@Value 拿到的是 YAML 原始值，不是 AWS 覆盖后的值
@Value("${bitgame.sign.salt}")
private String salt;  // 这不会反映 AWS 获取的盐值
```

> **原因**: `AwsSaltInitializer` 通过 `properties.setSalt()` 更新 `SignProperties` Bean，不更新 Spring Environment。`@Value` 在 Bean 创建时从 Environment 读取，之后不会更新。

---

## 9. 异常处理与防御性编码

### 9a. 核心原则

**任何签名组件异常都不影响业务执行，不影响其他字段值。**

### 9b. 异常处理架构

```
业务调用 (insert/update)
  │
  └── AutoSignHandler.insertFill / updateFill
        try { doFill() }
        catch (Throwable e) {
            try { 日志记录 } catch (Throwable) { 兜底日志 }
        }
  │
  └── SignMybatisInterceptor.intercept
        try { processParameter() }
        catch (Throwable e) { 日志记录 }
        return invocation.proceed()  ← try-catch 后始终执行，业务继续

  └── fillEntitySafe
        try { fillEntity() }
        catch (Throwable e) {
            try { 日志记录 } catch (Throwable) { 兜底日志 }
        }
```

### 9c. 双层 try-catch 保护

外层 `catch` 中的日志调用本身可能抛异常（如 `metaObject.getOriginalObject()` 为 null），因此内层再包一层 `try-catch`:

```java
} catch (Throwable e) {
    try {
        log.error("..., class={}, table={}",
                metaObject.getOriginalObject().getClass().getName(),  // 可能 NPE
                resolveTableName(metaObject));
    } catch (Throwable logEx) {
        log.error("[bitgame-sign] xxx 异常, 日志记录也失败", e);
    }
}
```

### 9d. 异常保护矩阵

| 场景 | 保护机制 | 结果 |
|---|---|---|
| 签名计算异常 | `catch(Throwable)` | INSERT/UPDATE 正常执行，data_hash 为空 |
| AWS 获取盐值失败 | `catch(Exception)` 回退本地 salt | 服务正常启动 |
| 批量插入某条异常 | 逐条 `catch(Throwable)` | 其余记录正常签名 |
| 反射获取字段失败 | 加入负缓存跳过 | 后续请求秒跳，不重复尝试 |
| 类型不匹配 | 直接跳过不写入 | 不影响其他字段值 |
| 日志调用本身异常 | 内层 `try-catch` | 兜底日志，不穿透业务层 |

---

## 10. 缓存策略

### 10a. 缓存一览

| 缓存 | 所在类 | Key | Value | 作用 |
|---|---|---|---|---|
| `annotationCache` | SignTableResolver | `Class<?>` | `SignTableConfig` | 注解解析结果 |
| `noAnnotationCache` | SignTableResolver | `Class<?>` | `Boolean` | 负缓存（无注解） |
| `sortedFieldsCache` | AutoSignHandler | `String`(表名) | `List<SignFieldConfig>` | 排序后字段 |
| `camelCaseCache` | AutoSignHandler | `String` | `String` | 下划线→驼峰 |
| `fieldCache` | SignMybatisInterceptor | `Class<?>` | `Map<String, Field>` | 类字段反射 |
| `sortedFieldsCache` | SignMybatisInterceptor | `String`(表名) | `List<SignFieldConfig>` | 排序后字段 |
| `camelCaseCache` | SignMybatisInterceptor | `String` | `String` | 下划线→驼峰 |
| `skipClasses` | SignMybatisInterceptor | `Class<?>` | `Boolean` | 负缓存（无 hash 字段） |
| `ANNOTATION_CACHE` | SignVerifier | `Class<?>` | `List<String>` | 注解有序字段列表 |
| `TS_CACHE` | SignVerifier | `Class<?>` | `String[2]` | 时间戳字段名 |
| `FIELD_META_CACHE` | SignVerifier | `Class<?>` | `List<FieldMeta>` | 实体字段元信息 |
| `MD_HOLDER` | SignUtils | ThreadLocal | `MessageDigest` | SHA-256 实例 |

### 10b. 缓存特点

- 全部使用 `ConcurrentHashMap`，线程安全
- 使用 `putIfAbsent` 避免竞态条件
- `fieldCache` 有上限 512 个类，超限清空重建（防止动态类加载 OOM）
- 排序后字段列表转为 `Collections.unmodifiableList`，防止意外修改
- `MessageDigest` 使用 `ThreadLocal`（非线程安全类）

---

## 11. 线程安全设计

| 共享状态 | 保护方式 |
|---|---|
| `ConcurrentHashMap` 缓存 | 并发安全容器 + `putIfAbsent` |
| `SignProperties` | Spring 单例 Bean，`setSalt()` 在启动阶段单线程执行 |
| `MessageDigest` | `ThreadLocal` 隔离 |
| `saltChecked` (AutoSignHandler) | `volatile` 保证可见性。`checkSaltOnce` 的 check-then-set 非原子操作，并发时可能输出多条 WARN 日志，但仅影响日志冗余，不影响正确性 |
| 排序后字段列表 | `Collections.unmodifiableList` 不可变 |

---

## 12. 日志策略

### 12a. 日志级别

| 级别 | 场景 |
|---|---|
| **INFO** | 真签名发生（始终输出）、组件启动、AWS 盐值初始化成功 |
| **INFO (debug 门控)** | 假签名日志、Resolver 解析日志、拦截器拦截日志、Handler 已处理跳过 — 使用 `log.info` 但受 `bitgame.sign.debug=true` 开关控制 |
| **WARN** | 盐值未配置、签名字段全空（UPDATE）、AWS Secret 未找到值、fieldCache 超限、验签时盐值为 null |
| **ERROR** | 签名计算异常、AWS 获取失败、解析 Secret 失败 |
| **DEBUG** | 验签失败详情（`log.debug`，需日志框架级别允许 DEBUG 输出） |

> **注意**: "debug 门控" 是组件级开关 `bitgame.sign.debug`，与 SLF4J 日志级别不同。假签名/Resolver/拦截器日志使用 `log.info` 但仅在 `debug=true` 时执行；验签失败日志使用 `log.debug`，需同时满足 `debug=true` 且日志框架级别允许 DEBUG。

### 12b. 日志脱敏规则

| 敏感数据 | 脱敏方式 | 示例 |
|---|---|---|
| `secretName` | `SignUtils.mask()` 中间 8 位替换为 `*` | `prod/bitgame/sign-salt` → `prod/bi********gn-salt` |
| `secretKey` | `SignUtils.mask()` | `WITHDRAW_SIGN_SALT` → `WITHD********_SALT` |
| 盐值明文 | **永不输出** | — |
| 完整 hash | 只输出前 8 位 + `...` | `e7b2c1d4...` |
| 完整签名串 | **永不输出** | — |
| Secret JSON 内容 | **永不输出** | — |

**脱敏实现** (`SignUtils.mask()`):

| 输入长度 | 规则 | 示例 |
|---|---|---|
| null / 空 | `"***"` | → `***` |
| ≤ 4 | 全部 `*` | `abc` → `***` |
| 5~12 | 保留首尾各 1 位，中间 `*` | `WITHDRAW` → `W******W` |
| > 12 | 保留前后各 `(len-8)/2` 位，中间 8 个 `*` | `prod/bitgame/sign-salt` → `prod/bi********gn-salt` |

### 12c. 日志格式规范

**统一前缀**: 所有日志以 `[bitgame-sign]` 开头，便于日志检索和过滤。

**模块标识**: 前缀后跟模块标识，区分日志来源：

| 模块标识 | 来源类 | 说明 |
|---|---|---|
| `[Handler]` | `AutoSignHandler` | MyBatis Plus MetaObjectHandler |
| `[Interceptor]` | `SignMybatisInterceptor` | MyBatis 拦截器 |
| `[Resolver]` | `SignTableResolver` | 配置解析器 |
| 无标识 | `AwsSaltInitializer` / `SignAutoConfiguration` / `SignVerifier` | 启动/初始化/验签 |

**日志示例**:

```
[bitgame-sign] 自动签名组件已启用, hashField=data_hash, 配置表数=3, 表级AWS=1
[bitgame-sign] MyBatis 拦截器已注册
[bitgame-sign] [Handler] table=fund_withdrawl_order, op=INSERT, type=REAL, signFields=5, hash=e7b2c1d4...
[bitgame-sign] [Interceptor] table=fund_withdrawl_order, op=UPDATE, class=FundWithdrawlOrder, type=REAL, signFields=5, hash=a1b2c3d4...
[bitgame-sign] [Resolver] table=fund_withdrawl_order, source=YAML, fields=5
[bitgame-sign] [Interceptor] Handler已处理跳过, class=FundWithdrawlOrder, hash=e7b2c1d4...
[bitgame-sign] 已从 AWS Secrets Manager 获取全局盐值, secretName=prod/bi********gn-salt, secretKey=WITHD********_SALT
[bitgame-sign] AWS 盐值初始化完成, 全局=成功, 表级=2
```

### 12d. 日志输出规则

**规则 1: 真签名日志始终输出**

真签名是核心业务行为，必须始终记录 INFO 日志，不受 `debug` 开关控制：

```java
// ✅ 真签名：始终输出
log.info("[bitgame-sign] [Handler] table={}, op={}, type=REAL, signFields={}, hash={}...",
        tableName, isUpdate ? "UPDATE" : "INSERT",
        tableConfig.getSignFields().size(),
        hash.substring(0, 8));
```

**规则 2: 假签名日志受 debug 控制**

假签名高频出现（每个非签名表的 INSERT/UPDATE 都触发），仅 debug 模式输出：

```java
// 假签名：仅 debug=true 时输出
if (debugOn) {
    log.info("[bitgame-sign] [Handler] table={}, op={}, type=FAKE, hash={}...",
            tableName, isUpdate ? "UPDATE" : "INSERT",
            hash.substring(0, 8));
}
```

**规则 3: WARN 日志用于可预期的不正常状态**

WARN 不代表系统故障，而是需要人工关注的状态：

| WARN 场景 | 含义 | 建议操作 |
|---|---|---|
| 盐值未配置 | 真签名无安全性 | 配置 `bitgame.sign.salt` 或 AWS Secret |
| 签名字段全空（UPDATE） | 可能是部分更新，签名基于空值计算 | 改为先 `selectById` 再 `updateById` |
| AWS Secret 未找到值 | Secret 存在但 key 不匹配 | 检查 `secret-key` 配置 |
| fieldCache 达上限 | 动态类加载过多 | 排查是否有异常类生成 |

**规则 4: ERROR 日志仅用于真正的异常**

ERROR 表示签名组件遇到了非预期异常，但业务不受影响：

| ERROR 场景 | 含义 | 影响 |
|---|---|---|
| 签名计算异常 | `doFill` / `fillEntity` 抛 Throwable | data_hash 为空，业务正常 |
| AWS 获取盐值失败 | 网络异常 / 权限不足 / Secret 不存在 | 回退 YAML salt |
| 解析 Secret 失败 | JSON 格式错误 / 网络异常 | 返回 null，回退 |
| 日志记录也失败 | 日志调用本身抛异常 | 兜底输出原始异常 |

**规则 5: DEBUG 日志用于排查问题**

DEBUG 日志默认不输出，开启 `bitgame.sign.debug=true` 后输出详细信息：

| DEBUG 场景 | 输出内容 |
|---|---|
| 验签失败 | hash 不匹配，expected 前 8 位 vs actual 前 8 位 |
| 验签失败（无配置） | 无签名字段配置（`verifyEntity` / `verifyByAnnotation`） |
| 验签失败（参数为空） | `verify` (Map) 的 `columnValues` 或 `orderedFields` 为 null/empty |
| **验签失败（hash 缺失）** | hash 值为 null 或长度非 64（仅 `verifyEntity` 检查长度） |
| 假签名生成 | table, op, type=FAKE, hash 前 8 位 |
| Resolver 解析 | table, source=YAML/ANNOTATION, fields 数量 |
| 拦截器拦截 | msId, cmdType, paramType |
| Handler 已处理跳过 | class, hash 前 8 位 |

> **`log.isDebugEnabled()` 守卫**: `SignVerifier` 中所有 DEBUG 日志均使用 `if (log.isDebugEnabled())` 守卫，即使 `bitgame.sign.debug=true` 但日志框架级别高于 DEBUG 时，也不会执行字符串拼接，避免无效开销。这是独立于组件 `debug` 开关的 SLF4J 级别检查。
>
> **`hash.substring` 安全性差异**: `verifyEntity` 在 DEBUG 日志中直接使用 `actualHash.substring(0, 8)`（安全，因为已验证 `length == 64`）；`verify` (Map) 使用 `actualHash.length() >= 8 ? actualHash.substring(0, 8) : actualHash`（安全，因为仅检查了 `null/empty`，hash 可能短于 8 位）。`expectedHash` 在 `verify` (Map) 中使用 `Math.min(8, expectedHash.length())` 防御性截断（`expectedHash` 始终为 64 位 hex，此为防御性代码），在 `verifyEntity` 中直接 `substring(0, 8)`（安全）。

### 12e. 日志安全注意事项

| 注意事项 | 说明 |
|---|---|
| **盐值永不入日志** | 盐值是签名安全的核心，泄露后攻击者可伪造签名。所有日志中均不输出盐值明文 |
| **hash 截断输出** | 只输出前 8 位 + `...`，防止完整 hash 被用于离线分析 |
| **Secret 内容不入日志** | `fetchSalt` 异常时只输出 secretName（脱敏），不输出 secretString 内容 |
| **签名串不入日志** | 签名串包含所有字段值，可能包含业务敏感数据，永不输出 |
| **secretName / secretKey 脱敏** | 使用 `SignUtils.mask()` 脱敏，防止 Secret 路径泄露 |
| **异常堆栈不包含敏感数据** | 异常对象中不携带盐值、Secret 内容等敏感信息 |
| **WarnPojo JSON 格式** | 告警日志使用 `WarnPojo` 序列化为 JSON，便于日志采集系统（如 ELK）结构化解析和告警 |

### 12f. 日志性能注意事项

| 注意事项 | 说明 |
|---|---|
| **debug 开关热配置** | `bitgame.sign.debug` 可通过 Spring Cloud Config 等热更新，无需重启 |
| **假签名日志门控** | 假签名高频出现，必须受 `debug` 门控，否则生产环境日志量爆炸 |
| **Resolver 日志门控** | 配置解析每次请求都触发，必须受 `debug` 门控 |
| **拦截器拦截日志门控** | 每次 INSERT/UPDATE 都触发，必须受 `debug` 门控 |
| **真签名日志不过度** | 真签名日志只输出关键信息（表名、操作类型、hash 前 8 位），不输出字段值 |
| **日志参数惰性求值** | 使用 SLF4J 占位符 `{}`，日志级别不满足时不拼接字符串 |
| **`log.isDebugEnabled()` 守卫** | `SignVerifier` 的 DEBUG 日志前有 `isDebugEnabled()` 检查，避免无效字符串拼接 |
| **hash.substring 安全** | hash 为 64 位 hex，`substring(0, 8)` 不会越界 |

### 12g. 日志与异常处理的协作

**日志调用本身可能失败，必须双层保护**:

```
业务调用
  └── try { 签名逻辑 }
        catch (Throwable e) {
            try {
                log.error("签名异常, class={}, table={}",
                        metaObject.getOriginalObject().getClass().getName(),  // ← 可能 NPE
                        resolveTableName(metaObject));                         // ← 可能反射异常
            } catch (Throwable logEx) {
                // 兜底：只输出原始异常，不访问任何可能失败的对象
                log.error("[bitgame-sign] xxx 异常, 日志记录也失败", e);
            }
        }
```

**为什么日志调用会失败**:
- `metaObject.getOriginalObject()` 可能为 null → NPE
- `resolveTableName()` 内部反射 `getAnnotation()` 可能失败
- `entity.getClass().getName()` 极端情况下可能失败
- 日志框架本身可能异常（如磁盘满、日志队列满）

**兜底日志原则**: 兜底日志只输出原始异常对象 `e`，不访问任何可能失败的对象，确保兜底日志本身不会失败。

### 12h. 日志检索指南

**检索真签名行为**:
```
# 检索所有真签名日志
grep "\[bitgame-sign\].*type=REAL" app.log

# 检索特定表的真签名
grep "\[bitgame-sign\].*table=fund_withdrawl_order.*type=REAL" app.log

# 检索 INSERT 操作
grep "\[bitgame-sign\].*op=INSERT.*type=REAL" app.log
```

**检索异常**:
```
# 检索所有 ERROR
grep "\[bitgame-sign\].*ERROR\|\[bitgame-sign\].*异常" app.log

# 检索 AWS 获取失败
grep "\[bitgame-sign\].*AWS.*失败" app.log
```

**检索告警**:
```
# 检索所有 WARN（JSON 格式）
grep "dbSignWarn" app.log

# 检索盐值未配置
grep "盐值未配置" app.log

# 检索部分更新告警
grep "签名字段全为空" app.log
```

**检索验签失败（需开启 debug）**:
```
grep "\[bitgame-sign\].*验签失败" app.log
```

---

## 13. 完整数据流

### 13a. 启动流程

```
Spring Boot 启动
  │
  ├── 扫描 spring.factories → SignAutoConfiguration
  │     ├── @ConditionalOnClass(MyBatis Plus) ✓
  │     └── @ConditionalOnProperty(enabled=true) ✓
  │
  ├── 创建 SignProperties (绑定 YAML 配置)
  │
  ├── 创建 SignTableResolver(properties)
  │
  ├── 创建 AutoSignHandler(properties, resolver) [如无自定义 Handler]
  │     └── 日志: "自动签名组件已启用, hashField=data_hash, 配置表数=N"
  │
  ├── 创建 SignMybatisInterceptor(properties, resolver)
  │     └── 日志: "MyBatis 拦截器已注册"
  │
  └── 创建 AwsSaltInitializer(properties)
        └── afterPropertiesSet()
              ├── initGlobalSalt() → fetchSalt() → properties.setSalt()
              ├── initTableSalts() → 遍历 tables → fetchSalt() → tableConfig.setSalt()
              └── 日志: "AWS 盐值初始化完成"
```

### 13b. INSERT 流程

```
业务代码: mapper.insert(entity)
  │
  ├── MyBatis Plus → AutoSignHandler.insertFill(metaObject)
  │     ├── checkSaltOnce() — 首次检查盐值
  │     ├── fillTimestamps() — 填充 create_time + last_update_time
  │     ├── resolve() — 获取签名配置
  │     ├── computeRealHash() 或 generateFakeHash()
  │     └── metaObject.setValue("dataHash", hash)
  │
  ├── MyBatis → SignMybatisInterceptor.intercept()
  │     ├── 检查 hash 是否已有 64 位值
  │     ├── 有 → 跳过（Handler 已处理）
  │     └── 无 → fillEntity() → 计算 hash → 反射写入
  │
  └── SQL 执行: INSERT INTO ... (data_hash='e7b2c1d4...')
```

### 13c. UPDATE 流程

```
业务代码: mapper.updateById(entity)
  │
  ├── MyBatis Plus → AutoSignHandler.updateFill(metaObject)
  │     ├── fillTimestamps() — 刷新 last_update_time
  │     ├── resolve() — 获取签名配置
  │     ├── 检测签名字段是否全空 → WARN
  │     ├── computeRealHash() — 基于当前实体值重新计算
  │     └── metaObject.setValue("dataHash", newHash)
  │
  ├── MyBatis → SignMybatisInterceptor.intercept()
  │     └── 同 INSERT，检查已有 hash 则跳过
  │
  └── SQL 执行: UPDATE ... SET data_hash='a1b2c3d4...'
```

### 13d. 验签流程

```
方式一: 实体验签
  SignVerifier.verifyEntity(entity, salt)
    ├── 从实体反射获取 @SignField 字段（带缓存）
    ├── 获取 actualHash = entity.dataHash
    ├── 拼接签名串: field1|field2|...|ts1|ts2 + salt
    ├── expectedHash = SHA-256(签名串)
    └── SignUtils.verify(expectedHash, actualHash) — 时间恒定比较

方式二: Map 验签 (binlog)
  SignVerifier.verify(columnValues, fields, tsFields, "data_hash", salt)
    ├── actualHash = columnValues.get("data_hash")
    ├── 拼接签名串: columnValues.get(field1)|...|columnValues.get(ts1)|... + salt
    ├── expectedHash = SHA-256(签名串)
    └── SignUtils.verify(expectedHash, actualHash)
```

---

## 14. 使用指南

### 14a. 业务方接入步骤

1. **引入依赖**:
```xml
<dependency>
    <groupId>com.bitgame</groupId>
    <artifactId>bitgame-sign-spring-boot-starter</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

2. **DDL 变更**: 添加 `data_hash` 列，去除 `ON UPDATE CURRENT_TIMESTAMP`
```sql
ALTER TABLE fund_withdrawl_order ADD COLUMN data_hash VARCHAR(64) DEFAULT NULL COMMENT '数据签名';

-- 去除数据库层面的自动更新时间戳，交由组件控制
ALTER TABLE fund_withdrawl_order MODIFY COLUMN last_update_time DATETIME
    COMMENT '更新时间(组件自动填充)';
-- 不要使用 ON UPDATE CURRENT_TIMESTAMP
```

3. **YAML 配置**: 配置 `bitgame.sign.tables.表名.sign-fields`
```yaml
bitgame:
  sign:
    enabled: true
    salt: ${WITHDRAW_SIGN_SALT:dev_salt}
    hash-field-name: data_hash
    timestamp-fields:
      - create_time
      - last_update_time
    tables:
      fund_withdrawl_order:
        sign-fields:
          - { name: order_no, order: 1 }
          - { name: user_id, order: 2 }
          - { name: amount, order: 3 }
        insert-timestamp-field: create_time
        update-timestamp-field: last_update_time
```

4. **实体类**: 确保有 `dataHash` 属性（YAML 驱动）或添加 `@SignTable` + `@SignField` 注解
```java
// 方式一：YAML 驱动（无需注解，只需 hash 属性）
@TableName("fund_withdrawl_order")
public class FundWithdrawlOrder {
    private String orderNo;
    private Long userId;
    private BigDecimal amount;
    private LocalDateTime createTime;
    private LocalDateTime lastUpdateTime;
    private String dataHash;  // 对应 data_hash 列
    // getter / setter ...
}

// 方式二：注解驱动（无需 YAML tables 配置）
@SignTable(insertTimestampField = "create_time", updateTimestampField = "last_update_time")
@TableName("fund_withdrawl_order")
public class FundWithdrawlOrder {
    @SignField(order = 1)
    private String orderNo;
    @SignField(order = 2)
    private Long userId;
    @SignField(order = 3)
    private BigDecimal amount;
    private LocalDateTime createTime;
    private LocalDateTime lastUpdateTime;
    @TableField("data_hash")
    private String dataHash;
    // getter / setter ...
}
```

5. **业务代码**: 无需改动，`mapper.insert()` / `mapper.updateById()` 自动签名
```java
// INSERT 自动签名
FundWithdrawlOrder order = new FundWithdrawlOrder();
order.setOrderNo("DEP20260822001");
order.setUserId(1001L);
order.setAmount(new BigDecimal("100.00"));
mapper.insert(order);  // data_hash 自动填充

// UPDATE 自动重新签名
order.setAmount(new BigDecimal("200.00"));
mapper.updateById(order);  // data_hash 自动刷新
```

6. **验签**: 注入 `SignProperties`，使用 `SignVerifier.verifyEntity()` 验签
```java
@Autowired
private SignProperties signProperties;

public boolean verifyOrder(Long id) {
    FundWithdrawlOrder order = mapper.selectById(id);
    SignTableConfig tableConfig = signProperties.getTables().get("fund_withdrawl_order");
    String salt = tableConfig.getEffectiveSalt(signProperties.getSalt());
    return SignVerifier.verifyEntity(order, salt);
}
```

### 14b. 业务代码约束

- **UPDATE 必须先查后改**: `selectById` → 修改 → `updateById`，不能使用 Wrapper 部分更新
- **不能使用 `ON UPDATE CURRENT_TIMESTAMP`**: 时间戳由组件控制
- **验签盐值必须从 `SignProperties` 获取**: 不能用 `@Value`

### 14c. 存量数据初始化

```java
@Autowired
private SignProperties signProperties;

public void initSign() {
    SignTableConfig tableConfig = signProperties.getTables().get("table_name");
    String salt = tableConfig.getEffectiveSalt(signProperties.getSalt());

    // 分页查询未签名记录
    // 对每条记录调用 SignVerifier.computeHash() 计算 hash
    // UPDATE 写回 data_hash
}
```

---

## 15. 设计决策记录

| 决策 | 原因 |
|---|---|
| YAML 优先于注解 | YAML 可热更新，注解需改代码重新部署 |
| 双层签名（Handler + Interceptor） | Handler 覆盖 MyBatis Plus 方法，Interceptor 兜底自定义 SQL |
| 假签名机制 | 不在配置中但有 hash 字段的表生成假签名，格式与真签名一致，防止攻击者识别哪些表有真签名 |
| 时间恒定比较 | 防止 timing attack 攻击者通过响应时间推断正确 hash |
| ThreadLocal MessageDigest | MessageDigest 非线程安全，ThreadLocal 避免每次创建 |
| ConcurrentHashMap + putIfAbsent | 线程安全缓存，避免竞态重复构建 |
| 负缓存 | 无 hash 字段的类、无注解的类加入负缓存，后续请求 O(1) 跳过 |
| fieldCache 上限 512 | 防止动态类加载场景（如 Groovy 脚本）导致 OOM |
| AWS SDK compile scope | 业务方无需单独引入 AWS 依赖 |
| 其他依赖 provided scope | 由业务项目提供，避免版本冲突 |
| 盐值无 `\|` 分隔符 | 签名串末尾直接拼接盐值，简化格式 |
| 两个时间戳都参与签名 | 验签时无需区分 INSERT/UPDATE，统一逻辑 |
| catch(Throwable) 而非 catch(Exception) | 捕获 Error 级别异常（如 StackOverflowError），确保业务不受影响 |
| 双层 try-catch | 日志调用本身可能抛异常，内层保护防止穿透 |

---

## 16. 安全审计

### 16a. 安全关注点与实现

| 关注点 | 风险 | 实现措施 | 所在类/方法 |
|---|---|---|---|
| **Timing Attack** | 攻击者通过响应时间逐字符推断正确 hash | 时间恒定比较：先 null 检查和长度检查（提前返回），再逐字符 XOR 累积，所有位都参与运算 | `SignUtils.verify()` |
| **盐值泄露** | 日志中输出盐值明文，攻击者可伪造签名 | 日志中盐值永不输出；secretName/secretKey 使用 `SignUtils.mask()` 脱敏 | `AwsSaltInitializer` 全部日志 |
| **hash 值泄露** | 完整 hash 可用于离线碰撞 | 日志只输出 hash 前 8 位 + `...` | `AutoSignHandler`、`SignMybatisInterceptor`、`SignVerifier` |
| **假签名可辨识** | 攻击者通过 hash 特征判断哪些表有真签名 | 假签名使用 `ThreadLocalRandom` 生成 64 位随机 hex，格式与真 SHA-256 完全一致 | `AutoSignHandler.generateFakeHash()`、`SignMybatisInterceptor.generateFakeHash()` |
| **盐值硬编码** | 代码或配置中硬编码盐值 | 支持环境变量注入 + AWS Secrets Manager 动态获取 | `SignProperties.salt`、`AwsSaltInitializer` |
| **反射访问安全** | `setAccessible(true)` 绕过访问控制 | 仅对自身组件需要的字段操作，不暴露敏感字段；字段缓存有上限防止滥用 | `SignMybatisInterceptor.getFields()`、`SignVerifier.getFieldMetas()` |
| **AWS 凭证安全** | AK/SK 硬编码在配置文件 | 依赖 IAM Role（ECS/EKS/EC2/Lambda 自动注入），不硬编码 | `AwsSaltInitializer.fetchSalt()` 使用 SDK 默认凭证链 |
| **签名串重构攻击** | 攻击者篡改字段值后重新计算 hash | 盐值保密（AWS Secrets Manager），攻击者无法重算签名 | `computeRealHash()` 追加盐值后 SHA-256 |
| **Secret 内容注入** | Secret 内容被注入到日志或异常信息 | Secret 解析失败时只输出 secretName（脱敏），不输出 secretString 内容 | `AwsSaltInitializer.fetchSalt()` catch 块 |

### 16b. 安全检查清单

- [x] 日志不输出盐值明文
- [x] 日志不输出完整 hash 值
- [x] secretName / secretKey 在日志中脱敏
- [x] hash 比较使用时间恒定算法
- [x] 假签名格式与真签名一致
- [x] AWS 凭证不硬编码
- [x] Secret 解析异常不暴露 Secret 内容
- [x] 反射操作仅限组件内部字段
- [x] 字段缓存有上限防止内存滥用

---

## 17. 线程安全审计

### 17a. 共享状态分析

| 共享状态 | 类型 | 线程安全保证 | 风险评估 |
|---|---|---|---|
| `SignProperties` | Spring 单例 Bean | 启动阶段单线程写入（`setSalt`），运行时多线程只读 | **低风险**：`setSalt` 在 `afterPropertiesSet` 中执行，此时 Bean 初始化完成但未对外暴露 |
| `sortedFieldsCache` (Handler) | `ConcurrentHashMap` | `putIfAbsent` 保证不重复构建 | **安全** |
| `sortedFieldsCache` (Interceptor) | `ConcurrentHashMap` | `putIfAbsent` 保证不重复构建 | **安全** |
| `camelCaseCache` (Handler) | `ConcurrentHashMap` | `putIfAbsent` | **安全** |
| `camelCaseCache` (Interceptor) | `ConcurrentHashMap` | `putIfAbsent` | **安全** |
| `fieldCache` (Interceptor) | `ConcurrentHashMap` | `putIfAbsent`，上限 512 | **安全**：超限 `clear()` 有极小窗口重复构建，不影响正确性 |
| `skipClasses` (Interceptor) | `ConcurrentHashMap` | `putIfAbsent` | **安全** |
| `annotationCache` (Resolver) | `ConcurrentHashMap` | `putIfAbsent` | **安全** |
| `noAnnotationCache` (Resolver) | `ConcurrentHashMap` | `putIfAbsent` | **安全** |
| `ANNOTATION_CACHE` (Verifier) | `ConcurrentHashMap` | `putIfAbsent` | **安全** |
| `TS_CACHE` (Verifier) | `ConcurrentHashMap` | `putIfAbsent` | **安全** |
| `FIELD_META_CACHE` (Verifier) | `ConcurrentHashMap` | `putIfAbsent` | **安全** |
| `MD_HOLDER` (SignUtils) | `ThreadLocal<MessageDigest>` | 每线程独立实例 | **安全**：`ThreadLocal` 保证线程隔离 |
| `saltChecked` (Handler) | `volatile boolean` | `volatile` 保证可见性 | **安全**：首次检查后设为 true，后续直接跳过 |
| 排序后字段列表 | `Collections.unmodifiableList` | 不可变列表 | **安全**：创建后不可修改 |

### 17b. 线程安全设计原则

1. **所有缓存使用 `ConcurrentHashMap`**：并发读写安全
2. **写入使用 `putIfAbsent`**：避免竞态条件下重复构建，且不会覆盖已有值
3. **`ThreadLocal` 隔离非线程安全类**：`MessageDigest` 非线程安全，通过 `ThreadLocal` 每线程一个实例
4. **`volatile` 保证可见性**：`saltChecked` 标志位使用 `volatile`，确保所有线程看到最新值
5. **不可变列表**：排序后的字段列表转为 `unmodifiableList`，防止被意外修改
6. **无锁设计**：全部使用 `ConcurrentHashMap` + `putIfAbsent`，无 `synchronized` 块，无死锁风险

### 17c. 线程安全检查清单

- [x] 所有共享缓存使用 `ConcurrentHashMap`
- [x] 缓存写入使用 `putIfAbsent` 避免竞态
- [x] `MessageDigest` 使用 `ThreadLocal` 隔离
- [x] `saltChecked` 使用 `volatile` 保证可见性
- [x] 排序后字段列表不可变
- [x] 无 `synchronized` 块，无死锁风险
- [x] `SignProperties.setSalt()` 在启动阶段单线程执行
- [x] `fieldCache.clear()` 有极小窗口重复构建但不影响正确性

---

## 18. 性能与资源审计（CPU / IO / OOM）

### 18a. CPU 优化

| 优化点 | 措施 | 效果 |
|---|---|---|
| **SHA-256 计算** | `ThreadLocal<MessageDigest>` 复用实例 | 避免每次 `MessageDigest.getInstance()` 开销 |
| **字段排序** | `sortedFieldsCache` 缓存排序结果 | 首次排序后后续请求 O(1) 获取 |
| **驼峰转换** | `camelCaseCache` 缓存转换结果 | 避免重复字符串操作 |
| **反射查找** | `fieldCache` 缓存类字段映射 | 避免每次请求重复反射 |
| **注解解析** | `annotationCache` + `noAnnotationCache` 双缓存 | 正缓存命中直接返回，负缓存命中 O(1) 跳过 |
| **负缓存跳过** | `skipClasses` 记录无 hash 字段的类 | 后续请求秒跳，不重复反射 |
| **假签名生成** | `ThreadLocalRandom` 无锁随机 | 比 `SecureRandom` 快数倍，无锁无竞争 |
| **StringBuilder** | 签名串拼接使用 `StringBuilder` | 避免 `+` 拼接的中间对象 |

### 18b. IO 优化

| 优化点 | 措施 | 效果 |
|---|---|---|
| **AWS Secrets Manager 调用** | 仅启动时调用一次，运行时无 IO | 不影响请求延迟 |
| **SecretsManagerClient 关闭** | `finally` 块中 `client.close()` | 防止连接泄漏 |
| **无数据库额外查询** | 签名基于实体当前字段值计算 | 不产生额外 SQL |
| **无网络调用** | 运行时全部内存操作 | 无网络 IO |

### 18c. OOM 防护

| 风险点 | 防护措施 | 所在代码 |
|---|---|---|
| **fieldCache 无限增长** | 上限 512 个类，超限 `clear()` 重建 + WARN 日志 | `SignMybatisInterceptor.getFields()` |
| **动态类加载场景** | Groovy 脚本等动态生成的类会持续新增缓存项 | 上限 + 清空机制 |
| **sortedFieldsCache 无限增长** | 表数量有限（业务表），实际不会无限增长 | 无额外措施需要 |
| **camelCaseCache 无限增长** | 列名数量有限，实际不会无限增长 | 无额外措施需要 |
| **annotationCache 无限增长** | 实体类数量有限，实际不会无限增长 | 无额外措施需要 |
| **StringBuilder 内存** | 签名串长度 = 字段值总和 + 盐值，单行数据量有上限 | 无额外措施需要 |
| **ThreadLocal MessageDigest** | 每线程一个实例，线程池场景下实例数 = 线程数 | 无泄漏风险（线程复用时 `get()` 返回已有实例） |
| **ObjectMapper 实例** | `static final` 单例 | 避免重复创建 |

### 18d. 性能检查清单

- [x] SHA-256 实例通过 ThreadLocal 复用
- [x] 字段排序结果缓存
- [x] 驼峰转换结果缓存
- [x] 反射结果缓存
- [x] 注解解析结果缓存（正缓存 + 负缓存）
- [x] 无 hash 字段的类负缓存跳过
- [x] AWS 调用仅启动时一次
- [x] SecretsManagerClient 正确关闭
- [x] fieldCache 有上限防 OOM
- [x] ObjectMapper 单例
- [x] 假签名使用 ThreadLocalRandom 无锁生成
- [x] 运行时无额外数据库查询和网络调用

---

## 19. 防御性编码审计

### 19a. 防御性编码原则

**核心原则：签名组件的任何异常都不影响业务执行，不影响其他字段值。**

### 19b. 防御性编码实现

| 场景 | 防护措施 | 代码位置 |
|---|---|---|
| **签名计算异常** | `catch(Throwable)` 包裹整个 `doFill` / `fillEntity` | `AutoSignHandler.insertFill/updateFill`、`SignMybatisInterceptor.fillEntitySafe` |
| **日志调用本身异常** | 内层 `try-catch` 包裹日志代码 | `AutoSignHandler.insertFill/updateFill` catch 块、`SignMybatisInterceptor.fillEntitySafe` catch 块 |
| **拦截器异常** | `catch(Throwable)` + `return invocation.proceed()`（try-catch 后始终执行） | `SignMybatisInterceptor.intercept` |
| **反射获取字段失败** | `catch(Exception)` 返回 null | `SignMybatisInterceptor.getFieldValue`、`SignVerifier.getFieldValueSafe` |
| **设置字段值失败** | `catch(IllegalAccessException)` + WARN 日志 | `SignMybatisInterceptor.setFieldValue` |
| **类型不匹配** | 不写入字段，跳过 | `AutoSignHandler.setTimestampValue`、`SignMybatisInterceptor.setTimestampValue` |
| **AWS 获取盐值失败** | `catch(Exception)` 回退 YAML salt | `AwsSaltInitializer.initGlobalSalt/initTableSalts` |
| **Secret 解析失败** | `catch(Exception)` 返回 null | `AwsSaltInitializer.fetchSalt` |
| **SecretsManagerClient 关闭失败** | `catch(Exception ignored)` | `AwsSaltInitializer.fetchSalt` finally 块 |
| **盐值为 null** | WARN 日志，签名结果不匹配但不抛异常 | `SignVerifier.verifyEntity` |
| **hash 值缺失或长度非 64** | 返回 false（验签失败），不抛异常 | `SignVerifier.verifyEntity/verify` |
| **参数为 null** | 提前返回 false/null，不抛 NPE | `SignVerifier` 所有公开方法 |
| **字段值 null** | 格式化为空字符串 `""` | `formatValue` 方法 |
| **时间戳字段不存在** | `hasProp` / `fields.get` 检查后跳过 | `fillTimestamps`、`appendTimestamp` |
| **批量插入某条异常** | 逐条 `fillEntitySafe` 独立 try-catch | `SignMybatisInterceptor.processParameter` |
| **fieldCache 超限** | `clear()` 重建 + WARN 日志 | `SignMybatisInterceptor.getFields` |

### 19c. 双层 try-catch 详解

**为什么需要双层 try-catch**:

外层 `catch` 块中的日志调用本身可能抛异常：
- `metaObject.getOriginalObject()` 可能为 null → NPE
- `resolveTableName(metaObject)` 内部反射可能失败
- `entity.getClass().getName()` 极端情况下可能失败

如果日志调用抛异常，会"穿透"外层 catch 块，传播到业务层，违反"异常不影响业务"原则。

**解决方案**:

```java
try {
    doFill(metaObject, false);
} catch (Throwable e) {
    try {
        log.error("..., class={}, table={}",
                metaObject.getOriginalObject().getClass().getName(),  // 可能 NPE
                resolveTableName(metaObject));
    } catch (Throwable logEx) {
        log.error("[bitgame-sign] xxx 异常, 日志记录也失败", e);  // 兜底：只输出原始异常
    }
}
```

### 19d. 异常不影响其他字段值

**设计保证**:

1. **签名计算与字段写入分离**: 先计算 hash，再写入。计算失败不会写入部分值
2. **时间戳填充独立**: 每个时间戳字段独立判断和写入，一个失败不影响其他
3. **类型不匹配跳过**: `setTimestampValue` 遇到不支持的类型直接跳过，不写入错误值
4. **反射异常返回 null**: `getFieldValue` 捕获 `IllegalAccessException` 返回 null，不传播
5. **hash 写入是最后一步**: 只有签名计算成功才写入 hash，计算失败 hash 保持原值

### 19e. 防御性编码检查清单

- [x] 所有公开入口 `catch(Throwable)` 包裹
- [x] catch 块内日志调用有内层 try-catch 保护
- [x] 拦截器 `return invocation.proceed()` 在 try-catch 后确保业务继续执行
- [x] 反射异常不传播
- [x] null 参数安全处理
- [x] 类型不匹配跳过不写入
- [x] 批量操作逐条独立异常处理
- [x] AWS 调用失败有回退策略
- [x] 资源（Client）在 finally 中关闭
- [x] 签名计算失败不影响其他字段值

---

## 20. 逻辑与流程完整性审计

### 20a. 启动流程完整性

```
Spring Boot 启动
  │
  ├── 1. spring.factories 扫描 → SignAutoConfiguration
  │     └── 条件检查: MyBatis Plus 存在 + enabled=true
  │
  ├── 2. SignProperties 创建 → 绑定 YAML 配置
  │     └── tables Map、aws 配置、salt、hashFieldName 等
  │
  ├── 3. SignTableResolver 创建 → 持有 properties 引用
  │
  ├── 4. AutoSignHandler 创建 (如无自定义 MetaObjectHandler)
  │     └── 日志: 配置表数、表级 AWS 数
  │
  ├── 5. SignMybatisInterceptor 创建
  │     └── 日志: 拦截器已注册
  │
  ├── 6. AwsSaltInitializer 创建 → afterPropertiesSet()
  │     ├── 6a. initGlobalSalt()
  │     │     ├── aws.isConfigured() == false → 跳过，返回 false
  │     │     ├── fetchSalt() 成功 → properties.setSalt() → 日志 INFO
  │     │     ├── fetchSalt() 返回 null → 日志 WARN
  │     │     └── fetchSalt() 异常 → 日志 ERROR，回退 YAML salt
  │     ├── 6b. initTableSalts()
  │     │     ├── tables 为空 → 返回 0
  │     │     ├── 遍历每表:
  │     │     │     ├── hasAwsConfig() == false → 跳过
  │     │     │     ├── fetchSalt() 成功 → tableConfig.setSalt() → 计数+1
  │     │     │     ├── fetchSalt() 返回 null → 日志 WARN
  │     │     │     └── fetchSalt() 异常 → 日志 ERROR，回退全局盐值
  │     │     └── 返回成功数
  │     └── 6c. 汇总日志
  │
  └── 启动完成，组件就绪
```

**完整性验证**:
- [x] 无 AWS 配置时不产生 IO
- [x] 全局盐值获取失败有回退
- [x] 表级盐值获取失败有回退（回退到全局）
- [x] 启动阶段异常不阻断应用启动（`InitializingBean` 异常会被 Spring 捕获）

### 20b. INSERT 签名流程完整性

```
mapper.insert(entity)
  │
  ├── AutoSignHandler.insertFill(metaObject)
  │     ├── enabled == false → return
  │     ├── try { doFill(metaObject, false) }
  │     │     ├── checkSaltOnce() — 首次盐值检查
  │     │     ├── hasProp(hashProp) == false → return (无签名字段)
  │     │     ├── fillTimestamps(now, isUpdate=false)
  │     │     │     ├── create 字段: null → 填充 now
  │     │     │     ├── update 字段: null → 填充 now
  │     │     │     └── 其他时间字段: null → 填充 now
  │     │     ├── resolve(tableName, clazz)
  │     │     │     ├── YAML 有配置 → 返回 YAML 配置
  │     │     │     ├── 注解有配置 → 返回注解配置
  │     │     │     └── 都没有 → null
  │     │     ├── tableConfig != null → computeRealHash()
  │     │     │     ├── 排序字段 (缓存)
  │     │     │     ├── 拼接: field1|field2|...|ts1|ts2 + salt
  │     │     │     └── SHA-256 → hash
  │     │     ├── tableConfig == null → generateFakeHash()
  │     │     └── metaObject.setValue(hashProp, hash)
  │     └── catch(Throwable) → 日志，不影响业务
  │
  ├── SignMybatisInterceptor.intercept()
  │     ├── enabled == false → proceed()
  │     ├── cmdType != INSERT/UPDATE → proceed()
  │     ├── processParameter(parameter, isUpdate=false)
  │     │     ├── Map 参数 → 检查 et/param1/list
  │     │     ├── Collection 参数 → 遍历
  │     │     └── 实体参数 → fillEntitySafe()
  │     ├── fillEntity()
  │     │     ├── skipClasses 命中 → return
  │     │     ├── 无 hash 字段 → 加入负缓存，return
  │     │     ├── hash 已有 64 位值 → 跳过 (Handler 已处理)
  │     │     ├── fillTimestamps()
  │     │     ├── resolve() → computeRealHash() 或 generateFakeHash()
  │     │     └── setFieldValue(entity, hashField, hash)
  │     └── catch(Throwable) → 日志
  │
  └── try-catch 后 → return invocation.proceed() → SQL 执行
```

**完整性验证**:
- [x] enabled=false 时完全跳过
- [x] 无签名字段的实体跳过
- [x] Handler 已处理的实体拦截器不重复处理
- [x] YAML 和注解配置都能正确解析
- [x] 无配置但有 hash 字段的表生成假签名
- [x] 时间戳 null 时才填充（不覆盖业务设置的值）
- [x] 批量插入逐条独立处理
- [x] 任何异常不影响 SQL 执行

### 20c. UPDATE 签名流程完整性

```
mapper.updateById(entity)
  │
  ├── AutoSignHandler.updateFill(metaObject)
  │     ├── enabled == false → return
  │     ├── try { doFill(metaObject, true) }
  │     │     ├── checkSaltOnce()
  │     │     ├── hasProp(hashProp) == false → return
  │     │     ├── fillTimestamps(now, isUpdate=true)
  │     │     │     ├── create 字段: 不修改
  │     │     │     └── update 字段: 总是刷新为 now
  │     │     ├── resolve(tableName, clazz)
  │     │     ├── isUpdate → 检测签名字段全空 → WARN
  │     │     ├── computeRealHash() 或 generateFakeHash()
  │     │     └── metaObject.setValue(hashProp, hash)
  │     └── catch(Throwable) → 日志
  │
  ├── SignMybatisInterceptor.intercept()
  │     └── 同 INSERT 流程
  │
  └── try-catch 后 → return invocation.proceed() → SQL 执行
```

**完整性验证**:
- [x] UPDATE 时 create 字段不修改
- [x] UPDATE 时 update 字段总是刷新
- [x] 签名字段全空时 WARN 提示（部分更新场景）
- [x] 重新计算 hash 覆盖旧值
- [x] 任何异常不影响 SQL 执行

### 20d. 验签流程完整性

```
SignVerifier.verifyEntity(entity, salt)
  │
  ├── entity == null → return false
  ├── getFieldMetas(clazz)
  │     ├── 无 @SignTable → return null → 验签失败
  │     ├── 无 @SignField → return null → 验签失败
  │     └── 解析成功 → 缓存 + 返回排序后字段列表
  ├── actualHash = getStringFieldValue(entity, clazz, "dataHash")
  │     ├── null → 验签失败 (DEBUG 日志)
  │     └── length != 64 → 验签失败 (DEBUG 日志)
  ├── 拼接签名串: field1|field2|...|ts1|ts2 + salt
  │     ├── salt == null → WARN 日志，不追加盐值
  │     └── 字段值通过反射获取，异常返回 null → ""
  ├── expectedHash = SHA-256(签名串)
  └── SignUtils.verify(expectedHash, actualHash)
        ├── 长度不同 → false
        └── 逐字符 XOR 比较 → 时间恒定
```

**完整性验证**:
- [x] null 实体安全处理
- [x] 无注解配置返回 false
- [x] hash 缺失或长度异常返回 false
- [x] salt 为 null 时 WARN 但不抛异常
- [x] 反射获取字段值异常返回 null
- [x] 时间恒定比较防 timing attack
- [x] 验签失败输出 DEBUG 日志（hash 前 8 位对比）

### 20e. 功能完整性检查清单

- [x] INSERT 自动签名
- [x] UPDATE 自动签名
- [x] 批量插入自动签名
- [x] 自动时间戳填充（INSERT + UPDATE）
- [x] YAML 配置驱动
- [x] 注解配置驱动
- [x] YAML 优先于注解
- [x] 无配置表生成假签名
- [x] 全局盐值支持
- [x] 表级盐值支持
- [x] AWS Secrets Manager 全局盐值
- [x] AWS Secrets Manager 表级盐值
- [x] AWS 获取失败回退
- [x] 实体对象验签
- [x] Map 验签（注解驱动）
- [x] Map 验签（字段列表驱动）
- [x] 存量数据初始化（computeHash）
- [x] debug 日志开关
- [x] 真签名日志始终输出
- [x] 敏感信息脱敏
- [x] 异常不影响业务执行
- [x] 异常不影响其他字段值
- [x] 双层签名（Handler + Interceptor）
- [x] 拦截器去重（已有 64 位 hash 跳过）
- [x] 负缓存快速跳过
- [x] 缓存上限防 OOM

---

## 21. 综合质量保证矩阵

| 质量维度 | 检查项数 | 通过 | 状态 |
|---|---|---|---|
| 安全审计 | 9 | 9 | ✅ 全部通过 |
| 线程安全 | 8 | 8 | ✅ 全部通过 |
| 性能优化 (CPU) | 12 | 12 | ✅ 全部通过 |
| IO 优化 | 4 | 4 | ✅ 全部通过 |
| OOM 防护 | 8 | 8 | ✅ 全部通过 |
| 防御性编码 | 10 | 10 | ✅ 全部通过 |
| 逻辑完整性 | 28 | 28 | ✅ 全部通过 |
| 边界条件 | 27 | 27 | ✅ 全部通过 |
| **合计** | **106** | **106** | **✅** |

---

## 22. 边界条件与异常场景处理

### 22a. 边界条件处理矩阵

| 边界条件 | 处理方式 | 所在代码 |
|---|---|---|
| `enabled=false` | Handler 直接 return；Interceptor 直接 `proceed()` | `AutoSignHandler.insertFill/updateFill`、`SignMybatisInterceptor.intercept` |
| 实体无 `dataHash` 属性 | Handler: `hasProp` 检查后 return；Interceptor: 加入 `skipClasses` 负缓存 | `AutoSignHandler.doFill`、`SignMybatisInterceptor.fillEntity` |
| `signFields` 为 null 或空列表 | 不进入真签名计算，走假签名路径 | `AutoSignHandler.doFill`、`SignMybatisInterceptor.fillEntity` |
| `timestampFields` 为 null | `fillTimestamps` 直接 return，不填充任何时间戳 | `AutoSignHandler.fillTimestamps`、`SignMybatisInterceptor.fillTimestamps` |
| `insertTimestampField` / `updateTimestampField` 为空字符串 | `appendTimestamp` 跳过，不追加到签名串 | `AutoSignHandler.appendTimestamp`、`SignMybatisInterceptor.appendTimestamp` |
| 盐值为空字符串 `""` | `getEffectiveSalt` 返回 `""`，`sb.append("")` 无效果（不追加任何内容），首次 WARN 日志 | `AutoSignHandler.checkSaltOnce`、`computeRealHash` |
| 盐值为 null（极端场景） | `getEffectiveSalt` 返回 null，`sb.append((String)null)` 追加字符串 `"null"`。`SignVerifier` 用 `if (salt != null)` 守卫，null 时不追加。**两者签名串不同，验签会失败**。但 `SignProperties.salt` 默认 `""` 不会为 null，实际不会触发 | `AutoSignHandler.computeRealHash`、`SignVerifier.verifyEntity/computeHash` |
| 字段值为 null | `formatValue(null)` 返回 `""`，拼入签名串 | `formatValue` 方法 |
| 时间戳字段已有值（INSERT） | 不覆盖，仅 null 时填充 | `fillTimestamps` — `existing == null` 检查 |
| 时间戳字段已有值（UPDATE） | update 字段总是刷新，create 字段不修改 | `fillTimestamps` — `isUpdate` 分支 |
| 字段类型不支持时间戳 | `setTimestampValue` 静默跳过，不写入错误值 | `AutoSignHandler.setTimestampValue`、`SignMybatisInterceptor.setTimestampValue` |
| `@TableName` 注解缺失 | 回退到 `camelToSnake(class.getSimpleName())` 推断表名 | `resolveTableName` 方法 |
| YAML 和注解都无配置 | 返回 null → 生成假签名 | `SignTableResolver.resolve` |
| `@SignTable` 存在但无 `@SignField` | WARN 日志，返回 null → 假签名 | `SignTableResolver.doResolveFromAnnotation` |
| 批量插入中某条实体异常 | `fillEntitySafe` 逐条独立 try-catch，其余正常 | `SignMybatisInterceptor.processParameter` |
| hash 已有 64 位值 | Interceptor 跳过，不重复签名 | `SignMybatisInterceptor.fillEntity` |
| hash 有值但长度非 64 | Interceptor 视为未处理，重新计算签名 | `SignMybatisInterceptor.fillEntity` |
| 验签时 entity 为 null | 返回 false | `SignVerifier.verifyEntity` |
| 验签时 hash 为 null 或长度非 64 | 返回 false，DEBUG 日志 | `SignVerifier.verifyEntity/verify` |
| 验签时 salt 为 null | WARN 日志，不追加盐值到签名串（`if (salt != null)` 守卫），结果不匹配但不抛异常 | `SignVerifier.verifyEntity` |
| 验签时反射获取字段失败 | 返回 null → `formatValue(null)` → `""` | `SignVerifier.getFieldValueSafe` |
| AWS 无配置 | 不产生 IO，直接跳过 | `AwsSaltInitializer.initGlobalSalt/initTableSalts` |
| AWS 获取盐值返回 null | WARN 日志，回退 YAML salt | `AwsSaltInitializer.initGlobalSalt` |
| AWS 表级获取失败 | ERROR 日志，回退全局盐值 | `AwsSaltInitializer.initTableSalts` |
| Secret JSON key 不存在 | 返回 null → WARN 日志 | `AwsSaltInitializer.fetchSalt` |
| Secret 非 JSON 格式 | 作为纯文本盐值返回 | `AwsSaltInitializer.fetchSalt` |
| `fieldCache` 达 512 上限 | `clear()` 清空 + WARN 日志，重建 | `SignMybatisInterceptor.getFields` |

### 22b. 时间戳填充决策表

| 条件 | `create` 字段 | `update` 字段 | 其他时间字段 |
|---|---|---|---|
| INSERT + 字段为 null | ✅ 填充 now | ✅ 填充 now | ✅ 填充 now |
| INSERT + 字段有值 | ❌ 不覆盖 | ❌ 不覆盖 | ❌ 不覆盖 |
| UPDATE + 字段为 null | ❌ 不修改 | ✅ 总是刷新 now | ❌ 不修改 |
| UPDATE + 字段有值 | ❌ 不修改 | ✅ 总是刷新 now | ❌ 不修改 |

> `create` / `update` 判断依据：列名包含 `"create"` 或 `"update"` 子串（大小写敏感）。

### 22c. 签名路径决策树

```
INSERT/UPDATE 请求
  │
  ├── enabled == false?
  │     └── YES → 跳过签名，正常执行 SQL
  │
  ├── 实体无 hashFieldName 属性?
  │     └── YES → 跳过（Interceptor 加入负缓存）
  │
  ├── hash 已有 64 位值? (仅 Interceptor)
  │     └── YES → 跳过（Handler 已处理）
  │
  ├── resolver.resolve(tableName, clazz)
  │     ├── YAML 有配置 → 真签名
  │     ├── 注解有配置 → 真签名
  │     └── 都没有 → 假签名
  │
  └── 写入 hash → SQL 执行
```

---

## 23. 常见问题 FAQ

### Q1: 为什么 `@Value("${bitgame.sign.salt}")` 拿不到 AWS 获取的盐值？

`AwsSaltInitializer` 通过 `properties.setSalt()` 更新 `SignProperties` Bean 的内存值，不更新 Spring Environment。`@Value` 在 Bean 创建时从 Environment 读取一次，之后不会更新。**正确做法**：注入 `SignProperties` Bean，调用 `getSalt()` 或 `getEffectiveSalt()`。

### Q2: 业务方已有自定义 `MetaObjectHandler`，签名还能生效吗？

`AutoSignHandler` 有 `@ConditionalOnMissingBean(MetaObjectHandler.class)` 条件，不会注册。但 `SignMybatisInterceptor` 无此条件，作为兜底仍然生效。拦截器会检查 hash 是否已有 64 位值，如果自定义 Handler 未填充签名，拦截器会计算并写入。

### Q3: 使用 Wrapper 部分更新（`UpdateWrapper.set("amount", 100)`）会签名吗？

`MetaObjectHandler` 不会触发（Wrapper update 不走 `updateFill`）。但 `SignMybatisInterceptor` 会拦截 `Executor.update`，检测到实体无 hash 字段或 hash 值缺失时尝试填充。**但 Wrapper update 不传入完整实体**，签名字段可能全为空，导致 WARN 日志和基于空值的签名。**建议**：始终使用 `selectById` → 修改 → `updateById` 的完整实体更新方式。

### Q4: 假签名有什么作用？

不在签名配置中但数据库表有 `data_hash` 列的实体，组件会生成 64 位随机 hex 作为假签名。格式与真 SHA-256 完全一致，攻击者无法通过 hash 特征判断哪些表有真签名保护。

### Q5: `debug=true` 应该什么时候开启？

- **上线初期**: 开启观察签名行为是否正确
- **排查问题**: 验签失败、签名不匹配时开启查看详细日志
- **正常运行**: 关闭，避免假签名和 Resolver 日志量过大

`debug` 可通过 Spring Cloud Config 等热更新，无需重启。

### Q6: 验签失败但日志没有输出？

验签失败日志是 DEBUG 级别，需要两个条件同时满足：
1. `bitgame.sign.debug=true`（组件 debug 开关）
2. 日志框架级别允许 DEBUG 输出（如 `logback.xml` 中 `com.bitgame` 包设为 DEBUG）

### Q7: `fieldCache` 清空重建会影响正在运行的请求吗？

`clear()` 和后续 `putIfAbsent` 之间有极小时间窗口，此时其他线程可能重复构建字段映射。但 `putIfAbsent` 保证不会覆盖已有值，最终一致性正确。重复构建仅产生少量额外 CPU 开销，不影响业务正确性。

### Q8: 多表共用一个实体类，签名配置会冲突吗？

不会。签名配置以**表名**为 key（YAML）或以**类**为 key（注解缓存）。如果多表共用一个实体类且使用 YAML 配置，每张表有独立的 `SignTableConfig`。如果使用注解配置，同一个类的注解解析结果会被缓存，多表共用相同的字段配置。

### Q9: `SignVerifier` 是静态方法，不需要注入 Spring Bean 吗？

`SignVerifier` 是纯静态工具类，不需要 Spring 管理。但验签所需的**盐值**需要从 `SignProperties` Bean 获取（通过注入），不能硬编码。

### Q10: 存量数据如何批量初始化签名？

```java
@Autowired
private SignProperties signProperties;

public void initExistingData() {
    SignTableConfig tableConfig = signProperties.getTables().get("fund_withdrawl_order");
    String salt = tableConfig.getEffectiveSalt(signProperties.getSalt());

    // 分页查询未签名记录（data_hash IS NULL）
    List<FundWithdrawlOrder> records = mapper.selectList(
        new LambdaQueryWrapper<FundWithdrawlOrder>()
            .isNull(FundWithdrawlOrder::getDataHash)
            .last("LIMIT 1000"));

    for (FundWithdrawlOrder record : records) {
        // 使用 SignVerifier.computeHash 计算签名
        // 注意：computeHash 接受 Map<String, String>，需要将实体转为列名→值字符串的 Map
        // 或直接使用 verifyEntity 验证已有数据是否正确
    }
}
```

> **注意**: `computeHash` 接受 `Map<String, String>` 参数（列名→值字符串），不接受实体对象。存量初始化时需要将实体字段值转为 Map 格式，确保与签名时的 `formatValue` 规则一致（特别是 `BigDecimal.toPlainString()` 和时间格式 `yyyy-MM-dd HH:mm:ss`）。

---

## 24. 构建与发布

### 24a. Maven 构建

```bash
mvn clean install
```

**构建产物**:
- `bitgame-sign-spring-boot-starter-1.0.0-SNAPSHOT.jar` — 组件 JAR
- `bitgame-sign-spring-boot-starter-1.0.0-SNAPSHOT-sources.jar` — 源码 JAR（`maven-source-plugin`）

### 24b. pom.xml 关键配置

| 配置 | 值 | 说明 |
|---|---|---|
| `groupId` | `com.bitgame` | 组织标识 |
| `artifactId` | `bitgame-sign-spring-boot-starter` | 组件标识 |
| `version` | `1.0.0-SNAPSHOT` | 版本号 |
| `packaging` | `jar` | 打包方式 |
| `java.version` | `1.8` | 编译目标 |
| `spring-boot.version` | `2.5.15` | Spring Boot 版本 |
| `mybatis-plus.version` | `3.5.3.1` | MyBatis Plus 版本 |
| `maven-source-plugin` | `3.2.1` | 源码 JAR 插件 |

### 24c. 依赖 scope 策略

| 依赖 | scope | 原因 |
|---|---|---|
| `spring-boot-starter` | provided | 业务项目提供 |
| `spring-boot-autoconfigure` | provided | 业务项目提供 |
| `spring-boot-configuration-processor` | optional | 编译期元数据，运行时不需要 |
| `mybatis-plus-boot-starter` | provided | 业务项目提供 |
| `slf4j-api` | provided | 业务项目提供 |
| `jackson-databind` | provided | 业务项目提供 |
| `aws-secretsmanager` | compile | 传递给业务方，无需单独引入 |

> **设计意图**: 除 AWS SDK 外全部 provided，避免与业务项目的版本冲突。AWS SDK 设为 compile 是因为业务方通常不直接依赖 AWS Secrets Manager SDK。
