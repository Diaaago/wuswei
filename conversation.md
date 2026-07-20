# objective-batch 迁移对话记录

> 本文档整理自用户与 Codex 的可见对话。系统/开发者指令、内部推理、运行环境注入信息未包含；本机绝对路径已脱敏；聊天中的图片附件未上传。

## 用户

我需要你根据<本机迁移指南目录>里面的从11.1.0到18.0.0的migration guide帮我把objective-batch的现有所有代码（src，resource等等）migrate到pfspring 18需要做的事情，改的代码，参考的哪里全部告诉我，不要直接更改。另外，不要局限于migration guide，其中提到的官方升级文档也一定要过目参考。其中的pom文件里面除了要把parent改成18.0.5以外还需要更改jcloud bom的版本，详细情况可以参考partner-batch或者out-batch的pom。只做迁移必须的更改，其他非必须的不做，非必要不迁移，迁移和不迁移都要说明原因和出处。

## 助手

我会按“只审查、不改文件”的方式做：先完整梳理 `objective-batch` 的源码、resources、测试和 POM，再逐份读取 11.1.0→18.0.0 的 migration guide 及其中引用的官方升级文档，并用 `partner-batch` / `out-batch` 的 POM 交叉核对 parent 与 jcloud BOM。最后会给出逐项“必须迁移 / 不迁移”的结论、具体建议代码、影响文件、原因与出处。

## 助手

初步盘点发现一个需要特别标注的事实：当前 `<本机工作区>\objective-batch` 下只有 `src` 和 IDE 元数据，没有 `pom.xml`、Dockerfile 或测试目录。因此我可以精确审查现有源码/resources，并给出应有的 POM 迁移内容，但无法对“objective-batch 当前 POM 的每一行”做差异核验；我会继续搜索工作区及 IDE 索引，确认是否有遗漏副本，同时以两个对照项目的 POM 为依据。全程不会写入任何项目文件。

## 助手

本地 guides 已完整覆盖到 18.0.0。接下来我会按 18.0.0 最后一节列出的官方资料逐一核对（Spring Boot 4、Framework 7、Data 2025.1、Batch 6、Security 7、Cloud 2025.1、Hibernate 7.x），重点不是泛读，而是确认 `objective-batch` 实际命中的包迁移、Batch API、数据库元数据和配置属性；未命中的条目也会明确写出“不迁移”的代码证据。

## 助手

我已经完成了源码、资源、迁移指南和官方文档的交叉核对。最后我会再核实几个本地文件的行号和“缺失文件”事实，然后给你一份按“必须改 / 条件性改 / 明确不改”分类的迁移清单；不会写入任何文件。

## 助手

未修改任何文件。结论是：现有 `objective-batch` 迁到 PF Spring 18.0.5，确定需要处理的是 POM、Spring Batch 6 包名、`RunIdIncrementer`、Batch 元数据、readiness 配置以及 Java/Maven 运行环境；业务逻辑、JPA 实体和 Repository 基本不用改。

不过当前目录只有 `.idea` 和 `src`，没有 `pom.xml`、Dockerfile、`.gitlab-ci.yml`、启动脚本和测试。因此 POM/部署部分只能给出准确目标，无法对原文件逐行审计。

## 1. 迁移前必须确认

- 如果应用运行在 WebSphere Liberty 上，不应升级到 PF Spring 18，因为 Spring Boot 4 使用 Servlet 6.1，PF 18 指南明确给出了这个限制：[18.0.0 guide](</<本机迁移指南目录>/18.0.0.md:5>)。
- Java 最低必须是 21：[PF 18 guide](</<本机迁移指南目录>/18.0.0.md:63>)。当前 IDEA 已配置 `corretto-21`：[misc.xml](/<本机工作区>/objective-batch/.idea/misc.xml:3)，但仍需确认 CI、Docker 和服务器 JVM。
- Maven 最低必须是 3.8.5：[17.0.1 guide](</<本机迁移指南目录>/17.0.1.md:5>)。IDE 配置指向 Maven 3.9.6：[workspace.xml](/<本机工作区>/objective-batch/.idea/workspace.xml:16)，本地满足要求，但 CI 未提供。

Java 25只是指南推荐，不是最低要求，因此按照“只做必需迁移”，不需要专门升到 Java 25。

## 2. POM 必须修改

`objective-batch` 当前没有 POM。恢复或取得真实 POM 后，至少需要以下内容。

### Parent

```xml
<parent>
    <groupId>com.bnpp.pf.pfspring</groupId>
    <artifactId>pf-spring-parent</artifactId>
    <version>18.0.5</version>
</parent>
```

参考：

- [partner-batch/pom.xml](/<本机工作区>/partner-batch/pom.xml:7)
- [out-batch/pom.xml](/<本机工作区>/out-batch/pom.xml:7)
- [PF 18 parent migration](</<本机迁移指南目录>/18.0.0.md:9>)

### JCloud BOM

版本必须调整为：

```xml
<properties>
    <dependency.jcloud-bom.version>
        2.0.4-pf-spring-as-parent.4
    </dependency.jcloud-bom.version>
</properties>
```

并确认已有以下导入；如果已经存在，只更新版本属性，不要重复添加：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.bnpp.pf.jcloud</groupId>
            <artifactId>jcloud-bom</artifactId>
            <version>${dependency.jcloud-bom.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

参考：

- [partner-batch JCloud 版本](/<本机工作区>/partner-batch/pom.xml:26)和[导入](/<本机工作区>/partner-batch/pom.xml:43)
- [out-batch JCloud 版本](/<本机工作区>/out-batch/pom.xml:26)和[导入](/<本机工作区>/out-batch/pom.xml:39)

不要复制 partner/out POM 中的其他依赖、插件或 exclusions；它们不一定适用于 objective-batch。

### 需要在真实 POM 中条件检查

- 如果已经使用 `pf-spring-starter-batch`，保留它。
- 如果直接使用 Spring Boot 4 的 `spring-boot-starter-batch`，而不是 PF starter，则应改用 `spring-boot-starter-batch-jdbc`。Boot 4 中普通 Batch starter 面向内存存储，而本项目明确使用持久化 Spring Batch 元数据。[Spring Boot 4 migration guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide)
- 如果直接依赖 `liquibase-core`，需要结合 PF starter 检查是否应由 `spring-boot-starter-liquibase` 提供。没有真实 POM，不能直接判定要加。
- 不要添加 Querydsl/OpenFeign Querydsl。现有代码没有 `com.querydsl`、Q 类、Querydsl Repository 或 annotation processor 使用。
- 父 POM管理的 Lombok、Hibernate、Spring Cloud 等版本不要再自行覆盖；只有真实 POM中存在不受 PF 管理的依赖时，才检查其 Java 21/Boot 4 兼容性。[PF 18 guide](</<本机迁移指南目录>/18.0.0.md:703>)

## 3. Java 源码必须修改

Spring Batch 6 把基础设施 API 移到了 `org.springframework.batch.infrastructure`，并调整了 `Job`、`Step` 包名。[PF 18 package mapping](</<本机迁移指南目录>/18.0.0.md:247>)、[Spring Batch 6 official migration guide](https://github.com/spring-projects/spring-batch/wiki/Spring-Batch-6.0-Migration-Guide)。

### `MonthGoalsBatchConfig.java`

文件：[MonthGoalsBatchConfig.java](/<本机工作区>/objective-batch/src/main/java/com/bnpp/pf/swm/objective/batch/config/MonthGoalsBatchConfig.java:8)

修改 import：

```java
// before
import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.item.data.RepositoryItemReader;

// after
import org.springframework.batch.core.job.Job;
import org.springframework.batch.core.step.Step;
import org.springframework.batch.infrastructure.item.data.RepositoryItemReader;
```

以下 import 删除：

```java
import org.springframework.batch.core.launch.support.RunIdIncrementer;
```

`JobRepository` 继续使用：

```java
import org.springframework.batch.core.repository.JobRepository;
```

两个 Job 定义都删除 incrementer：

```java
// 删除
.incrementer(new RunIdIncrementer())
```

对应当前约第 75、124 行。PF 18/Spring Batch 6 中，Job 上配置 incrementer 后，运行时传入的参数可能被忽略，并影响失败任务重启，因此这是必须修改项：[PF 18 guide](</<本机迁移指南目录>/18.0.0.md:350>)。

### 两个 Processor

文件：

- [ObjectiveProcessor.java](/<本机工作区>/objective-batch/src/main/java/com/bnpp/pf/swm/objective/batch/processor/ObjectiveProcessor.java:6)
- [PartnerMdsProcessor.java](/<本机工作区>/objective-batch/src/main/java/com/bnpp/pf/swm/objective/batch/processor/PartnerMdsProcessor.java:6)

修改为：

```java
import org.springframework.batch.infrastructure.item.ItemProcessor;
```

### 两个 Writer

文件：

- [ObjectiveWriter.java](/<本机工作区>/objective-batch/src/main/java/com/bnpp/pf/swm/objective/batch/writer/ObjectiveWriter.java:3)
- [PartnerMdsWriter.java](/<本机工作区>/objective-batch/src/main/java/com/bnpp/pf/swm/objective/batch/writer/PartnerMdsWriter.java:3)

修改为：

```java
import org.springframework.batch.infrastructure.item.Chunk;
import org.springframework.batch.infrastructure.item.ItemWriter;
```

### `.chunk(20, transactionManager)` 不迁移

当前两个 step 使用：

```java
.chunk(20, myTransactionManager)
```

PF 18 指南介绍了新的 chunk model，但它是“切换到新模型”的选择；Spring Batch 6 官方说明旧 overload 在整个 6.x 仍保留，只是 deprecated。因此本次只做必须迁移时应保持现状，不改成新模型：[PF 18 guide](</<本机迁移指南目录>/18.0.0.md:460>)。

## 4. 启动参数必须配合修改

删除 `RunIdIncrementer` 后，实际批处理启动器应在用户参数前生成一次性 `run.id`：

```bash
ARGUMENTS+=" run.id=$(date +%s%3N),java.lang.Long"
ARGUMENTS+=" $*"
```

用户显式传入的 `run.id` 放在后面，可用于重启指定 JobInstance。当前仓库没有启动脚本、调度配置或 `.gitlab-ci.yml`，因此这项要在实际 Shell、scheduler 或部署仓库修改，不能在现有源码中完成。

## 5. Spring Batch 数据库必须迁移

现有配置通过 [schema_spring_batch_metadata.yaml](/<本机工作区>/objective-batch/src/main/resources/db/changelog/v1/model/schema_spring_batch_metadata.yaml:1) 加载 Spring Batch PostgreSQL schema。

需要分两种情况：

- 全新数据库：升级后使用 Spring Batch 6 自带的 `schema-postgresql.sql` 即可，不需要修改这个文件。
- 已存在 Spring Batch 5 元数据：上线 PF 18 前必须执行 Batch 6 migration changelog。标准 PostgreSQL 迁移的核心变更是：

```sql
ALTER SEQUENCE BATCH_JOB_SEQ
RENAME TO BATCH_JOB_INSTANCE_SEQ;
```

出处：[PF 18 Batch DB migration](</<本机迁移指南目录>/18.0.0.md:342>)、[Spring Batch 官方 PostgreSQL migration SQL](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-core/src/main/resources/org/springframework/batch/core/migration/6.0/migration-postgresql.sql)。

PF 指南指定的文件是：

```text
changelog-spring-batch-6-0-migration.yaml
```

如果由应用 Liquibase 执行，把该 PF changelog 放入 changelog 目录并由 [db.changelog-master.yaml](/<本机工作区>/objective-batch/src/main/resources/db/changelog/db.changelog-master.yaml:1) include；如果由 DBA 执行，则资源文件可以不改。

当前 master 文件只有 `databaseChangeLog:`，没有 include 任何 changelog。这是现有配置缺口，但无法确认生产是否由外部 DBA 管理，所以不能直接认定必须把 v1/schema 也重新 include。尤其已有数据库不能重新执行初始化 schema。

此外，在切换 Spring Batch 6 前，所有 Spring Batch 5 中失败且将来还需要 restart 的 execution 必须先成功重启，或者明确 abandon。Batch 6 改变了 JobParameter 的持久化格式，旧失败实例不能直接跨版本重启。

## 6. readiness 配置必须清理

以下三个文件都含有旧 workaround：

- [application-dev.yaml](/<本机工作区>/objective-batch/src/main/resources/application-dev.yaml:42)
- [application-pprod.yaml](/<本机工作区>/objective-batch/src/main/resources/application-pprod.yaml:41)
- [application-prod.yaml](/<本机工作区>/objective-batch/src/main/resources/application-prod.yaml:41)

当前内容：

```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          exclude: "readinessState"
```

PF 17 起：

- 标准 `readiness` 只包含 `readinessState`。
- Batch 使用单独的 `readiness-batch`，默认包含除 `readinessState` 以外的 health indicators。

因此此处应删除整个 block。它现在等于从只包含 `readinessState` 的标准 readiness 中再排除 `readinessState`，会使标准 readiness 失去意义。

只有原配置还包含其他确实用于 batch diagnostics 的自定义 include/exclude 时，才应把那些自定义项迁到 `readiness-batch`；当前没有这种配置。[PF 17 readiness guide](</<本机迁移指南目录>/17.0.0.md:23>)。

## 7. `.gitignore`

PF 14 指南要求添加：

```gitignore
**/.flattened-pom.xml
```

当前 `objective-batch` 根目录没有 `.gitignore`，因此应创建或在上级仓库的 ignore 中确认已有该规则。这不影响运行，但属于 PF parent flatten plugin 引入后的迁移要求：[14.0.0 guide](</<本机迁移指南目录>/14.0.0.md:5>)。

## 8. 条件性部署变更

PF 16 的 ECS 日志迁移取决于部署平台：

- PaaS：如果仍使用旧 JSON/logstash Logback 配置，改用 `logging.structured.format.console: ecs`。
- IAAS batch：需要修改外部 RPM/Gclweb Logback 配置。

现有 YAML 中的 `logging.config` 本身不需要因为 Boot 4 而删除，Boot 4 仍支持该属性。[Spring Boot logging documentation](https://docs.spring.io/spring-boot/4.0/how-to/logging.html)。但仓库里没有 `logback.xml`、`logback_local.xml` 或部署日志配置，无法判断 ECS 迁移是否已在外部完成。[PF 16 guide](</<本机迁移指南目录>/16.0.0.md:5>)。

## 9. 明确不需要迁移的内容

- 实体已经全部使用 `jakarta.persistence`，无需再次做 `javax`→`jakarta`：[PF 13 guide](</<本机迁移指南目录>/13.0.0.md:23>)。
- `ObjectiveBatchApplication` 不需要修改。
- Repository 使用普通 Spring Data JPA、`JpaSpecificationExecutor` 和标准 JPQL，没有命中 Spring Data 2025.1 的破坏性 API：[Spring Data 2025.1 notes](https://github.com/spring-projects/spring-data-commons/wiki/Spring-Data-2025.1-Release-Notes)。
- 实体只使用标准 JPA 和 Hibernate `@UpdateTimestamp`，没有自定义 Hibernate type、旧 generator API、Envers 特殊配置等，不需要针对 Hibernate 7 改源码：[Hibernate 7.0 migration guide](https://docs.hibernate.org/orm/7.0/migration-guide/)。
- 没有 Querydsl，不添加 OpenFeign Querydsl 依赖或 processor。
- 没有 Security/OAuth2 配置，不需要 Spring Security 7 迁移：[Spring Security 7 migration](https://docs.spring.io/spring-security/reference/7.0/migration/index.html)。
- 没有 Jackson 配置、REST client、Swagger、SOAP、SSL、proxy、`TestRestTemplate`、`MockBean`、`JobLauncherTestUtils` 或 Batch JSON marshaller，对应指南章节不适用。
- 没有 WAR 构建信息，不添加 `pf-spring-bom-war`；该 BOM 仅用于 WAR/Liberty 模块。
- 当前 YAML 没有命中 Spring Boot 4 配置 changelog 中已删除或更名的属性，因此除 readiness 外不做机械式配置改名：[Boot 4 configuration changelog](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Configuration-Changelog)。
- `spring.jpa.database-platform: org.hibernate.dialect.PostgreSQLDialect`、`logging.config`、Liquibase 和 Batch schema 初始化属性在 Boot 4 中仍受支持，不做非必要清理。
- 不为“顺便现代化”而修改业务查询、chunk size、命名、日志级别、事务结构或代码风格。

最终需要落地的最小变更集合是：POM parent/JCloud BOM、5 个 Java 文件的 Batch 6 import、移除两个 `RunIdIncrementer`、启动器补 `run.id`、已有 Batch 数据库升级、删除三个旧 readiness block，以及添加 flattened POM ignore。POM和部署文件缺失是目前唯一无法做到逐行确认的部分。


## 用户

image 1是现在的batchconfig代码 但是报错如图二

> [图片附件 × 2，未上传]

## 助手

我先把图二里的四个编译错误逐一对应到 Spring Batch 6 的 `RepositoryItemReader` API，再核对你当前工作区中的代码和 PF 18 migration guide。重点看起来不是泛型本身，而是 `RepositoryItemReader` 在 Batch 6 中改成了构造器/不可变配置方式，原来的 setter 已不存在。整个过程只读，不改文件。

## 助手

图二已经能精确解释：`new RepositoryItemReader<>()` 的无参构造失败，随后 `setSort` 也不存在；这是 Batch 6“必填依赖必须在构造时提供”的 API 变化。为了避免给你一个“看起来能编译但调用签名不对”的版本，我再核对两个 repository 的方法签名（最后一个参数必须是 `Pageable`）以及 Batch 6 构造器的真实参数顺序。

## 助手

这个报错原因已经确定：Spring Batch 6 删除了 `RepositoryItemReader` 的无参构造器，`repository` 和 `sorts` 必须通过构造器提供；同时 `setSort(...)` 已变成 `setSorts(...)`。

这确实是我上一轮迁移报告遗漏的一项必须变更。Spring Batch 6 官方迁移文档明确说明：Batch artifact 不能再使用“无参构造器 + setters”的旧方式创建。[官方 Spring Batch 6 migration guide](https://github.com/spring-projects/spring-batch/wiki/Spring-Batch-6.0-Migration-Guide)

## 图二四个错误的对应关系

- 第 34、76 行：`new RepositoryItemReader<>()`
  - Batch 6 无参构造器已不存在。
  - 新构造器是 `RepositoryItemReader(repository, sorts)`。
- 第 45、87 行：`setSort(...)`
  - 方法名已经变成 `setSorts(...)`。
  - 但这里无需再调用 setter，因为 `sorts` 已经传给构造器。

官方 Batch 6 API：[RepositoryItemReader](https://docs.spring.io/spring-batch/reference/api/org/springframework/batch/infrastructure/item/data/RepositoryItemReader.html)

## 两个 Reader 应改成下面这样

只展示需要替换的方法，不修改其他代码。

```java
@Bean
public RepositoryItemReader<ObjectiveEntity> reader(
        ObjectiveRepository objectiveRepository) {

    var lastMonth = LocalDate.now().minusMonths(1);

    List<Integer> arguments = new ArrayList<>();
    arguments.add(lastMonth.getMonthValue());
    arguments.add(lastMonth.getYear());

    HashMap<String, Direction> sorts = new HashMap<>();
    sorts.put("id", Direction.DESC);

    RepositoryItemReader<ObjectiveEntity> objectiveQuery =
            new RepositoryItemReader<>(objectiveRepository, sorts);

    objectiveQuery.setMethodName("findLastMonth");
    objectiveQuery.setArguments(arguments);
    objectiveQuery.setPageSize(40);

    return objectiveQuery;
}
```

第二个 Reader：

```java
@Bean
public RepositoryItemReader<PartnerMdsEntity> readerPartnerMds(
        PartnerMdsRepository partnerMdsRepository) {

    var lastMonth = LocalDate.now().minusMonths(1);

    List<Integer> arguments = new ArrayList<>();
    arguments.add(lastMonth.getMonthValue());
    arguments.add(lastMonth.getYear());

    HashMap<String, Direction> sorts = new HashMap<>();
    sorts.put("mdsId", Direction.DESC);

    RepositoryItemReader<PartnerMdsEntity> partnerMdsQuery =
            new RepositoryItemReader<>(partnerMdsRepository, sorts);

    partnerMdsQuery.setMethodName("findLastMonth");
    partnerMdsQuery.setArguments(arguments);
    partnerMdsQuery.setPageSize(40);

    return partnerMdsQuery;
}
```

也就是说：

```java
new RepositoryItemReader<>()
```

改成：

```java
new RepositoryItemReader<>(repository, sorts)
```

并删除：

```java
reader.setRepository(repository);
reader.setSort(sorts);
```

其余三个 setter：

```java
setMethodName(...)
setArguments(...)
setPageSize(...)
```

在 Batch 6 中仍然存在，可以保留。

## Repository 方法参数也应修改

Spring Batch 6 API明确要求被调用的 Repository 方法最后一个参数是 `Pageable`。当前两个 Repository 使用的是具体实现类 `PageRequest`：

- [ObjectiveRepository.java](/<本机工作区>/objective-batch/src/main/java/com/bnpp/pf/swm/objective/batch/repository/ObjectiveRepository.java:18)
- [PartnerMdsRepository.java](/<本机工作区>/objective-batch/src/main/java/com/bnpp/pf/swm/objective/batch/repository/PartnerMdsRepository.java:32)

import 改为：

```java
import org.springframework.data.domain.Pageable;
```

方法签名改为：

```java
Page<ObjectiveEntity> findLastMonth(
        int month,
        int year,
        Pageable pageable);
```

以及：

```java
Page<PartnerMdsEntity> findLastMonth(
        int month,
        int year,
        Pageable pageable);
```

这不是图二当前四个编译错误的直接原因，但属于 `RepositoryItemReader` 的正式调用契约。继续使用 `PageRequest` 会依赖内部实现细节，迁移时应该一起修正。

`pageSize(40)` 和 `.chunk(20, transactionManager)` 可以保持现状，它们不是这次报错原因。上述变更修正后，图二显示的四个 `RepositoryItemReader` 编译错误应全部消失。

## 用户

重新看看代码

> [图片附件 × 2，未上传]

## 助手

你说得对，我重新按照片里的实际代码顺序看。原代码是先 `new RepositoryItemReader<>()`，最后才创建 `sorts`；所以不能只把构造器那一行机械改掉，必须先把 `sorts` 提到构造器之前。下面我会按你照片中的变量名给出最小修改。

## 助手

重新看后，照片里的关键点是：`sorts` 在方法后半段才创建。因此需要先创建 `sorts`，再调用 Batch 6 的双参数构造器。当前照片里的写法确实还是 Batch 5 写法。

## 第一个 Reader 的最小修改

```java
@Bean
public RepositoryItemReader<ObjectiveEntity> reader(
        ObjectiveRepository objectiveRepository) {

    var lastMonth = LocalDate.now().minusMonths(1);

    List<Integer> list = new ArrayList<>();
    list.add(lastMonth.getMonthValue());
    list.add(lastMonth.getYear());

    // 必须放在 RepositoryItemReader 构造器之前
    HashMap<String, Direction> sorts = new HashMap<>();
    sorts.put("id", Direction.DESC);

    // Spring Batch 6 构造器必须接收 repository 和 sorts
    RepositoryItemReader<ObjectiveEntity> objectiveQuery =
            new RepositoryItemReader<>(objectiveRepository, sorts);

    objectiveQuery.setMethodName("findLastMonth");
    objectiveQuery.setArguments(list);
    objectiveQuery.setPageSize(40);

    return objectiveQuery;
}
```

也就是删除照片中的：

```java
RepositoryItemReader<ObjectiveEntity> objectiveQuery =
        new RepositoryItemReader<>();

objectiveQuery.setRepository(objectiveRepository);

objectiveQuery.setSort(sorts);
```

`setSort` 不能简单改成 `setSorts`，因为无参构造器本身也已经不存在；`repository` 和 `sorts` 必须传给构造器。

## 第二个 Reader 的最小修改

```java
@Bean
public RepositoryItemReader<PartnerMdsEntity> readerPartnerMds(
        PartnerMdsRepository partnerMdsRepository) {

    var lastMonth = LocalDate.now().minusMonths(1);

    List<Integer> list = new ArrayList<>();
    list.add(lastMonth.getMonthValue());
    list.add(lastMonth.getYear());

    HashMap<String, Direction> sorts = new HashMap<>();
    sorts.put("mdsId", Direction.DESC);

    RepositoryItemReader<PartnerMdsEntity> partnerMdsQuery =
            new RepositoryItemReader<>(partnerMdsRepository, sorts);

    partnerMdsQuery.setMethodName("findLastMonth");
    partnerMdsQuery.setArguments(list);
    partnerMdsQuery.setPageSize(40);

    return partnerMdsQuery;
}
```

删除照片中的：

```java
RepositoryItemReader<PartnerMdsEntity> partnerMdsQuery =
        new RepositoryItemReader<>();

partnerMdsQuery.setRepository(partnerMdsRepository);

partnerMdsQuery.setSort(sorts);
```

最终变化只有三点：

1. 把 `sorts` 创建代码提前。
2. 使用 `new RepositoryItemReader<>(repository, sorts)`。
3. 删除 `setRepository(...)` 和 `setSort(...)`。

这是 Spring Batch 6 的真实构造器签名：[RepositoryItemReader API](https://docs.spring.io/spring-batch/reference/api/org/springframework/batch/infrastructure/item/data/RepositoryItemReader.html)。

照片二中的 `RunIdIncrementer` 已经注释掉，这部分方向正确；本次四个编译错误与 Job 和 Step 配置无关。未修改你的文件。

## 用户

好的，这个已经解决了。我要继续进行迁移还要干什么

## 助手

`RepositoryItemReader` 解决后，继续按下面顺序迁移即可。

## 1. 完成剩余 Java API 迁移

### BatchConfig

确认使用：

```java
import org.springframework.batch.core.job.Job;
import org.springframework.batch.core.step.Step;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.infrastructure.item.data.RepositoryItemReader;
```

确保两个 Job 都不再实际调用：

```java
.incrementer(new RunIdIncrementer())
```

相关 import 也删除：

```java
import org.springframework.batch.core.launch.support.RunIdIncrementer;
```

照片中已经注释掉 incrementer，运行行为已经移除；正式提交时建议直接删掉注释行。

出处：[PF 18 Spring Batch 迁移](/<本机迁移指南目录>/18.0.0.md:247)。

### Processor

两个 Processor 的 import 改为：

```java
import org.springframework.batch.infrastructure.item.ItemProcessor;
```

文件：

- [ObjectiveProcessor.java](/<本机工作区>/objective-batch/src/main/java/com/bnpp/pf/swm/objective/batch/processor/ObjectiveProcessor.java:6)
- [PartnerMdsProcessor.java](/<本机工作区>/objective-batch/src/main/java/com/bnpp/pf/swm/objective/batch/processor/PartnerMdsProcessor.java:6)

### Writer

两个 Writer 改为：

```java
import org.springframework.batch.infrastructure.item.Chunk;
import org.springframework.batch.infrastructure.item.ItemWriter;
```

文件：

- [ObjectiveWriter.java](/<本机工作区>/objective-batch/src/main/java/com/bnpp/pf/swm/objective/batch/writer/ObjectiveWriter.java:3)
- [PartnerMdsWriter.java](/<本机工作区>/objective-batch/src/main/java/com/bnpp/pf/swm/objective/batch/writer/PartnerMdsWriter.java:3)

### Repository 分页参数

两个 `findLastMonth` 的最后一个参数改为接口类型：

```java
import org.springframework.data.domain.Pageable;
```

```java
Page<ObjectiveEntity> findLastMonth(
        int month,
        int year,
        Pageable pageable);
```

```java
Page<PartnerMdsEntity> findLastMonth(
        int month,
        int year,
        Pageable pageable);
```

删除：

```java
import org.springframework.data.domain.PageRequest;
```

这是 `RepositoryItemReader` 官方要求的调用签名。

## 2. 完成 POM 迁移

Parent：

```xml
<parent>
    <groupId>com.bnpp.pf.pfspring</groupId>
    <artifactId>pf-spring-parent</artifactId>
    <version>18.0.5</version>
</parent>
```

JCloud BOM：

```xml
<dependency.jcloud-bom.version>
    2.0.4-pf-spring-as-parent.4
</dependency.jcloud-bom.version>
```

确认存在 BOM import：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.bnpp.pf.jcloud</groupId>
            <artifactId>jcloud-bom</artifactId>
            <version>${dependency.jcloud-bom.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

参考：

- [partner-batch POM](/<本机工作区>/partner-batch/pom.xml:7)
- [out-batch POM](/<本机工作区>/out-batch/pom.xml:7)

另外确认 Batch 依赖：

- 如果使用 `pf-spring-starter-batch`，保留。
- 如果直接使用 Boot 4 的 `spring-boot-starter-batch`，需要改为 JDBC 版本 `spring-boot-starter-batch-jdbc`。
- 不要添加 Querydsl，当前代码没有使用。
- 不要复制 partner/out 中其他与 objective-batch 无关的依赖。

## 3. 替代 `RunIdIncrementer`

删除 incrementer 后，实际启动脚本或 scheduler 每次启动时需要传递唯一参数：

```bash
run.id=$(date +%s%3N),java.lang.Long
```

如果使用 PF 指南中的参数拼接：

```bash
ARGUMENTS+=" run.id=$(date +%s%3N),java.lang.Long"
ARGUMENTS+=" $*"
```

自动生成的参数应放在用户参数之前，以便重启时显式传入旧 `run.id`。

如果是通过 Control-M、Kubernetes Job 或其他调度器启动，同样需要由调度器生成唯一 `run.id`。

出处：[PF 18 incrementer migration](/<本机迁移指南目录>/18.0.0.md:350)。

## 4. 迁移 Spring Batch 数据库

如果生产/pprod 已经有 Spring Batch 5 的 `BATCH_*` 表，上线 PF 18 前必须执行 Batch 6 migration。

PostgreSQL 的标准变更是：

```sql
ALTER SEQUENCE BATCH_JOB_SEQ
RENAME TO BATCH_JOB_INSTANCE_SEQ;
```

PF 指南要求使用：

```text
changelog-spring-batch-6-0-migration.yaml
```

可以：

- 由应用 Liquibase include 这个 changelog；或者
- 由 DBA 在部署前执行迁移 SQL。

当前 [db.changelog-master.yaml](/<本机工作区>/objective-batch/src/main/resources/db/changelog/db.changelog-master.yaml:1) 是空的。如果 Liquibase 由应用负责，需要把迁移 changelog include 进去；如果 DB migration 由外部部署流程负责，则这里不一定修改。

全新数据库不执行上述 rename，直接使用 Batch 6 自带 schema 即可。

另外，上线前必须处理 Batch 5 中未完成的失败任务：

- 能重启的先在旧版本成功重启；或者
- 明确标记 abandon。

否则 Batch 6 无法可靠重启旧格式的 JobExecution。[Spring Batch 6 migration guide](https://github.com/spring-projects/spring-batch/wiki/Spring-Batch-6.0-Migration-Guide)

## 5. 删除旧 readiness workaround

从以下三个文件删除整个 block：

```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          exclude: "readinessState"
```

文件：

- [application-dev.yaml](/<本机工作区>/objective-batch/src/main/resources/application-dev.yaml:42)
- [application-pprod.yaml](/<本机工作区>/objective-batch/src/main/resources/application-pprod.yaml:41)
- [application-prod.yaml](/<本机工作区>/objective-batch/src/main/resources/application-prod.yaml:41)

PF 17 起：

- `readiness` 只检查 `readinessState`。
- Batch diagnostics 使用新的 `readiness-batch`。

当前 exclusion 会让标准 readiness 失去有效 indicator，所以需要删除。[PF 17 readiness migration](/<本机迁移指南目录>/17.0.0.md:23)

## 6. 构建和部署环境

确认所有地方：

- Java ≥ 21
- Maven ≥ 3.8.5
- CI 使用 Java 21 image
- Docker基础镜像至少 Java 21
- 服务器实际运行 JVM也是 Java 21或以上

Java 25只是推荐，不需要为了本次迁移升级。

如果应用运行在 Liberty 上，则停止迁移并确认架构，因为 PF 18/Spring Boot 4要求 Servlet 6.1，PF 指南明确说明 Liberty 场景不能直接升级。

## 7. `.gitignore`

加入：

```gitignore
**/.flattened-pom.xml
```

这是 PF 14 parent/flatten plugin 的迁移要求：[PF 14 guide](/<本机迁移指南目录>/14.0.0.md:5)。

## 8. 最后验证

建议依次执行：

```bash
mvn clean compile
mvn test
mvn verify
```

然后分别启动两个 Job，至少验证：

- Reader 能调用 `findLastMonth`。
- `Pageable` 参数正常传入。
- Objective 和 PartnerMds 分页排序正确。
- Writer 能接收新的 `Chunk`。
- 第一次运行成功。
- 使用新 `run.id` 可以再次运行。
- 使用相同 `run.id` 时符合 restart/AlreadyComplete 预期。
- pprod/prod 能连接已有 Batch 元数据表。
- readiness 和 `readiness-batch` 均正常。

暂时不要做的内容：不要切换新的 chunk model、不要添加 Querydsl、不要修改现有 JPQL、不要重写实体、不要为了迁移更改 chunk/page size，也不要随意升级父 POM未要求的业务依赖。

## 用户

这两个test报错

> [图片附件 × 2，未上传]

## 助手

这次是测试代码仍引用 Spring Batch 5 已删除/移动的 API，不是生产代码问题。我会把图二的四个 `cannot find symbol` 分成两类：`RunIdIncrementer` 测试残留，以及已经被 PF 的 builder factory 替代的 `JobBuilderFactory/StepBuilderFactory`。我只给出测试应如何改，不直接修改。

## 助手

图二的 4 个错误都来自测试代码还保留旧 API，不是生产代码的新错误。

## 1. `ObjectiveBatchConfigTest`

删除 import：

```java
import org.springframework.batch.core.launch.support.RunIdIncrementer;
```

Job 测试中删除：

```java
when(jobBuilder.incrementer(
        Mockito.any(RunIdIncrementer.class)))
        .thenReturn(jobBuilder);
```

生产代码已经不再调用 `.incrementer(...)`，测试也不能继续 mock 它。不要尝试给 `RunIdIncrementer` 换新包名。

修改后 Job 测试核心部分应是：

```java
when(pfJobBuilderFactory.get(
        Mockito.anyString(),
        Mockito.any()))
        .thenReturn(jobBuilder);

when(jobBuilder.start(Mockito.any(Step.class)))
        .thenReturn(simpleJobBuilder);

when(simpleJobBuilder.build())
        .thenReturn(job);

assertNotNull(monthGoalsBatchConfig.monthlyExtractionJob(
        pfJobBuilderFactory,
        jobRepository,
        step));
```

## 2. `PartnerMdsBatchConfigTest`

删除三个旧 import：

```java
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.core.launch.support.RunIdIncrementer;
```

前两个 class 已不存在，而且测试中实际使用的是 PF factory：

```java
PfJobBuilderFactory
PfStepBuilderFactory
```

因此不用替换，只需删除旧 import。

Partner Job 测试中同样删除：

```java
when(jobBuilder.incrementer(
        Mockito.any(RunIdIncrementer.class)))
        .thenReturn(jobBuilder);
```

保留：

```java
when(pfJobBuilderFactory.get(
        Mockito.anyString(),
        Mockito.any()))
        .thenReturn(jobBuilder);

when(jobBuilder.start(Mockito.any(Step.class)))
        .thenReturn(simpleJobBuilder);

when(simpleJobBuilder.build())
        .thenReturn(job);
```

## 3. 两个 Step 测试还要同步生产代码

你之前的生产代码现在使用的是：

```java
.chunk(20)
.transactionManager(myTransactionManager)
```

但照片里的测试仍然 mock 旧调用：

```java
when(stepBuilder.chunk(20, transactionManager))
        .thenReturn(simpleStepBuilder);
```

这可能不会产生当前的 `testCompile` 错误，但测试运行时会因为 mock 不匹配而得到 `null`，随后抛出 `NullPointerException`。

两个测试都应改成：

```java
when(stepBuilder.chunk(20))
        .thenReturn(simpleStepBuilder);

when(simpleStepBuilder.transactionManager(transactionManager))
        .thenReturn(simpleStepBuilder);
```

后面的 reader、processor、writer mock 保持：

```java
when(simpleStepBuilder.reader(Mockito.any(ItemReader.class)))
        .thenReturn(simpleStepBuilder);

when(simpleStepBuilder.processor(Mockito.any(ItemProcessor.class)))
        .thenReturn(simpleStepBuilder);

when(simpleStepBuilder.writer(Mockito.any(ItemWriter.class)))
        .thenReturn(simpleStepBuilder);

when(simpleStepBuilder.build())
        .thenReturn(taskletStep);
```

## 本次两个测试的最小修改总结

- `ObjectiveBatchConfigTest`
  - 删除 `RunIdIncrementer` import。
  - 删除 incrementer mock。
  - 更新 chunk/transactionManager mock。

- `PartnerMdsBatchConfigTest`
  - 删除旧 `JobBuilderFactory` import。
  - 删除旧 `StepBuilderFactory` import。
  - 删除 `RunIdIncrementer` import。
  - 删除 incrementer mock。
  - 更新 chunk/transactionManager mock。

删除旧 factory 的依据是 Spring Batch 6 API调整；删除 incrementer 测试的依据是生产代码已经按 [PF 18 migration guide](/<本机迁移指南目录>/18.0.0.md:350) 移除了 incrementer。修改这些之后再运行：

```bash
mvn test
```

图二显示的四个 `cannot find symbol` 会消失，同时避免紧接着出现两个 Step 测试的 NPE。

## 用户

> [图片附件 × 1，未上传]

## 助手

现在这个错误很明确：`.chunk(20)` 在新模型中返回的是 `ChunkOrientedStepBuilder<Object,Object>`，但测试仍让 Mockito 返回旧的 `SimpleStepBuilder<Object,Object>`。我再确认一下新 builder 的 `build()` 返回类型，避免你改完 mock 类型后马上遇到下一处类型错误。

## 助手

这个错误是因为测试里仍然使用：

```java
SimpleStepBuilder<Object, Object>
```

但生产代码调用：

```java
stepBuilder.chunk(20)
```

在 Spring Batch 6 中返回的是：

```java
ChunkOrientedStepBuilder<I, O>
```

所以 Mockito 的 `thenReturn(simpleStepBuilder)` 类型不匹配。

我上一条只让你修改调用方式，但遗漏了 mock 类型也必须一起改。

## Objective 测试修改

删除 import：

```java
import org.springframework.batch.core.step.builder.SimpleStepBuilder;
import org.springframework.batch.core.step.tasklet.TaskletStep;
```

添加：

```java
import org.springframework.batch.core.step.builder.ChunkOrientedStepBuilder;
import org.springframework.batch.core.step.item.ChunkOrientedStep;
```

把旧 mock：

```java
@Mock
SimpleStepBuilder<Object, Object> simpleStepBuilder;

@Mock
TaskletStep taskletStep;
```

改成：

```java
@Mock
ChunkOrientedStepBuilder<ObjectiveEntity, ObjectiveEntity>
        chunkOrientedStepBuilder;

@Mock
ChunkOrientedStep<ObjectiveEntity, ObjectiveEntity>
        chunkOrientedStep;
```

Step 测试改成：

```java
@Test
void monthlyExtractionStepTest() {
    when(pfStepBuilderFactory.get(
            Mockito.anyString(),
            Mockito.any()))
            .thenReturn(stepBuilder);

    when(stepBuilder
            .<ObjectiveEntity, ObjectiveEntity>chunk(20))
            .thenReturn(chunkOrientedStepBuilder);

    when(chunkOrientedStepBuilder
            .transactionManager(transactionManager))
            .thenReturn(chunkOrientedStepBuilder);

    when(chunkOrientedStepBuilder
            .reader(Mockito.any(ItemReader.class)))
            .thenReturn(chunkOrientedStepBuilder);

    when(chunkOrientedStepBuilder
            .processor(Mockito.any(ItemProcessor.class)))
            .thenReturn(chunkOrientedStepBuilder);

    when(chunkOrientedStepBuilder
            .writer(Mockito.any(ItemWriter.class)))
            .thenReturn(chunkOrientedStepBuilder);

    when(chunkOrientedStepBuilder.build())
            .thenReturn(chunkOrientedStep);

    assertNotNull(monthGoalsBatchConfig.monthlyExtractionStep(
            pfStepBuilderFactory,
            jobRepository,
            objectiveRepository,
            objectiveWriter,
            transactionManager,
            objectiveProcessor));
}
```

保留测试类上的：

```java
@SuppressWarnings("unchecked")
```

因为当前 Mockito matcher 使用了泛型接口的 raw class。

## PartnerMds 测试同样修改

Mock 类型使用：

```java
@Mock
ChunkOrientedStepBuilder<PartnerMdsEntity, PartnerMdsEntity>
        chunkOrientedStepBuilder;

@Mock
ChunkOrientedStep<PartnerMdsEntity, PartnerMdsEntity>
        chunkOrientedStep;
```

对应 stubbing：

```java
when(stepBuilder
        .<PartnerMdsEntity, PartnerMdsEntity>chunk(20))
        .thenReturn(chunkOrientedStepBuilder);

when(chunkOrientedStepBuilder
        .transactionManager(transactionManager))
        .thenReturn(chunkOrientedStepBuilder);

when(chunkOrientedStepBuilder
        .reader(Mockito.any(ItemReader.class)))
        .thenReturn(chunkOrientedStepBuilder);

when(chunkOrientedStepBuilder
        .processor(Mockito.any(ItemProcessor.class)))
        .thenReturn(chunkOrientedStepBuilder);

when(chunkOrientedStepBuilder
        .writer(Mockito.any(ItemWriter.class)))
        .thenReturn(chunkOrientedStepBuilder);

when(chunkOrientedStepBuilder.build())
        .thenReturn(chunkOrientedStep);
```

注意 `build()` 也不能继续返回 `TaskletStep`。新 builder 的返回类型是：

```java
ChunkOrientedStep<I, O>
```

官方 API明确给出了这两个返回类型：[ChunkOrientedStepBuilder](https://docs.spring.io/spring-batch/reference/api/org/springframework/batch/core/step/builder/ChunkOrientedStepBuilder.html)。

因此两组测试都需要同时替换：

```text
SimpleStepBuilder  → ChunkOrientedStepBuilder
TaskletStep        → ChunkOrientedStep
```

这样可以解决照片中 `thenReturn(SimpleStepBuilder)` 的编译错误，并避免下一步在 `build().thenReturn(taskletStep)` 处再次出现类型错误。


## 用户

还有什么要迁移的

## 助手

从你目前发的截图看，已经处理了：

- `Job`、`Step` 新包名。
- `RepositoryItemReader` Batch 6 构造方式。
- 生产代码移除 `RunIdIncrementer`。
- 新 `ChunkOrientedStepBuilder`。
- 两个 BatchConfig 测试的旧 builder/incrementer 问题。

接下来按以下清单完成。

## 1. 扫描剩余旧 Batch API

在项目根目录执行：

```bash
rg -n "org\.springframework\.batch\.item" src
rg -n "org\.springframework\.batch\.core\.(Job|Step)" src
rg -n "RunIdIncrementer|JobBuilderFactory|StepBuilderFactory|SimpleStepBuilder|TaskletStep" src
```

应该确认：

```java
org.springframework.batch.item...
```

全部迁到：

```java
org.springframework.batch.infrastructure.item...
```

重点检查：

```java
// Processor
org.springframework.batch.infrastructure.item.ItemProcessor

// Writer
org.springframework.batch.infrastructure.item.Chunk
org.springframework.batch.infrastructure.item.ItemWriter

// Reader
org.springframework.batch.infrastructure.item.data.RepositoryItemReader
```

如果所有 Step 已经使用新 chunk model，则测试里也不应再有：

```java
SimpleStepBuilder
TaskletStep
```

但业务上真正使用 tasklet step 的地方除外。

## 2. Repository 使用 `Pageable`

两个 `findLastMonth` 最后一个参数应为：

```java
import org.springframework.data.domain.Pageable;
```

```java
Page<ObjectiveEntity> findLastMonth(
        int month,
        int year,
        Pageable pageable);
```

```java
Page<PartnerMdsEntity> findLastMonth(
        int month,
        int year,
        Pageable pageable);
```

删除 `PageRequest` import。

## 3. POM

确认最终值：

```xml
<parent>
    <groupId>com.bnpp.pf.pfspring</groupId>
    <artifactId>pf-spring-parent</artifactId>
    <version>18.0.5</version>
</parent>
```

```xml
<dependency.jcloud-bom.version>
    2.0.4-pf-spring-as-parent.4
</dependency.jcloud-bom.version>
```

确认 JCloud BOM 已 import：

```xml
<dependency>
    <groupId>com.bnpp.pf.jcloud</groupId>
    <artifactId>jcloud-bom</artifactId>
    <version>${dependency.jcloud-bom.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

同时检查：

- 保留 `pf-spring-starter-batch`。
- 不添加未使用的 Querydsl。
- 不复制 partner/out 的其他无关依赖。
- 没有自己覆盖父 POM管理的 Spring、Hibernate、Lombok 等版本。

## 4. 启动器补充 `run.id`

移除 `RunIdIncrementer` 后，调度器或启动脚本必须传递唯一参数：

```bash
run.id=$(date +%s%3N),java.lang.Long
```

PF Shell 示例：

```bash
ARGUMENTS+=" run.id=$(date +%s%3N),java.lang.Long"
ARGUMENTS+=" $*"
```

否则相同业务参数再次运行时可能被识别为同一个已完成的 JobInstance。

## 5. Spring Batch 数据库

如果 pprod/prod 已存在 Batch 5元数据表，上线前必须执行：

```sql
ALTER SEQUENCE BATCH_JOB_SEQ
RENAME TO BATCH_JOB_INSTANCE_SEQ;
```

或者使用 PF 提供的：

```text
changelog-spring-batch-6-0-migration.yaml
```

当前 [db.changelog-master.yaml](/<本机工作区>/objective-batch/src/main/resources/db/changelog/db.changelog-master.yaml:1) 是空的。如果 Liquibase 由应用执行，需要 include migration changelog；如果由 DBA执行，则由部署流程处理。

上线前还要把 Batch 5 中需要重启的失败 execution：

- 在旧版本完成重启；或者
- 明确 abandon。

否则无法在 Batch 6 中可靠重启。

## 6. readiness 配置

从 dev、pprod、prod 删除：

```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          exclude: "readinessState"
```

PF 17以后 Batch 使用 `readiness-batch`，旧 workaround 不应继续保留。

## 7. 环境和仓库文件

确认：

- Java ≥ 21。
- Maven ≥ 3.8.5。
- CI image 是 Java 21或以上。
- Docker/runtime JVM是 Java 21或以上。
- 不是 Liberty 部署。
- `.gitignore` 包含：

```gitignore
**/.flattened-pom.xml
```

## 8. 完整验证

依次执行：

```bash
mvn clean compile
mvn test
mvn verify
```

然后做运行验证：

- 两个 Job 分别可以启动。
- Reader 能调用 `findLastMonth`。
- 分页排序正常。
- Writer 收到并处理 `Chunk`。
- 不同 `run.id` 能启动新实例。
- 指定旧 `run.id` 的失败实例能正常 restart。
- pprod/prod Batch metadata migration 成功。
- `readiness` 和 `readiness-batch` 正常。

不要额外修改实体、JPQL、chunk size、page size或业务逻辑；它们没有命中 PF 18的必需迁移项。

## 用户

先讲讲第二点，我改了但是不懂为什么改，不足尽可能详细的前置知识

## 助手

我先纠正一个重要点：把 `PageRequest` 改成 `Pageable` 更准确地说是“遵循 Spring Data / RepositoryItemReader 的公开契约”，并不一定是 PF 18 下绝对无法编译的硬性修改。`PageRequest` 本身实现了 `Pageable`，旧签名在当前实现里很可能仍能工作。下面我会从 Java 接口、分页原理、Spring Data 方法解析和 Batch Reader 调用链四层解释清楚。

## 助手

先给结论：`PageRequest` 改成 `Pageable` 是为了让 Repository 方法遵守 Spring Data 和 `RepositoryItemReader` 的公开接口契约。

但需要纠正我前面的一点：这不是 PF 18 下必然编译失败的硬性迁移。因为 `PageRequest` 实现了 `Pageable`，原签名在当前 Spring 实现中很可能仍然能运行。它更像是“消除对具体实现的依赖，使代码符合官方约定”。你既然已经改了，建议保留，不会改变分页行为。

## 1. 先理解为什么需要分页

假设数据库中有一百万条 Objective。

如果 Repository 一次性执行：

```java
List<ObjectiveEntity> findLastMonth(int month, int year);
```

Spring Data可能需要一次把上月所有数据加载到 JVM 内存中。这会带来：

- SQL 查询时间长。
- JVM内存占用高。
- Batch 启动后长时间没有输出。
- 数据量增大后可能发生 `OutOfMemoryError`。
- 失败重启时难以定位读取进度。

分页的思路是：每次只从数据库取固定数量的数据。

例如：

```text
第 0 页：第 1～40 条
第 1 页：第 41～80 条
第 2 页：第 81～120 条
```

对应 SQL 的概念类似：

```sql
SELECT ...
FROM objective
WHERE month = ?
  AND year = ?
ORDER BY id DESC
LIMIT 40 OFFSET 0;
```

下一页：

```sql
LIMIT 40 OFFSET 40;
```

再下一页：

```sql
LIMIT 40 OFFSET 80;
```

分页信息需要包含：

- 当前请求第几页。
- 每页多少条。
- 按什么字段排序。
- 升序还是降序。

Spring Data用 `Pageable` 表示这一组信息。

## 2. `Pageable` 和 `PageRequest` 的关系

### `Pageable` 是接口

它描述“一个分页请求应该具有什么能力”：

```java
public interface Pageable {
    int getPageNumber();
    int getPageSize();
    long getOffset();
    Sort getSort();
}
```

真实接口还有其他方法，但核心就是页码、大小、offset 和排序。

它只规定能力，不规定具体如何实现。

### `PageRequest` 是具体实现类

简化理解：

```java
public class PageRequest implements Pageable {
    // 保存 pageNumber、pageSize、sort
}
```

通常这样创建：

```java
Pageable pageable = PageRequest.of(
        0,
        40,
        Sort.by(Direction.DESC, "id"));
```

变量类型是接口：

```java
Pageable
```

实际对象是实现类：

```java
PageRequest
```

官方 API也明确说明 `PageRequest` 实现了 `Pageable`：[Pageable API](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/Pageable.html)、[PageRequest API](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/PageRequest.html)。

关系可以理解为：

```text
Pageable：任何“分页请求”
    ↑
PageRequest：Spring Data提供的一种分页请求实现
```

类似于常见的：

```java
List<String> list = new ArrayList<>();
```

这里：

- `List` 是接口。
- `ArrayList` 是具体实现。

通常不把方法参数写死成：

```java
void process(ArrayList<String> list)
```

而是写成：

```java
void process(List<String> list)
```

因为方法真正需要的是“一个 List”，并不关心它是不是 `ArrayList`。

`Pageable` 和 `PageRequest` 同理。

## 3. 两种 Repository 签名有什么区别

原来的代码：

```java
Page<ObjectiveEntity> findLastMonth(
        int month,
        int year,
        PageRequest pageRequest);
```

表示：

> 这个方法只接受 `PageRequest` 类型，或者它的子类。

迁移后的代码：

```java
Page<ObjectiveEntity> findLastMonth(
        int month,
        int year,
        Pageable pageable);
```

表示：

> 这个方法接受任何符合 `Pageable` 契约的实现。

因此下面的调用可以传给 `Pageable` 参数：

```java
PageRequest pageRequest = PageRequest.of(0, 40);

repository.findLastMonth(6, 2026, pageRequest);
```

因为 `PageRequest` 是 `Pageable`。

但是如果以后 Spring 使用另一种实现：

```java
Pageable pageable = someOtherPageableImplementation;
```

它能传给：

```java
findLastMonth(..., Pageable pageable)
```

却不能传给：

```java
findLastMonth(..., PageRequest pageRequest)
```

所以 `Pageable` 更宽、更稳定，也降低了 Repository 对 Spring 内部实现的耦合。

## 4. `RepositoryItemReader` 怎样调用这个方法

你的 BatchConfig 中大概是：

```java
RepositoryItemReader<ObjectiveEntity> objectiveQuery =
        new RepositoryItemReader<>(objectiveRepository, sorts);

objectiveQuery.setMethodName("findLastMonth");
objectiveQuery.setArguments(list);
objectiveQuery.setPageSize(40);
```

其中：

```java
list.add(lastMonth.getMonthValue());
list.add(lastMonth.getYear());
```

所以业务参数是：

```text
month
year
```

`RepositoryItemReader` 在读取每一页时，还会创建分页参数。逻辑上相当于：

```java
Pageable pageable = PageRequest.of(
        currentPage,
        pageSize,
        sort);
```

然后调用：

```java
objectiveRepository.findLastMonth(
        month,
        year,
        pageable);
```

因此最终的方法参数顺序必须是：

```java
findLastMonth(
    month,
    year,
    pageable
)
```

也就是：

```java
Page<ObjectiveEntity> findLastMonth(
        int month,
        int year,
        Pageable pageable);
```

为什么 `Pageable` 必须在最后？

因为 `RepositoryItemReader` 的配置中：

```java
setArguments(list)
```

只提供了业务参数：

```text
month、year
```

Reader 自己会把分页参数追加到最后面，然后通过方法名调用 Repository：

```text
[month, year] + [pageable]
```

Spring Batch 6 的 `RepositoryItemReader#setMethodName` 官方文档明确规定：

> 被调用的方法必须把 `Pageable` 作为最后一个参数。

参见 [Spring Batch 6 RepositoryItemReader API](https://docs.spring.io/spring-batch/reference/api/org/springframework/batch/infrastructure/item/data/RepositoryItemReader.html)。

## 5. Spring Data如何处理 `Pageable`

Repository 方法是：

```java
@Query("""
    FROM ObjectiveEntity obj
    WHERE obj.month = :month
      AND obj.year = :year
    """)
Page<ObjectiveEntity> findLastMonth(
        int month,
        int year,
        Pageable pageable);
```

这里 `month` 和 `year` 是普通查询参数。

`Pageable` 是 Spring Data识别的特殊参数，不会作为普通 JPQL参数绑定到 SQL 中。

Spring Data会从它里面读取：

```java
pageable.getPageNumber();
pageable.getPageSize();
pageable.getOffset();
pageable.getSort();
```

然后把分页和排序应用到查询上。

概念上：

```java
month = 6
year = 2026
pageNumber = 0
pageSize = 40
sort = id DESC
```

会产生类似：

```sql
SELECT ...
FROM objective
WHERE month = 6
  AND year = 2026
ORDER BY id DESC
LIMIT 40 OFFSET 0;
```

Spring Data官方示例也是：

```java
Page<User> findByLastname(
        String lastname,
        Pageable pageable);
```

参见 [Spring Data JPA query methods](https://docs.spring.io/spring-data/jpa/reference/repositories/query-methods-details.html)。

## 6. 为什么返回 `Page<Entity>`

方法返回：

```java
Page<ObjectiveEntity>
```

`Page` 不只包含本页数据，还可能包含分页元数据：

```java
page.getContent();
page.getNumber();
page.getSize();
page.getTotalElements();
page.getTotalPages();
page.hasNext();
```

例如：

```text
本页：40条
总数据：125条
总页数：4页
当前：第0页
还有下一页：是
```

为了得到 `totalElements` 和 `totalPages`，Spring Data通常还需要执行一条 count 查询：

```sql
SELECT COUNT(*)
FROM objective
WHERE month = ?
  AND year = ?;
```

这不是 PF 18新增行为，原来的 `Page` 就是这样工作的。

虽然 `Slice` 有时可以避免完整 count，但本次只是迁移，不需要把 `Page` 改成 `Slice`。

## 7. `pageSize(40)` 和 `chunk(20)` 不是一回事

你的配置中有两个数字：

```java
objectiveQuery.setPageSize(40);
```

以及：

```java
.chunk(20)
```

它们控制不同层次。

### Page size：数据库每次读取多少条

```java
setPageSize(40)
```

表示 Reader 每次向数据库请求大约 40 条。

### Chunk size：每个事务处理/提交多少条

```java
.chunk(20)
```

表示：

- 读取 20 条。
- 处理 20 条。
- 写入 20 条。
- 提交事务。
- 再处理下一组 20 条。

所以一次数据库分页返回的 40 条，通常可以供两个 chunk 使用：

```text
数据库查询一次：40条
    ├─ chunk 1：20条 → process/write/commit
    └─ chunk 2：20条 → process/write/commit
```

这两个值不必相等，也不需要为了迁移修改。

## 8. 为什么排序是分页的必要条件

你的 Reader 配置了：

```java
sorts.put("id", Direction.DESC);
```

PartnerMds 配置了：

```java
sorts.put("mdsId", Direction.DESC);
```

分页必须有稳定排序。

如果没有稳定排序，数据库第一次查询可能返回：

```text
A, B, C, D
```

第二次查询时，数据顺序发生变化，可能变成：

```text
C, E, F, G
```

这样可能导致：

- `C` 被重复读取。
- 某些记录被跳过。
- Batch 重启后读取位置不稳定。

因此 `Pageable` 中不只是 page number 和 page size，还包含 `Sort`。

`RepositoryItemReader` 构造器要求同时提供 repository 和 sorts，也是为了保证分页读取顺序。

## 9. 原来的 `PageRequest` 为什么可能仍然有效

严格从 Java 类型关系来看：

```java
PageRequest implements Pageable
```

而 `RepositoryItemReader` 当前通常创建的实际分页对象也是 `PageRequest`。

所以原方法：

```java
findLastMonth(int month, int year, PageRequest pageRequest)
```

很可能仍然能被当前实现调用。

这也是为什么：

- 它可能在 PF 18 下正常编译。
- 它不一定立刻报运行错误。
- `PageRequest`→`Pageable` 不是像 `RepositoryItemReader` 无参构造器删除那样的硬性破坏。

但官方只承诺：

```java
findLastMonth(..., Pageable pageable)
```

而不承诺 Reader 永远使用具体的 `PageRequest` 实现。

因此区别是：

```text
使用 PageRequest：
依赖当前具体实现，今天可能工作。

使用 Pageable：
依赖官方公开契约，长期兼容性更好。
```

## 10. 对本项目的实际影响

改成 `Pageable` 后：

- JPQL不需要修改。
- 返回类型不需要修改。
- `RepositoryItemReader` 配置不需要修改。
- 数据库 SQL逻辑不变。
- page size 不变。
- sort 不变。
- chunk size 不变。
- 业务结果不变。

唯一可能需要同步的是测试。如果测试中有：

```java
Mockito.any(PageRequest.class)
```

应改为：

```java
Mockito.any(Pageable.class)
```

或者测试直接调用：

```java
repository.findLastMonth(
        month,
        year,
        PageRequest.of(0, 40));
```

则仍然可以保留，因为 `PageRequest` 可以传给 `Pageable` 参数。

最终评价：如果严格坚持“只做 PF 18绝对必需的修改”，`PageRequest` 改 `Pageable` 不应被归类为硬性必改；但它符合 Spring Batch 5和6一直公开声明的调用契约，而且已经改完也不会引入行为变化，所以建议保留。

## 用户

第四点呢

## 助手

第四点是“移除 `RunIdIncrementer` 后，为什么启动器必须负责提供 `run.id`”。核心前置知识是 Spring Batch 如何用“Job 名称 + identifying JobParameters”定义一个 JobInstance，以及“新运行”和“失败重启”为什么是两件完全不同的事。我会顺便把正常运行、重复运行、失败重启三种场景分清。

## 助手

第四点的核心不是“必须使用一个叫 `run.id` 的参数”，而是：

> Spring Batch 必须能够区分“这是一次全新的业务运行”，还是“对上次失败运行的重启”。

`run.id` 是最简单的技术性唯一标识。

## 1. 先理解 Spring Batch 的三个核心概念

### Job

Job 是任务定义，也就是代码里的：

```java
@Bean
public Job monthlyExtractionJob(...) {
    return pfJobBuilderFactory
            .get("monthlyExtractionJob", jobRepository)
            .start(monthlyExtractionStep)
            .build();
}
```

这里的 Job 名称是：

```text
monthlyExtractionJob
```

它类似一个程序模板，描述执行哪些 Step，但不代表某一次具体运行。

### JobInstance

JobInstance 表示一次逻辑上的业务任务。

Spring Batch 根据下面两部分确定 JobInstance：

```text
Job 名称 + identifying JobParameters
```

可以近似理解为：

```text
JobInstance 唯一键 =
    monthlyExtractionJob
    + run.id
    + 其他 identifying 参数
```

例如：

```text
Job 名称：monthlyExtractionJob
run.id：1001
```

这代表一个 JobInstance。

换成：

```text
Job 名称：monthlyExtractionJob
run.id：1002
```

就是另一个 JobInstance。

### JobExecution

JobExecution 是对某个 JobInstance 的一次执行尝试。

例如 `run.id=1001` 第一次执行失败：

```text
JobInstance: monthlyExtractionJob + run.id=1001
    └── JobExecution 1：FAILED
```

使用相同参数重启后：

```text
JobInstance: monthlyExtractionJob + run.id=1001
    ├── JobExecution 1：FAILED
    └── JobExecution 2：COMPLETED
```

这是“重启”，不是新任务。

如果换成 `run.id=1002`：

```text
JobInstance: monthlyExtractionJob + run.id=1002
    └── JobExecution 3：STARTING
```

这就是新的任务实例。

## 2. 相同参数再次启动会发生什么

假设第一次启动：

```text
monthlyExtractionJob
run.id=1001
```

成功完成后，再次使用完全相同的 identifying 参数启动：

```text
monthlyExtractionJob
run.id=1001
```

Spring Batch会找到原来的 JobInstance，并看到它已经成功完成，因此通常抛出：

```text
JobInstanceAlreadyCompleteException
```

这是 Spring Batch用来防止同一批业务数据被无意中重复处理的保护机制。

不同状态的行为如下：

| 现有状态 | 使用相同参数再次启动 |
|---|---|
| `COMPLETED` | 拒绝，通常抛出 `JobInstanceAlreadyCompleteException` |
| `FAILED` | 创建新的 JobExecution，重启原 JobInstance |
| `STOPPED` | 按重启规则继续 |
| `STARTING/STARTED` | 拒绝，因为已有 execution 正在运行 |
| 不同 identifying 参数 | 创建新的 JobInstance |

所以“相同参数”不是简单的重复运行，而是 Spring Batch判断新运行与重启的依据。

## 3. 你的 Job 为什么需要额外参数

你的 `MonthGoalsBatchConfig` 在 Reader 内部这样计算月份：

```java
var lastMonth = LocalDate.now().minusMonths(1);
```

也就是说，处理月份不是从 JobParameters 中传入的。

如果启动时完全没有 identifying 参数，那么每次运行的身份可能都是：

```text
monthlyExtractionJob + 空参数
```

第一次成功后，下个月再次运行仍然是：

```text
monthlyExtractionJob + 空参数
```

Spring Batch可能认为这是同一个已经完成的 JobInstance，从而拒绝运行。

加入：

```text
run.id=不同值
```

以后：

```text
2026-07 第一次：
monthlyExtractionJob + run.id=1721484000000

2026-08 第二次：
monthlyExtractionJob + run.id=1724162400000
```

Spring Batch就会把它们识别为两个不同 JobInstance。

严格来说，更有业务含义的设计也可以使用：

```text
business.month=2026-07
```

作为 identifying 参数。但这会扩大改动范围，需要改 Reader、调度参数和重启流程。本次仅做 PF 18迁移，使用 PF 指南要求的 `run.id` 更直接。

## 4. 原来的 `RunIdIncrementer` 做了什么

原生产代码：

```java
.incrementer(new RunIdIncrementer())
```

它的设计用途是：当框架请求“启动下一个实例”时，自动生成或递增：

```text
run.id=1
run.id=2
run.id=3
```

概念上类似：

```java
if (previousRunId == null) {
    nextRunId = 1;
} else {
    nextRunId = previousRunId + 1;
}
```

需要注意：Spring Batch 6并没有彻底删除 `RunIdIncrementer`，而是把它移动到了：

```java
org.springframework.batch.core.job.parameters.RunIdIncrementer
```

但是本项目不应该只是把 import 改到新包后继续使用。

## 5. 为什么 PF 18要求删除 incrementer

Spring Batch 6重新明确了 incrementer 的语义：

- incrementer 代表由框架管理的实例序列。
- 使用 incrementer 时，框架负责计算下一个实例参数。
- 用户不应该一边使用自动序列，一边又自行修改下一实例的参数。

在 Batch 6的 next-instance/incrementer 路径中，如果 Job 配置了 incrementer，用户额外传入的 JobParameters 可能被忽略，并产生警告。[Spring Batch 6 migration guide](https://github.com/spring-projects/spring-batch/wiki/Spring-Batch-6.0-Migration-Guide)

例如调度器传入：

```text
month=7
year=2026
tenant=FR
```

但 Job 又有：

```java
.incrementer(new RunIdIncrementer())
```

框架可能按照 incrementer 自己计算下一组参数，而不是按调用方期望使用传入参数。

这会引起两个风险。

### 风险一：运行参数被忽略

调度器认为自己启动了：

```text
month=7
year=2026
```

但框架走 incrementer 序列后，实际参数不一定是调度器提供的完整参数。

### 风险二：重启可能变成新实例

失败实例：

```text
monthlyExtractionJob + run.id=1001
    └── FAILED
```

正确重启必须继续使用：

```text
run.id=1001
```

如果 incrementer自动产生：

```text
run.id=1002
```

Spring Batch创建的是全新 JobInstance，而不是重启失败的 `1001`。

这意味着：

- 已成功处理的记录可能重新处理。
- Reader 的 restart position 不会复用。
- 原来的失败实例仍然保持 FAILED。
- Batch metadata 中会出现另一个新实例。

因此 PF 18指南要求从 Job 定义中移除 incrementer，并把“新实例参数的生成”交给外部启动器：[PF 18 guide](/<本机迁移指南目录>/18.0.0.md:350)。

## 6. 为什么使用时间戳作为 `run.id`

PF 示例：

```bash
run.id=$(date +%s%3N),java.lang.Long
```

分成三部分理解。

### 参数名

```text
run.id
```

名字本身没有特殊魔法，只是 Spring Batch长期使用的约定名称。

如果 incrementer 已经删除，理论上也可以叫：

```text
execution.id
batch.request.id
```

但为了兼容运维习惯和 PF 示例，继续使用 `run.id` 最清楚。

### 参数值

```bash
$(date +%s%3N)
```

在 GNU/Linux 中表示 Unix epoch 毫秒，例如：

```text
1784569123456
```

正常情况下，每次启动都会得到不同值。

它不必连续：

```text
1、2、3
```

Spring Batch只要求新实例的 identifying 参数组合不同。

### 参数类型

```text
java.lang.Long
```

完整形式：

```text
run.id=1784569123456,java.lang.Long
```

表示：

- 名称：`run.id`
- 值：`1784569123456`
- Java类型：`Long`

如果不声明类型，具体解析结果可能是 String。虽然 String 也可以作为 identifying 参数，但保持 Long 与原 `RunIdIncrementer` 的语义及 PF 示例一致。

`run.id` 默认作为 identifying 参数参与 JobInstance 身份计算。

## 7. 正常新运行和失败重启必须区别处理

这是最重要的地方。

### 正常的新运行

例如每天或每月调度一次：

```bash
run.id=$(date +%s%3N),java.lang.Long
```

每次生成新值：

```text
第一次：run.id=1784569123456
第二次：run.id=1787247523456
```

每次都会创建新的 JobInstance。

### 失败后的重启

假设：

```text
run.id=1784569123456
```

对应的执行失败。

重启时不能再次生成新时间戳，否则会创建新 JobInstance。

错误做法：

```text
原失败参数：
run.id=1784569123456

重试时新生成：
run.id=1784569999999
```

这不是 restart，而是全新运行。

正确做法是：

```text
重试仍然使用：
run.id=1784569123456
```

或者直接通过 JobOperator/运维平台对原失败 execution 执行 restart，让框架复用原 JobParameters。

因此启动流程必须有两种模式：

```text
新运行：
生成新的 run.id

失败重启：
复用原来的 run.id / 原 JobParameters
```

## 8. PF Shell 示例为什么把 `$*` 放在后面

PF 指南示例：

```bash
ARGUMENTS+=" run.id=$(date +%s%3N),java.lang.Long"
ARGUMENTS+=" $*"
```

先放默认生成的 `run.id`，然后放用户提供的参数。

其意图是：

- 普通运行没有提供 `run.id`：使用自动时间戳。
- 运维重启显式提供旧 `run.id`：允许启动参数覆盖默认值。

概念上：

```bash
./launch.sh run.id=1784569123456,java.lang.Long
```

启动脚本先产生默认值，但用户提供的重启值在后面，由参数解析流程采用显式值。

实际是否“后值覆盖前值”还要由 PF launcher 的参数解析方式确认，因此迁移完成后应做一次失败重启测试，不能只验证全新运行。

## 9. 如果从 IDEA启动

IDEA Program Arguments 中可以填写：

```text
run.id=1784569123456,java.lang.Long
```

如果还需要指定 Job：

```text
--spring.batch.job.name=monthlyExtractionJob run.id=1784569123456,java.lang.Long
```

具体是否使用 `spring.batch.job.name` 取决于 PF launcher 和项目启动方式，但 `run.id` 是非 `--` 开头的 Job 参数，不应错误写成：

```text
--run.id=...
```

因为 `--xxx` 通常被 Spring Boot当作 application property，而不是 Batch JobParameter。

## 10. macOS 注意事项

你截图里的开发环境是 macOS，但：

```bash
date +%s%3N
```

主要是 GNU/Linux `date` 的写法。macOS 自带 BSD `date` 对 `%N` 的支持不同。

如果这段代码运行在 Kubernetes/Linux 容器或 Linux服务器中，使用 PF 示例通常没有问题。

如果启动脚本实际运行在 macOS，需要单独确认命令输出：

```bash
date +%s%3N
```

输出必须是真正的数字，例如：

```text
1784569123456
```

不能包含字面量 `%3N` 或 `N`。生产启动器通常运行在 Linux，因此不要因为本地 Mac 环境随意修改生产脚本。

## 11. 两个 Job 使用同一个 `run.id` 是否冲突

不冲突，因为 JobInstance 身份包含 Job 名称。

例如：

```text
monthlyExtractionJob + run.id=1001
monthlyExtractionPartnerMdsJob + run.id=1001
```

它们是两个不同 JobInstance，因为 Job 名称不同。

所以一次应用启动时，两个 Job拿到相同 `run.id` 没有问题。

## 12. 对测试的影响

单元测试中，生产代码已经删除：

```java
.incrementer(new RunIdIncrementer())
```

因此测试也要删除：

```java
when(jobBuilder.incrementer(...))
```

集成测试则应覆盖两种情况：

```java
// 新运行
run.id=1001

// 另一次新运行
run.id=1002
```

以及重启语义：

```java
// 第一次执行失败
run.id=1001

// 重启仍然使用相同参数
run.id=1001
```

最后总结：

```text
RunIdIncrementer
    ↓ 删除
外部启动器生成 run.id
    ↓
新运行：新 run.id
失败重启：复用旧 run.id
```

如果调度平台已经始终提供其他唯一的 identifying 参数，那么技术上不一定必须再增加名为 `run.id` 的参数；必须保证的是“新实例参数唯一、重启参数不变”。但 objective-batch 当前在代码内部计算月份，没有明显的外部业务标识，所以按 PF 指南增加 `run.id` 是最小、最安全的迁移方式。

## 用户

但是这个项目并不是iaas

## 助手

对，这个区别很关键。如果项目不是 IAAS，那么我前面给出的 Shell `ARGUMENTS+=...` 不能直接当成你的必改项；它很可能只是指南针对 IAAS/Gclweb 启动脚本的落地示例。我先核对 PF 18指南中这一段的适用范围，再把“所有平台都需要的语义”和“仅 IAAS需要的脚本修改”分开。

## 助手

指南已经确认：`execJava_batch_helloworld.env` 的两行 Shell 明确属于“IAAS 示例”，所以你的项目不需要创建或修改这个文件。现在只需要确认 PaaS 的实际启动入口是否已经由平台自动生成唯一 Job 参数；我在现有同类项目里找一下 `run.id`/Helm/CronJob 的用法，避免把 IAAS做法套到 PaaS。

## 助手

你说得对：既然项目不是 IAAS，就不应该修改或创建：

```text
execJava_batch_*.env
```

也不需要直接照抄：

```bash
ARGUMENTS+=" run.id=$(date +%s%3N),java.lang.Long"
ARGUMENTS+=" $*"
```

PF 18指南第 380 行明确说这是：

> For example, with a batch job deployed on IAAS

所以那段 Shell 只是 IAAS 的实现示例，不是所有项目都必须照搬。[PF 18 guide](/<本机迁移指南目录>/18.0.0.md:380)

## 所有平台都必须做的部分

指南对所有 Batch Job 的要求是：

```text
All batch-job definitions must no longer use a JobParametersIncrementer.
```

也就是生产代码必须删除：

```java
.incrementer(new RunIdIncrementer())
```

这是平台无关的要求：[PF 18 guide](/<本机迁移指南目录>/18.0.0.md:350)。

## 只有 IAAS 才做的部分

以下属于 IAAS实现方式：

```text
execJava_batch_helloworld.env
ARGUMENTS+=" ..."
```

你的项目不是 IAAS，所以不做这项。

## PaaS真正需要确认什么

PaaS 下通常由平台的 Batch launcher、Helm CronJob、调度服务或其他部署组件启动应用。

你需要确认的不是“有没有 IAAS Shell”，而是：

> PaaS 每次启动新的批处理时，是否已经传入一个新的 identifying JobParameter。

这个参数可能叫：

```text
run.id
```

也可能是：

```text
execution.id
request.id
business.date
schedule.timestamp
```

只要满足下面两个条件即可：

- 每次全新运行时值不同。
- 失败重启时复用原值。

## 三种情况

### 情况一：PaaS自动注入 `run.id`

例如平台每次启动时实际传入：

```text
run.id=1784569123456
```

下一次：

```text
run.id=1787247523456
```

这种情况下：

- 删除 Java 中的 incrementer。
- 不修改 Shell。
- 不需要在 application 代码里生成 `run.id`。
- 不需要额外增加配置。

这可能是你们 PaaS 平台已经提供的标准能力。

### 情况二：PaaS注入其他唯一参数

例如：

```text
execution.id=a84fc7...
```

或者：

```text
business.date=2026-07-20
```

只要它是 identifying 参数，能够区分 JobInstance，就不一定必须再增加 `run.id`。

Spring Batch识别 JobInstance的规则是：

```text
Job名称 + 所有 identifying JobParameters
```

因此名称不一定非得是 `run.id`。

### 情况三：PaaS完全没有传 JobParameters

如果每次启动都是：

```text
monthlyExtractionJob + 空参数
```

第一次成功后，第二次可能被 Spring Batch认为是同一个已经完成的 JobInstance，出现：

```text
JobInstanceAlreadyCompleteException
```

这种情况下才需要在 PaaS launcher、Helm/CronJob 或调度平台中配置一个唯一参数。

注意：应当在 PaaS启动层配置，不要为了代替 IAAS Shell，在 Java Bean 内部写：

```java
System.currentTimeMillis()
```

自动生成 JobParameter。那样会让失败重启难以复用原参数。

## 怎么确认 PaaS是否已经处理

建议在 dev 环境检查一次真实运行。

### 查看启动日志

搜索 JobParameters，确认是否出现类似：

```text
run.id=...
```

或者其他 execution/timestamp 参数。

### 查看 Batch 元数据

查看 `BATCH_JOB_EXECUTION_PARAMS` 中一次执行对应的参数，重点确认：

- 是否有 `run.id` 或其他唯一参数。
- 是否标记为 identifying。
- 两次正常调度的参数值是否不同。
- 失败重启时是否复用原值。

### 做两次运行测试

第一次正常完成后，再触发一次新的 PaaS调度。

结果：

- 如果第二次创建新的 JobInstance：PaaS已经提供唯一参数，不需要额外修改。
- 如果出现 `JobInstanceAlreadyCompleteException`：PaaS没有提供新的 identifying 参数，需要修改 PaaS启动配置。
- 如果失败任务重试时创建了全新 JobInstance：平台没有正确实现 restart，需要让重启复用原参数。

## 对 objective-batch 的准确结论

你现在确定需要做的是：

```java
// 删除
.incrementer(new RunIdIncrementer())
```

不需要做的是：

```text
不创建 execJava_batch_*.env
不添加 IAAS ARGUMENTS Shell
```

尚需确认的是：

```text
PaaS launcher 是否已经为每次新运行提供唯一 identifying 参数
```

如果已经提供，第四点到此结束，没有其他代码需要修改。如果没有，则应修改 PaaS调度/部署配置，而不是 Java BatchConfig。之前我把 IAAS Shell 例子直接泛化到了你的项目，这一点不准确。


## 用户

我需要你把公司的migration guide和官方的migration guide关于这个incrementer的原文结合所有可能影响我理解现在情况的小点全部穿起来，有逻辑顺序的再给我讲一遍

## 助手

我会把内容分成三层串起来：先逐段引用 PF 公司指南，再对应 Spring Batch 6 官方原文，最后落到 objective-batch 的代码、测试、PaaS启动和重启行为。特别会区分“官方只是改变语义/移动类”和“公司指南为什么进一步要求所有 PF Batch 都删除 incrementer”。

## 助手

先给你一个最准确的结论：

> Spring Batch 6官方并没有删除 `RunIdIncrementer`，也没有要求所有 Spring Batch应用都禁止使用它；官方重新收紧了它的语义。PF公司的迁移指南基于 PF Batch 的启动、诊断和重启要求，进一步制定了更严格的规则：所有 PF Batch Job 定义都必须移除 `JobParametersIncrementer`。

对你的 PaaS 项目来说：

- Java Job 定义中必须删除 `.incrementer(...)`。
- 测试中的 incrementer mock/import 必须删除。
- 不需要照搬 IAAS 的 `execJava_batch_*.env` Shell。
- 需要确认 PaaS launcher 是否已经为每次逻辑上的新运行提供唯一且 identifying 的 JobParameter。
- PaaS失败重试必须复用原参数，不能每次 Pod 重启都产生新值。

下面把公司指南、官方指南和实际行为完整串起来。

---

## 一、公司 PF 18指南到底说了什么

公司指南这一节标题是：

```text
4.4.2. JobParametersIncrementer changes
```

第一条是全平台规则，不区分 IAAS/PaaS：

> All batch-job definitions must no longer use a `JobParametersIncrementer`.

即：

> 所有 Batch Job 定义都不能再使用 `JobParametersIncrementer`。

出处：[PF 18 guide 第350～354行](/<本机迁移指南目录>/18.0.0.md:350)。

这句话说的是：

```java
pfJobBuilderFactory
    .get(...)
    .incrementer(...)
```

必须删除。

它不只针对：

```java
new RunIdIncrementer()
```

也包括任何自定义实现：

```java
.incrementer(new CustomJobParametersIncrementer())
.incrementer(new RunIdParameterIncrementer())
```

### 公司指南给出的两个原因

原文：

> Spring Batch 6.0 introduced two breaking limitations for jobs that have an incrementer attached:
>
> - Runtime parameters are ignored: any parameters you pass when launching the job are discarded.
> - Restart support is broken: Spring Boot cannot restart a failed job that uses an incrementer.

出处：[PF 18 guide 第356～361行](/<本机迁移指南目录>/18.0.0.md:356)。

公司指南将影响总结为：

1. 运行时传入的参数可能被丢弃。
2. 带 incrementer 的失败 Job 无法按 PF/Boot 预期重启。

### 为什么 PF 特别在意

公司指南继续说明，PF Batch 必须支持：

> Preserve the ability to restart

以及：

> force a diagnostic check or stop after the diagnostic

出处：[PF 18 guide 第363～366行](/<本机迁移指南目录>/18.0.0.md:363)。

也就是说，PF Batch启动时除了业务参数，还可能有诊断参数，例如：

```text
forceDiagnostic
stopAfterDiagnostic
```

如果 incrementer 接管了下一实例参数的计算，这些启动时参数可能不能按调用方预期生效。

因此 PF的决定不是“换一个 import”，而是：

```text
彻底不让 Job 定义挂 incrementer
```

---

## 二、Spring Batch 6官方到底改了什么

官方首先说明了 `JobParametersIncrementer` 的设计目的：

- 它适合存在“自然序列”的 Job。
- 例如每小时、每天、每个连续编号运行一次。
- 框架根据上一实例参数计算下一实例参数。
- 它主要服务于 `startNextInstance` 语义。

官方最关键的原文是：

> “it does not make sense to provide job parameters when using an incrementer”

以及：

> “any additional parameters supplied by the user will be ignored with a warning”

[Spring Batch 6 official migration guide](https://github.com/spring-projects/spring-batch/wiki/Spring-Batch-6.0-Migration-Guide)

官方的逻辑是：

```text
如果 Job 使用 incrementer
    ↓
下一实例的参数应完全由 incrementer 控制
    ↓
调用方不应该同时自行修改下一实例参数
```

这类似数据库 sequence：

```text
数据库负责生成 1、2、3、4
```

调用方不应该一边让数据库自动生成，一边又要求下一值是 100。

### 官方没有删除 `RunIdIncrementer`

Spring Batch 6 中它仍存在，类移动到了：

```java
org.springframework.batch.core.job.parameters.RunIdIncrementer
```

它仍然会递增 Long 类型的 `run.id`：

```text
没有 run.id → 产生 run.id=1
run.id=1    → 产生 run.id=2
run.id=2    → 产生 run.id=3
```

[Spring Batch 6 RunIdIncrementer API](https://docs.spring.io/spring-batch/reference/api/org/springframework/batch/core/job/parameters/package-summary.html)

所以你最初测试中的报错：

```java
import org.springframework.batch.core.launch.support.RunIdIncrementer;
```

单从官方包迁移角度，可以改成新包。

但是对 PF项目来说不能这么处理，因为：

```text
官方：类仍可用于标准 startNextInstance 场景
公司：所有 PF Batch Job 都不能再挂 incrementer
```

因此 objective-batch 的正确处理是删除，不是改 import。

---

## 三、为什么会涉及 JobInstance 和重启

理解 incrementer 前，需要先理解四个概念：

```text
Job
JobParameters
JobInstance
JobExecution
```

### Job

Job 是代码定义：

```java
.get("monthlyExtractionJob", jobRepository)
.start(monthlyExtractionStep)
.build();
```

它描述“要执行什么”，但不是某一次运行。

### JobParameters

是本次启动传给 Job 的参数，例如：

```text
run.id=1001
country=FR
forceDiagnostic=true
```

每个参数可以是：

- identifying
- non-identifying

### JobInstance

Spring Batch用下面的组合识别一个逻辑任务实例：

```text
Job 名称 + identifying JobParameters
```

例如：

```text
monthlyExtractionJob + run.id=1001
```

是一个 JobInstance。

```text
monthlyExtractionJob + run.id=1002
```

是另一个 JobInstance。

### JobExecution

JobExecution 是一次执行尝试。

例如：

```text
JobInstance：
monthlyExtractionJob + run.id=1001

JobExecution 1：FAILED
JobExecution 2：COMPLETED
```

这代表第一次失败，第二次重启成功；两个 Execution 属于同一个 JobInstance。

---

## 四、identifying 和 non-identifying 参数

### identifying 参数

参与 JobInstance 身份计算。

例如：

```text
run.id=1001, identifying=true
```

把它改成 `1002` 会创建新的 JobInstance。

### non-identifying 参数

传给 Job使用，但不改变 JobInstance身份。

例如公司指南中的：

```java
.parameter(FORCE_DIAG_PARAM)
    .nonIdentifying()
```

假设：

```text
run.id=1001
forceDiagnostic=false
```

改成：

```text
run.id=1001
forceDiagnostic=true
```

如果 `forceDiagnostic` 是 non-identifying，这两个参数仍指向同一个 JobInstance。

公司示例中：

```java
.parameter(STOP_AFTER_DIAG_PARAM)
    .identifying()

.parameter(FORCE_DIAG_PARAM)
    .nonIdentifying()
```

说明这两个诊断参数有不同身份语义。[PF 18 guide](/<本机迁移指南目录>/18.0.0.md:397)

---

## 五、`RunIdIncrementer` 原来解决什么问题

假设一个 Job启动时没有任何 identifying 参数：

```text
monthlyExtractionJob + 空参数
```

第一次成功后，再次用空参数运行，Spring Batch会找到同一个已经完成的 JobInstance，通常抛出：

```text
JobInstanceAlreadyCompleteException
```

`RunIdIncrementer` 的旧用法是：

```java
.incrementer(new RunIdIncrementer())
```

当 launcher 请求启动下一实例时，框架生成：

```text
第一次：run.id=1
第二次：run.id=2
第三次：run.id=3
```

所以每次都是新 JobInstance：

```text
monthlyExtractionJob + run.id=1
monthlyExtractionJob + run.id=2
monthlyExtractionJob + run.id=3
```

这个设计本身没有逻辑错误。问题是 Spring Batch 6明确规定：

> 一旦选择 incrementer，下一实例参数应由框架控制。

这和 PF Batch 需要调用方传入诊断参数、运维参数、重启参数产生冲突。

---

## 六、`start`、`startNextInstance` 和 `restart` 不是同一件事

这是理解官方变化最关键的地方。

### `start`

调用方主动提供参数：

```text
start(
    monthlyExtractionJob,
    run.id=1001,
    forceDiagnostic=true
)
```

参数属于调用方。

### `startNextInstance`

概念上是：

```text
请框架计算并启动下一实例
```

它不要求调用方提供下一组参数，而是使用 Job 上挂载的 incrementer。

官方 API 中 `startNextInstance` 的语义就是：

```text
根据 JobParametersIncrementer 确定下一实例
```

### `restart`

重启一个已经存在的失败 execution：

```text
restart(jobExecutionId)
```

它应复用原 JobInstance 和原 identifying 参数。

因此三者分别代表：

```text
start              调用方给参数
startNextInstance  incrementer给参数
restart            复用历史参数
```

PF的问题在于原有 Boot/PF启动链可能把这些语义混合起来：Job 一旦挂 incrementer，框架可能走下一实例参数计算，导致运行时参数或重启参数不能按预期使用。

---

## 七、为什么 incrementer 会破坏“失败重启”

假设第一次运行：

```text
monthlyExtractionJob + run.id=1001
    └── JobExecution 1：FAILED
```

正确重启必须保持：

```text
run.id=1001
```

这样得到：

```text
monthlyExtractionJob + run.id=1001
    ├── JobExecution 1：FAILED
    └── JobExecution 2：COMPLETED
```

但如果 incrementer产生：

```text
run.id=1002
```

Spring Batch看到的是：

```text
monthlyExtractionJob + run.id=1002
```

于是创建新的 JobInstance：

```text
JobInstance 1001：仍然 FAILED
JobInstance 1002：新的运行
```

后果包括：

- 原失败实例没有被重启。
- 原 Step ExecutionContext 不会作为同一实例的 restart state 使用。
- 之前成功处理的数据可能被再次处理。
- Batch metadata 中留下一个 FAILED实例和一个新实例。
- 诊断参数也可能没有按调用方要求进入新参数集合。

这就是公司指南所说：

> Restart support is broken

它不是说 Job代码完全不能再次执行，而是说：

> 原本应该属于同一 JobInstance 的 restart，可能变成另一个新 JobInstance。

---

## 八、公司指南要求怎样替代 incrementer

公司原文：

> The simplest migration path is to replace the incrementer with a timestamp-based parameter.
>
> Provide a timestamp value for the parameter that would normally be set by the `JobParametersIncrementer`.
>
> This preserves uniqueness for each run while keeping the job restartable and allowing runtime-supplied parameters to be taken into account.

出处：[PF 18 guide 第368～374行](/<本机迁移指南目录>/18.0.0.md:368)。

这里的关键不是“时间戳必须由 Java生成”，而是：

```text
Job Definition 不再生成参数
    ↓
启动方负责传入参数
```

也就是职责从：

```java
Job Bean 内的 incrementer
```

移到：

```text
launcher / scheduler / PaaS平台
```

---

## 九、为什么 timestamp 参数要放在运行时参数之前

公司原文：

> Add the timestamp parameter (commonly `run.id`) before the parameters that can be provided at runtime.
>
> By placing it first, the timestamp can be overridden when necessary—for example, when you need to restart a failed job instance.

出处：[PF 18 guide 第376～378行](/<本机迁移指南目录>/18.0.0.md:376)。

目标是同时支持两种情况。

### 新运行

启动器产生新的默认值：

```text
run.id=2001
```

因此创建新 JobInstance。

### 失败重启

运维明确提供原值：

```text
run.id=1001
```

它覆盖默认 timestamp，于是重新进入原来的 JobInstance。

逻辑是：

```text
默认值：用于新运行
显式值：用于重启
```

---

## 十、参数字符串到底表示什么

公司 IAAS 示例：

```text
run.id=$(date +%s%3N),java.lang.Long
```

展开后类似：

```text
run.id=1784569123456,java.lang.Long
```

Spring Batch 6完整的命令行参数格式是：

```text
name=value,type,identifying
```

例如：

```text
run.id=1784569123456,java.lang.Long,true
```

三部分分别是：

```text
名称：run.id
值：1784569123456
类型：java.lang.Long
identifying：true
```

如果省略最后的 `true/false`，默认转换器会把参数作为 identifying 参数。[Spring Batch command-line documentation](https://docs.spring.io/spring-batch/reference/job/running.html)

所以 PF 示例：

```text
run.id=1784569123456,java.lang.Long
```

等价于默认 identifying 的 Long 参数。

---

## 十一、公司指南中的 Validator 为什么没有删除

公司迁移前：

```java
.incrementer(new RunIdParameterIncrementer())
.validator(...)
```

迁移后：

```java
.validator(...)
```

incrementer 删除，但 validator 保留。

迁移后的 Validator 仍要求：

```java
.parameter(RUN_ID_PARAM)
    .required()
    .identifying()
    .longTypeAndValueComplying(gt(0))
```

这说明新的职责划分是：

```text
launcher：负责产生 run.id
validator：负责验证 run.id
Job：只消费已经验证过的参数
```

而不是：

```text
Job incrementer：自己生成 run.id
```

但是 objective-batch 当前没有这种 `JobParametersValidator`。公司示例并没有要求所有项目无条件新增 Validator，所以按照“只做必须迁移”：

- 不因为示例而给 objective-batch 新增 Validator。
- 如果项目原来已有 Validator，则保留，并确认 `run.id` 仍是 required/identifying Long。
- 如果 PaaS平台自身已保证参数格式，也可以继续依赖平台约束。

---

## 十二、为什么 IAAS 示例不适用于你的 PaaS

公司指南先讲了通用原则：

```text
删除 incrementer
由启动方提供 timestamp 参数
默认值必须可被 restart 参数覆盖
```

然后第380行明确说：

> For example, with a batch job deployed on IAAS

接着才给出：

```text
execJava_batch_helloworld.env
```

以及：

```bash
ARGUMENTS+=" run.id=$(date +%s%3N),java.lang.Long"
ARGUMENTS+=" $*"
```

因此结构是：

```text
通用规则
    ↓
IAAS 的一种实现示例
```

不是：

```text
所有平台都必须有 execJava_batch_*.env
```

你的项目是 PaaS，所以：

- 不创建 IAAS env 文件。
- 不复制 Shell `ARGUMENTS`。
- 不在 Java里模拟这个 Shell。
- 应由 PaaS launcher、CronJob、Helm配置或平台调度服务提供参数。

---

## 十三、PaaS 下真正要检查的四个条件

### 条件一：新逻辑运行得到新 identifying 参数

例如第一次调度：

```text
run.id=1001
```

第二次调度：

```text
run.id=1002
```

这样是两个 JobInstance。

### 条件二：同一逻辑运行的 Pod 重试不能产生新参数

这是 PaaS 特别容易忽略的地方。

假设 Kubernetes启动一个 Batch Pod：

```text
run.id=1001
```

Pod因为节点故障被重新创建。如果新 Pod 自动产生：

```text
run.id=1002
```

Spring Batch会认为这是全新任务，而不是对 `1001` 的恢复。

因此唯一参数必须在“逻辑调度实例”层生成，而不是每次 Java进程启动时生成。

正确：

```text
Cron/Scheduler execution 42
    run.id=1001
    ├── Pod attempt 1：失败
    └── Pod attempt 2：仍使用1001
```

错误：

```text
Pod attempt 1：run.id=1001
Pod attempt 2：run.id=1002
```

### 条件三：Spring Batch restart 复用原参数

如果使用 JobOperator：

```text
restart(jobExecutionId)
```

应该由 Batch metadata 复用历史参数。

如果平台通过重新启动应用实现 restart，就必须重新传入原来的：

```text
run.id
```

以及其他 identifying 和 non-identifying 参数。

### 条件四：参数必须确实是 identifying

如果 PaaS传入：

```text
run.id=1001
```

但把它标成：

```text
identifying=false
```

那么它不会参与 JobInstance身份计算，无法区分新运行。

需要在日志、平台配置或 `BATCH_JOB_EXECUTION_PARAMS` 中确认 identifying 标志。

---

## 十四、objective-batch 自身为什么需要某种唯一参数

你的 Reader 使用：

```java
LocalDate.now().minusMonths(1)
```

业务月份是在 Java运行时计算的，不是 JobParameter。

因此如果 PaaS不传任何 identifying 参数，两个月的启动可能都是：

```text
monthlyExtractionJob + 空参数
```

第一次完成后，下一次可能出现：

```text
JobInstanceAlreadyCompleteException
```

所以 objective-batch 必须满足下面二选一：

1. PaaS已经传入一个唯一 identifying 参数。
2. 在 PaaS调度配置中新增 `run.id` 或等价参数。

不一定非要叫 `run.id`。如果 PaaS已经传入：

```text
execution.id
schedule.id
business.date
```

且它满足：

- identifying=true
- 每个新调度不同
- 同一调度重试保持不变

那么不需要再增加 `run.id`。

---

## 十五、两个 Job 使用同一个 `run.id` 是否冲突

objective-batch 有两个 Job：

```text
monthlyExtractionJob
monthlyExtractionPartnerMdsJob
```

JobInstance身份包含 Job名称，所以：

```text
monthlyExtractionJob + run.id=1001
```

与：

```text
monthlyExtractionPartnerMdsJob + run.id=1001
```

是两个不同 JobInstance，不冲突。

但是是否应该并发运行是另一个业务问题。`run.id` 只负责身份识别，不保证两个 Job并发操作相同数据时一定安全。

---

## 十六、`RunIdIncrementer` 和数据库 sequence 不是一回事

Spring Batch 6还要求数据库迁移：

```sql
ALTER SEQUENCE BATCH_JOB_SEQ
RENAME TO BATCH_JOB_INSTANCE_SEQ;
```

这个数据库 sequence 是给内部主键生成 ID：

```text
JOB_INSTANCE_ID
JOB_EXECUTION_ID
```

它与：

```java
RunIdIncrementer
```

完全不同。

区别：

| 项目 | 用途 |
|---|---|
| `RunIdIncrementer` | 生成 identifying JobParameter，如 `run.id` |
| `BATCH_JOB_INSTANCE_SEQ` | 给数据库中的 JobInstance记录生成内部主键 |

完成数据库 sequence rename 不会自动替代 `run.id`，删除 Java incrementer也不影响数据库内部 ID sequence。

---

## 十七、还有一个独立的“历史失败任务”问题

Spring Batch 6改变了 JobParameter 的领域模型和持久化格式。

官方迁移指南还明确说明：

- Batch 5保存的参数序列化格式与 Batch 6不同。
- Batch 5中失败的 JobInstance 不能直接在 Batch 6中重启。
- 升级前要么在 Batch 5中重启成功，要么将失败 execution abandon。

这与 incrementer 是两个独立问题：

```text
问题A：
带 incrementer 的 Batch 6启动/重启语义变化

问题B：
Batch 5历史参数格式不能直接用于 Batch 6 restart
```

即使你正确删除了 incrementer，仍要在正式升级前处理 Batch 5遗留失败 execution。

---

## 十八、落实到 objective-batch 的完整动作

### Java生产代码

两个 Job 都删除：

```java
.incrementer(new RunIdIncrementer())
```

删除旧 import，不改成新包。

### 单元测试

删除：

```java
import ...RunIdIncrementer;
```

删除：

```java
when(jobBuilder.incrementer(...))
```

测试直接 mock：

```java
when(jobBuilder.start(...))
```

### PaaS部署

不创建：

```text
execJava_batch_*.env
```

检查 PaaS实际 JobParameters：

- 是否有唯一 identifying 参数。
- 新调度是否产生新值。
- 同一调度的 Pod重试是否保持原值。
- 失败 restart 是否复用原参数。

### 数据库/运行验证

第一次使用：

```text
run.id=1001
```

人为使其失败。

然后验证：

- 使用 `1001`：进入同一个 JobInstance，产生新 JobExecution。
- 使用 `1002`：创建全新 JobInstance。
- `1001` 成功完成后再使用 `1001`：应被判定为 already complete。

---

最终逻辑可以压缩成这一条链：

```text
Spring Batch 6明确：
incrementer代表“框架拥有下一实例参数序列”
    ↓
因此调用方附加参数会与 incrementer语义冲突
    ↓
PF Batch必须接收诊断参数并可靠 restart
    ↓
公司规定所有 PF Batch Job删除 incrementer
    ↓
新实例唯一参数改由启动平台提供
    ↓
IAAS通过 Shell提供
PaaS通过平台 launcher/scheduler提供
    ↓
新运行使用新值
失败重启复用旧值
```

对你当前的 PaaS 项目，唯一尚不能从源码判断的是：PaaS是否已经自动提供满足上述条件的 identifying 参数。这个问题不能通过继续改 Java代码解决，需要检查实际 PaaS启动参数、调度配置或 Batch metadata。

## 用户

那你给我解释一下为什么partnerbatch和outbatch迁移的时候都需要增加代码来自己构建runid，并且还方法还不一样，但是objectivebatch里面什么都不用干

## 助手

这个问题不能仅凭“都是 PaaS”判断，差别通常来自三个地方：它们如何启动 Job、是否绕过 PF/Boot 的默认 launcher、以及一个进程里启动几个 Job。我先把本地 `partner-batch`、`out-batch` 和 `objective-batch` 的启动入口、Job 构建方式、JobParameters 处理逐项对照，再解释为什么代码位置和方法会不同；如果本地版本并不是你所说的已迁移版本，我也会明确指出，避免臆测。

## 助手

本地代码已经显示出三者的关键架构差异：`partner-batch` 在业务方法里显式 `jobLauncher.run(..., JobParameters)`；`out-batch` 在自定义 `CommandLineRunner` 里显式启动所有 Job；`objective-batch` 没有任何显式 launcher，依赖 Boot/PF自动启动。这能解释为什么前两个必须在 Java里构造参数，但还不能证明 objective “什么都不用做”——它必须由 PaaS启动参数补上，否则第二次运行会有相同空参数。我再核对 Boot 4自动启动的官方行为，然后给出明确结论。

## 助手

最关键的答案是：

> 不是因为 objective-batch 天生不需要 `run.id`，而是三个项目启动 Job 的责任方不同。partner-batch 和 out-batch 自己调用 `jobLauncher.run(...)`，所以必须自己构建 `JobParameters`；objective-batch 没有自行启动 Job，而是依赖 Boot/PF自动启动，因此参数应从 PaaS launcher 进入。

所以“objective-batch 什么都不用干”这个说法不够准确。准确说法是：

> objective-batch 可能不需要在 Java代码里构建 `run.id`，但整个项目仍必须确认或修改 PaaS启动参数。

## 三个项目的启动方式对比

| 项目 | 谁调用 `jobLauncher.run` | 当前参数来源 | 移除 incrementer 后谁负责唯一参数 |
|---|---|---|---|
| partner-batch | 自己的业务代码 | `JobParametersBuilder` | partner-batch Java代码 |
| out-batch | 自定义 `CommandLineRunner` | 当前是空 `JobParameters` | out-batch Java代码 |
| objective-batch | Boot/PF自动启动 | 应用启动参数/PaaS launcher | PaaS launcher，除非改成自定义 runner |

这不是 IAAS/PaaS 的区别，而是“谁拥有 Job 启动过程”的区别。

---

## 1. partner-batch 为什么必须在代码里构建 run ID

partner-batch 不是简单地在应用启动时自动运行一个 Job。

它在业务流程中自己调用：

```java
jobLauncher.run(
        jobInitializer.partnerJob(nameFile),
        jobParametersBuilder.toJobParameters());
```

位置：[PartnerBatchConfig.java](/<本机工作区>/partner-batch/src/main/java/com/bnpp/pf/swm/partners/batch/config/PartnerBatchConfig.java:197)

它已经自己创建参数：

```java
var jobParametersBuilder = new JobParametersBuilder();

jobParametersBuilder.addString(
        Constants.COUNTRY,
        country);

jobParametersBuilder.addDate(
        Constants.DATE,
        new Date());
```

然后根据文件名动态选择 Job：

```java
jobInitializer.partnerJob(nameFile)
```

也就是说 partner-batch 的控制流是：

```text
收到/发现文件
    ↓
识别国家
    ↓
根据文件名选择具体 Job
    ↓
自己构建 JobParameters
    ↓
自己调用 jobLauncher.run()
```

Spring Boot自动启动器根本没有参与这次 Job启动。

因此移除：

```java
.incrementer(new RunIdIncrementer())
```

以后，partner-batch 必须在自己已有的 `JobParametersBuilder` 中补充唯一参数，例如迁移代码可能是：

```java
jobParametersBuilder.addLong(
        "run.id",
        System.currentTimeMillis());
```

最终：

```java
jobLauncher.run(
        job,
        jobParametersBuilder.toJobParameters());
```

因为是 partner-batch 自己调用 `jobLauncher.run`，PaaS应用启动参数不会自动替它修改这个方法内部创建的 `JobParameters`。

### partner-batch 还有一个特殊点

它在一个循环中可能针对多个文件多次启动 Job：

```java
for (int i = 0; i < files.size(); i++) {
    jobParametersBuilder.addDate(
            Constants.DATE,
            new Date());

    jobLauncher.run(...);
}
```

并且 [JobInitializer.java](/<本机工作区>/partner-batch/src/main/java/com/bnpp/pf/swm/partners/batch/config/JobInitializer.java:30) 中多个不同流程使用相同的：

```java
Constants.PARTNER_JOB
```

所以每次文件处理必须有足够的 identifying 参数区分 JobInstance。

这也是为什么 partner-batch 的 run ID 通常在文件处理或 `runJob` 附近生成，而不是放到 `main` 方法里。

---

## 2. out-batch 为什么也必须自己构建，但写法不一样

out-batch 有一个自定义启动器：

```java
@Bean
public CommandLineRunner run() {
    return args -> {
        for (Job job : availableJobs) {
            jobLauncher.run(
                    job,
                    new JobParameters());
        }
    };
}
```

位置：[JobRunnerConfig.java](/<本机工作区>/out-batch/src/main/java/com/bnpp/pf/swm/out/batch/config/JobRunnerConfig.java:17)

它的控制流是：

```text
应用启动
    ↓
自定义 CommandLineRunner 被调用
    ↓
遍历 availableJobs
    ↓
自己调用 jobLauncher.run()
```

当前传入的是：

```java
new JobParameters()
```

也就是空参数。

旧版本依赖各 Job 上的：

```java
.incrementer(new RunIdIncrementer())
```

来产生下一实例参数。

移除 incrementer 后，如果继续传空参数：

```java
jobLauncher.run(job, new JobParameters());
```

那么第一次完成后，下一次应用启动仍然使用：

```text
同一个 Job名称 + 空 identifying 参数
```

因此可能出现：

```text
JobInstanceAlreadyCompleteException
```

所以 out-batch 必须把空参数替换成含唯一 ID 的参数。

它可能使用 builder：

```java
JobParameters jobParameters =
        new JobParametersBuilder()
                .addLong(
                        "run.id",
                        System.currentTimeMillis())
                .toJobParameters();
```

也可能直接使用 Spring Batch 6新的 immutable/record API构建：

```java
JobParameters jobParameters =
        new JobParameters(...);
```

两种写法形式不同，但目的相同：

```text
构造一个包含 identifying run.id 的 JobParameters
```

### 为什么 out-batch 和 partner-batch 方法不同

partner-batch 已经有：

```java
JobParametersBuilder
country
date
文件循环
动态 Job选择
```

所以最自然的是往现有 builder 加一个参数。

out-batch 原来只有：

```java
new JobParameters()
```

它没有现成的业务参数 builder，因此需要从空参数重新构造。

这是代码结构不同，不是 Spring Batch要求两个项目使用不同的 run ID算法。

最终它们都要得到相同语义：

```text
JobParameters
    └── run.id：Long、identifying、每次新运行唯一
```

---

## 3. objective-batch 为什么没有构建 JobParameters 的代码

objective-batch 的 main 只有：

```java
public static void main(String[] args) {
    SpringApplication.run(
            ObjectiveBatchApplication.class,
            args);
}
```

位置：[ObjectiveBatchApplication.java](/<本机工作区>/objective-batch/src/main/java/com/bnpp/pf/swm/objective/batch/ObjectiveBatchApplication.java:9)

它没有：

```java
JobLauncher
CommandLineRunner
ApplicationRunner
jobLauncher.run(...)
JobParametersBuilder
```

配置中又是：

```yaml
spring:
  batch:
    job:
      enabled: true
```

因此它依赖 Spring Boot/PF提供的自动 runner：

```text
PaaS启动应用
    ↓
SpringApplication.run(args)
    ↓
Boot/PF读取应用参数
    ↓
Boot/PF自动找到 Job
    ↓
转换启动参数为 JobParameters
    ↓
启动 Job
```

Spring Boot官方文档说明，在 Batch自动配置开启时，Job可以在应用启动时由 Boot自动执行；命令行中的非 `--` 参数会被转换成 JobParameter。[Spring Boot Batch documentation](https://docs.spring.io/spring-boot/4.0/reference/io/spring-batch.html)、[Boot Batch application arguments](https://docs.spring.io/spring-boot/how-to/batch.html)

因此 objective-batch 的职责原本就是：

```text
定义 Job和 Step
```

而不是：

```text
自己决定怎样启动 Job
```

这就是为什么不能机械复制 partner/out 的：

```java
new JobParametersBuilder()
jobLauncher.run(...)
```

到 objective-batch。

---

## 4. 但这不等于 objective-batch 不需要 run ID

Spring Boot自动 runner 不会凭空保证每次都有唯一 `run.id`。它主要是把传入的应用参数转换为 JobParameters。

如果 PaaS实际启动命令中有：

```text
run.id=1001,java.lang.Long
```

那么自动 runner 会把它传给 Job。

下一次：

```text
run.id=1002,java.lang.Long
```

会创建新 JobInstance。

这种情况下 objective-batch 不需要新增 Java代码。

但如果 PaaS启动时没有任何 JobParameter：

```text
java -jar objective-batch.jar
```

那么自动 runner得到的可能是空 JobParameters。

第一次执行：

```text
monthlyExtractionJob + 空参数
```

第二次执行仍然是：

```text
monthlyExtractionJob + 空参数
```

第二次就可能被认为是同一个已完成实例。

所以 objective-batch 不是“什么都不用做”，而是存在两种情况。

### PaaS已经提供唯一 identifying 参数

那么只需要：

```java
// 删除
.incrementer(new RunIdIncrementer())
```

不用新增 Java launcher。

### PaaS没有提供唯一 identifying 参数

那么必须选择一种解决办法：

1. 修改 PaaS launcher/Helm/CronJob参数，这是优先方案。
2. 如果平台无法提供，再考虑 objective-batch 自己实现 runner。

---

## 5. 为什么不建议直接给 objective-batch 加 `CommandLineRunner`

如果复制 out-batch 的方式，新增：

```java
@Bean
CommandLineRunner run(...) {
    return args -> jobLauncher.run(...);
}
```

但仍保留：

```yaml
spring.batch.job.enabled: true
```

那么可能发生：

```text
Boot自动 runner 启动一次
+
自定义 CommandLineRunner 又启动一次
```

也就是同一个应用启动两次 Job。

如果 objective-batch 真要自己管理 Job启动，就需要同时：

```yaml
spring:
  batch:
    job:
      enabled: false
```

然后自行解决：

- 两个 Job启动哪个。
- 是否两个都启动。
- 参数怎样从 PaaS传入。
- 新运行怎样生成唯一参数。
- 失败重启怎样复用原参数。
- Job执行顺序。
- 退出码。
- PaaS诊断参数。
- 重启与 Pod retry。

这显然比“让 PaaS launcher传入 run.id”改动大得多，不符合只做必需迁移的原则。

---

## 6. objective-batch 还有两个 Job 的问题

objective-batch 中定义了：

```text
monthlyExtractionJob
monthlyExtractionPartnerMdsJob
```

标准 Spring Boot行为是：如果 ApplicationContext 中有多个 Job，通常需要用：

```yaml
spring:
  batch:
    job:
      name: monthlyExtractionJob
```

或由启动参数指定要运行哪个 Job。

PF starter 可能提供自己的多 Job处理机制，但不能只凭源码推断。

因此 PaaS启动配置需要同时回答：

```text
这次启动哪个 Job？
传入什么 JobParameters？
```

例如概念上：

```text
Job：monthlyExtractionJob
run.id：1001
```

或者：

```text
Job：monthlyExtractionPartnerMdsJob
run.id：1001
```

两个 Job使用相同 `run.id` 不冲突，因为 Job名称不同。

---

## 7. 三个项目为什么不能统一复制同一段代码

因为三者的启动边界不同：

```text
partner-batch
业务事件/文件触发 → Java自己启动 Job

out-batch
应用启动 → 自定义 CommandLineRunner 启动 Job

objective-batch
应用启动 → Boot/PF自动 runner 启动 Job
```

`run.id` 必须在“真正创建 JobParameters 的地方”生成或进入。

所以：

- partner-batch：在已有 `JobParametersBuilder` 中处理。
- out-batch：在自定义 `CommandLineRunner` 中处理。
- objective-batch：在 PaaS启动参数中处理，而不是默认在 Java中处理。

它们方法不同是合理的，最终要求相同：

```text
新运行：
唯一 identifying 参数

失败重启：
复用原 identifying 参数
```

---

## 8. 还需要注意：Java里每次生成时间戳不一定天然正确

假设 partner/out 的迁移代码无条件执行：

```java
System.currentTimeMillis()
```

那么每次进程重启都会生成新参数。

对于全新运行，这是正确的。

但对于失败重启：

```text
第一次：run.id=1001，FAILED
Pod重启：run.id=1002
```

这会创建新 JobInstance，而不是重启 `1001`。

因此 partner/out 的迁移代码除了“生成新 ID”，还应具备以下一种能力：

- 如果启动方显式提供旧 run ID，则优先使用旧值。
- 或者失败重启通过 `JobOperator.restart(executionId)` 完成。
- 或者它们的业务失败模型明确不是 Spring Batch restart，而是将文件放回队列并作为全新业务实例重新处理。

没有看到你所说的最终迁移代码前，不能仅凭“加了 `System.currentTimeMillis()`”就判断重启语义一定正确。

---

## 9. 当前本地版本与您看到的迁移版本并不一致

本地工作区中：

- partner-batch 仍使用旧包和 `.incrementer(new RunIdIncrementer())`。
- out-batch 已把类 import 移到新包，但仍保留多个 `.incrementer(...)`。
- objective-batch 本地版本也仍是旧代码。

所以你说的 partner/out“已经新增构建 run ID 的迁移代码”应该在你 Mac 上的 `feature/migration-java` 分支或另一个版本中，并没有同步到当前 Windows工作区。

因此我现在可以准确解释三者为什么需要在不同位置处理，但不能对那两段新增方法逐行判断。

## 最终结论

我前面说“objective-batch 什么都不用干”是不严谨的。正确结论是：

```text
partner-batch：
Java自己启动 Job
→ Java必须构建 JobParameters/run ID

out-batch：
自定义 CommandLineRunner启动 Job
→ Java必须构建 JobParameters/run ID

objective-batch：
Boot/PF自动启动 Job
→ Java通常不构建 run ID
→ 但 PaaS launcher必须提供唯一 identifying 参数
```

如果 objective-batch 的 PaaS配置没有提供这种参数，那么迁移仍未完成；只是修改位置应该优先在 PaaS部署/调度配置，而不是复制 partner/out 的 Java实现。


## 用户

再给我讲一下你前面提到的：
4. 迁移 Spring Batch 数据库
   也是一点点循序渐进的讲

## 助手

我会先从“Spring Batch为什么需要自己的数据库表”讲起，再区分全新数据库和已有生产数据库，最后落到 objective-batch 现有的 Liquibase 文件、配置和上线顺序。重点也会解释：为什么不能直接重新执行 Batch 6 的完整建表脚本，以及 `BATCH_JOB_SEQ` 与前面讲的 `run.id` 完全不是同一件事。

## 助手

先给结论：

> Spring Batch数据库迁移不是迁移 objective、partner 等业务表，而是迁移 Spring Batch自己保存运行历史、重启位置、Step状态和 JobParameters 的元数据表。

对 PostgreSQL 来说，Spring Batch 6官方标准 schema 的直接 DDL变化非常小：

```sql
ALTER SEQUENCE BATCH_JOB_SEQ
RENAME TO BATCH_JOB_INSTANCE_SEQ;
```

但是实际迁移不能只看这一行 SQL。你还要判断：

- 数据库是全新的还是从 Batch 5升级。
- Batch schema 由应用 Liquibase 管理还是 DBA/部署流程管理。
- 当前 DB 到底处于哪个版本。
- 是否还有 Batch 5失败任务需要重启。
- PaaS滚动部署时会不会同时运行 Batch 5和Batch 6。
- objective-batch 当前的 Liquibase master 为什么是空的。

下面一点点讲。

---

## 1. Spring Batch 为什么需要数据库

你的业务表保存的是：

```text
ObjectiveEntity
PartnerEntity
PartnerMdsEntity
```

Spring Batch还需要保存“任务执行过程”，例如：

```text
这个 Job运行过几次？
哪次成功、哪次失败？
当前执行到哪个 Step？
已经读了多少条？
失败重启应该从哪里继续？
本次运行使用了什么 JobParameters？
```

这些信息保存在 Spring Batch metadata 表中，常见的有：

```text
BATCH_JOB_INSTANCE
BATCH_JOB_EXECUTION
BATCH_JOB_EXECUTION_PARAMS
BATCH_STEP_EXECUTION
BATCH_STEP_EXECUTION_CONTEXT
BATCH_JOB_EXECUTION_CONTEXT
```

以及用于生成内部数据库 ID 的 sequence。

概念关系是：

```text
BATCH_JOB_INSTANCE
    └── BATCH_JOB_EXECUTION
            ├── BATCH_JOB_EXECUTION_PARAMS
            ├── BATCH_JOB_EXECUTION_CONTEXT
            └── BATCH_STEP_EXECUTION
                    └── BATCH_STEP_EXECUTION_CONTEXT
```

### 各表大致保存什么

`BATCH_JOB_INSTANCE`：

```text
这是哪个逻辑 JobInstance
Job名称是什么
JobParameters计算出的 Job key是什么
```

`BATCH_JOB_EXECUTION`：

```text
某个 JobInstance 的一次执行尝试
开始时间
结束时间
状态：STARTED/FAILED/COMPLETED
```

`BATCH_JOB_EXECUTION_PARAMS`：

```text
这次执行使用的 run.id、日期、国家等参数
哪些参数是 identifying
```

`BATCH_STEP_EXECUTION`：

```text
某个 Step读了多少条
写了多少条
提交了多少次
跳过了多少条
状态是什么
```

`ExecutionContext` 表：

```text
用于保存 restart state
例如 Reader已经读到第几页、第几条
```

所以 metadata 不只是日志；它参与 Job身份判断和失败重启。

---

## 2. 它和业务数据库是什么关系

Spring Batch metadata 可以：

- 和业务表放在同一个 PostgreSQL database/schema。
- 放在同一个 database 的独立 schema。
- 使用独立 datasource/database。

objective-batch 中看不到显式独立 Batch DataSource，所以大概率由 PF/JCloud配置提供。

这需要在实际环境确认：

```text
BATCH_* 表在哪个 database？
在哪个 schema？
使用哪个数据库账号？
是否多个 tenant/schema 各有一套？
```

如果多个 schema 都有 Spring Batch metadata，那么每个包含旧 sequence 的 schema 都需要迁移，不能只迁默认 schema。

---

## 3. 公司 migration guide 的原文

PF 18指南写道：

> Spring Batch 6.0 has changed it's database schema.
>
> To update the Spring Batch database model, you can use `changelog-spring-batch-6-0-migration.yaml` Liquibase changelog that is available in Create a Batch > Spring batch datasource > Liquibase documentation section.

出处：[PF 18 guide 第344～348行](/<本机迁移指南目录>/18.0.0.md:344)。

公司要求是：

```text
使用 PF内部提供的
changelog-spring-batch-6-0-migration.yaml
```

这通常比自己手写 SQL更好，因为公司 changelog 可能还包含：

- PostgreSQL DBMS条件。
- sequence 是否存在的 precondition。
- Liquibase changeset ID/author。
- rollback。
- 公司 datasource/schema约定。

目前这个内部 changelog 不在 objective-batch 目录里，所以无法确认它的具体 precondition 和 rollback 内容。

---

## 4. Spring Batch官方说了什么

Spring Batch 6官方迁移指南对数据库 schema 的说明是：

> “The sequence `BATCH_JOB_SEQ` was renamed to `BATCH_JOB_INSTANCE_SEQ`”

[Spring Batch 6 official migration guide](https://github.com/spring-projects/spring-batch/wiki/Spring-Batch-6.0-Migration-Guide)

官方 PostgreSQL migration 文件只有一行：

```sql
ALTER SEQUENCE BATCH_JOB_SEQ
RENAME TO BATCH_JOB_INSTANCE_SEQ;
```

[官方 PostgreSQL migration SQL](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-core/src/main/resources/org/springframework/batch/core/migration/6.0/migration-postgresql.sql)

所以对标准 PostgreSQL schema 来说，没有：

- 删除 Batch history。
- 重建所有表。
- 复制所有数据。
- 修改业务表。
- 重置 sequence 数值。

核心只是重命名 sequence。

---

## 5. sequence 是什么

PostgreSQL sequence 用于生成递增数字。

例如：

```text
1
2
3
4
...
```

Spring Batch需要给每个 JobInstance 一个内部数据库 ID：

```text
JOB_INSTANCE_ID=100
JOB_INSTANCE_ID=101
JOB_INSTANCE_ID=102
```

在旧版本中，用于 JobInstance ID的 sequence 名称比较宽泛：

```text
BATCH_JOB_SEQ
```

Spring Batch 6把名称改得更明确：

```text
BATCH_JOB_INSTANCE_SEQ
```

意思是：

```text
这是专门生成 JobInstance内部 ID的 sequence
```

执行 `ALTER SEQUENCE ... RENAME` 后：

- sequence 中当前计数值不会重置。
- 已经存在的 JobInstance ID不会变化。
- 已有 metadata 不会被删除。
- 只是对象名称发生变化。
- Batch 6代码以后查找新名称。

---

## 6. 它和前面讲的 `run.id` 完全不同

这两个名字都与“ID”有关，很容易混淆。

### `run.id`

这是 JobParameter：

```text
run.id=1001
```

它可能参与 JobInstance身份计算：

```text
Job名称 + identifying run.id
```

它保存在：

```text
BATCH_JOB_EXECUTION_PARAMS
```

### `BATCH_JOB_INSTANCE_SEQ`

这是 PostgreSQL数据库内部 sequence：

```text
JOB_INSTANCE_ID=4711
```

用于给 `BATCH_JOB_INSTANCE` 表生成数据库主键。

对比：

| 项目 | `run.id` | `BATCH_JOB_INSTANCE_SEQ` |
|---|---|---|
| 类型 | JobParameter | PostgreSQL sequence |
| 谁提供 | launcher/scheduler | 数据库 |
| 作用 | 区分业务 JobInstance | 生成数据库内部主键 |
| 是否 identifying | 通常是 | 不适用 |
| 是否业务可见 | 可以 | 一般不可见 |
| Batch 6变化 | incrementer语义变化 | sequence 重命名 |

所以：

```sql
ALTER SEQUENCE ...
```

不会替你生成 `run.id`。

删除：

```java
RunIdIncrementer
```

也不会影响 PostgreSQL内部 sequence。

---

## 7. 为什么不能直接执行 Batch 6完整建表脚本

objective-batch 当前有这个文件：

[schema_spring_batch_metadata.yaml](/<本机工作区>/objective-batch/src/main/resources/db/changelog/v1/model/schema_spring_batch_metadata.yaml:1)

它执行：

```yaml
sqlFile:
  path: classpath:/org/springframework/batch/core/schema-postgresql.sql
```

这个 `schema-postgresql.sql` 是完整初始化脚本，会执行类似：

```sql
CREATE TABLE BATCH_JOB_INSTANCE ...
CREATE TABLE BATCH_JOB_EXECUTION ...
CREATE SEQUENCE ...
```

它适合：

```text
一个完全没有 Spring Batch metadata 的新数据库
```

不适合已有生产数据库。

如果已有 DB已经存在：

```text
BATCH_JOB_INSTANCE
BATCH_JOB_EXECUTION
...
```

再次执行完整 schema 会出现：

```text
relation BATCH_JOB_INSTANCE already exists
sequence already exists
```

更严重的是，如果有人为了让脚本成功，先 drop 表再重建，会丢失：

- 历史 JobExecution。
- 失败重启状态。
- Step ExecutionContext。
- 已完成实例记录。
- 防重复运行信息。

所以已有 DB必须执行“增量 migration”，不能重新执行完整 schema。

---

## 8. 全新数据库和已有数据库要分开

### 情况一：全新数据库

数据库中完全没有：

```text
BATCH_JOB_INSTANCE
BATCH_JOB_EXECUTION
BATCH_JOB_SEQ
BATCH_JOB_INSTANCE_SEQ
```

此时使用 PF 18/Spring Batch 6依赖中自带的：

```text
schema-postgresql.sql
```

它应直接创建：

```text
BATCH_JOB_INSTANCE_SEQ
```

不需要先创建旧的 `BATCH_JOB_SEQ` 再 rename。

正确流程：

```text
空数据库
    ↓
执行 Batch 6完整 schema
    ↓
直接得到 BATCH_JOB_INSTANCE_SEQ
```

### 情况二：已有 Batch 5数据库

已经存在：

```text
BATCH_JOB_SEQ
BATCH_JOB_INSTANCE
BATCH_JOB_EXECUTION
...
```

正确流程：

```text
已有 Batch 5 metadata
    ↓
保留所有表和数据
    ↓
执行 migration
    ↓
BATCH_JOB_SEQ 改名为 BATCH_JOB_INSTANCE_SEQ
```

### 情况三：数据库已经迁移过

已经存在：

```text
BATCH_JOB_INSTANCE_SEQ
```

并且不存在：

```text
BATCH_JOB_SEQ
```

说明 schema可能已经迁移。

这时再次执行 rename 会失败：

```text
relation BATCH_JOB_SEQ does not exist
```

所以 migration 必须只执行一次，或者公司 changelog 必须有 precondition。

---

## 9. 如何判断数据库当前状态

在 PostgreSQL中可以先只读查询 sequence。

概念查询：

```sql
SELECT schemaname, sequencename
FROM pg_sequences
WHERE upper(sequencename) IN (
    'BATCH_JOB_SEQ',
    'BATCH_JOB_INSTANCE_SEQ'
);
```

可能结果如下。

### 只有旧 sequence

```text
BATCH_JOB_SEQ
```

说明大概率是 Batch 5，需要迁移。

### 只有新 sequence

```text
BATCH_JOB_INSTANCE_SEQ
```

说明大概率已经迁移。

### 两个都没有

可能是：

- 全新数据库。
- metadata 在其他 schema。
- 使用了自定义 table prefix。
- 使用了不同 datasource。
- Batch metadata 根本不在这个 database。

不能立即执行 rename，要继续查 schema/datasource。

### 两个都存在

这不是标准迁移后的正常结果。

可能发生过：

- 手工创建新 sequence，但没有删除旧的。
- 执行了部分脚本。
- 不同版本应用各创建了一套对象。
- 两个 schema被 search path混在一起理解。

需要 DBA进一步检查，不能简单再次执行 migration。

---

## 10. objective-batch 当前是谁负责建表

`application.yaml` 中：

```yaml
spring:
  batch:
    jdbc:
      initialize-schema: never
```

位置：[application.yaml](/<本机工作区>/objective-batch/src/main/resources/application.yaml:10)

这表示：

> Spring Boot不能自动创建或更新 Spring Batch schema。

所以不能指望升级到 PF 18后，Boot自动把 sequence 改名。

数据库 schema责任只能在以下两者之一：

```text
Liquibase
或
外部 DBA/部署流程
```

---

## 11. objective-batch 的 Liquibase 配置

pprod 和 prod 中 Liquibase 开启：

```yaml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.yaml
```

文件：

- [application-pprod.yaml](/<本机工作区>/objective-batch/src/main/resources/application-pprod.yaml:30)
- [application-prod.yaml](/<本机工作区>/objective-batch/src/main/resources/application-prod.yaml:30)

dev 中关闭：

```yaml
spring:
  liquibase:
    enabled: false
```

位置：[application-dev.yaml](/<本机工作区>/objective-batch/src/main/resources/application-dev.yaml:31)

因此：

```text
mvn test / dev启动
```

通常不会自动验证这个 migration。

真正可能执行 Liquibase 的是：

```text
pprod
prod
```

---

## 12. 当前 master changelog 是空的

[db.changelog-master.yaml](/<本机工作区>/objective-batch/src/main/resources/db/changelog/db.changelog-master.yaml:1) 只有：

```yaml
databaseChangeLog:
```

它没有 include：

```yaml
v1/changelog-1.0.yaml
```

而 [v1/changelog-1.0.yaml](/<本机工作区>/objective-batch/src/main/resources/db/changelog/v1/changelog-1.0.yaml:1) 才 include 了：

```yaml
model/schema_spring_batch_metadata.yaml
```

目前引用关系是：

```text
application-prod.yaml
    ↓
db.changelog-master.yaml
    ↓
什么都没有
```

另一个文件虽然存在：

```text
v1/changelog-1.0.yaml
    ↓
schema_spring_batch_metadata.yaml
```

但它没有被 master 引用，因此从当前配置看是不可达的。

### 这意味着什么

有几种可能：

1. Batch schema 一直由 DBA/外部 pipeline 创建。
2. master 文件在其他部署仓库被覆盖。
3. 项目缺少 include。
4. v1 changelog 是历史遗留，实际未由应用执行。
5. 当前本地代码不是生产实际部署版本。

不能直接假设“把 v1 include 回去就好了”。

---

## 13. 为什么不能盲目把 v1 include 回 master

如果生产 DB表已经由 DBA创建，但 Liquibase 的 `DATABASECHANGELOG` 中没有记录这个 changeset，那么添加：

```yaml
- include:
    file: v1/changelog-1.0.yaml
```

后，Liquibase会认为：

```text
create-spring-batch-metadata 从未执行
```

然后尝试重新执行完整建表脚本。

结果可能是：

```text
table already exists
```

因此在修改 master 前必须先确认：

- 生产 DB是否已有 `BATCH_*` 表。
- `DATABASECHANGELOG` 是否记录 `create-spring-batch-metadata`。
- 初始 schema 是谁创建的。
- 迁移应该归应用还是 DBA负责。

这是数据库迁移中非常重要的 ownership 问题。

---

## 14. 如果由应用 Liquibase 负责

合理的结构概念上应是：

```text
db.changelog-master.yaml
    ├── 已有应用 schema changesets
    └── Spring Batch 6 migration changeset
```

例如公司提供的文件放在适当目录后，由 master include：

```yaml
databaseChangeLog:
  - include:
      file: changelog-spring-batch-6-0-migration.yaml
      relativeToChangelogFile: true
```

但这只是结构示例，实际不能直接照抄，因为需要确认：

- 公司文件实际存放位置。
- 它是否只用于 existing DB。
- 是否有 `sequenceExists` precondition。
- 是否支持 fresh DB。
- 是否限定 PostgreSQL。
- 是否有 schemaName。
- 是否有 rollback。
- changeset context是什么。

应该优先使用公司内部文档提供的原文件，而不是自己重写一个同名文件。

---

## 15. 如果由 DBA/部署流程负责

那么代码仓库中的 Liquibase 文件可以不改。

部署前由 DBA执行：

```sql
ALTER SEQUENCE BATCH_JOB_SEQ
RENAME TO BATCH_JOB_INSTANCE_SEQ;
```

或者由数据库发布 pipeline执行公司 changelog。

此时要在发布流程中明确顺序：

```text
停止旧 Batch任务
    ↓
确认没有 Batch 5执行正在写 metadata
    ↓
备份/确认恢复方案
    ↓
执行数据库 migration
    ↓
部署 PF 18应用
    ↓
验证 Batch 6运行
```

这是一项必须完成的迁移工作，但不一定表现为 Java或 resource文件修改。

---

## 16. 为什么必须先停旧版本

Batch 5代码寻找：

```text
BATCH_JOB_SEQ
```

Batch 6代码寻找：

```text
BATCH_JOB_INSTANCE_SEQ
```

执行 rename 后：

```text
Batch 5看不到 BATCH_JOB_SEQ
```

执行 rename 前：

```text
Batch 6看不到 BATCH_JOB_INSTANCE_SEQ
```

所以不能让 Batch 5和Batch6同时访问同一套 metadata。

这对 PaaS滚动部署尤其重要。

普通 rolling deployment 可能出现：

```text
旧 Pod：PF 17 / Batch 5
新 Pod：PF 18 / Batch 6
```

两者短时间同时存活。

数据库 sequence 不能同时满足它们的标准预期。因此迁移窗口应避免两个版本重叠，例如：

```text
先停止旧 Batch/Cron触发
执行 DB migration
再启动新版本
```

如果回滚 PF 18到旧版本，也要考虑把 sequence 名称回滚：

```sql
ALTER SEQUENCE BATCH_JOB_INSTANCE_SEQ
RENAME TO BATCH_JOB_SEQ;
```

回滚同样必须在没有 Job运行时进行。

---

## 17. 数据库权限也要确认

执行：

```sql
ALTER SEQUENCE ...
```

的数据库账号需要有足够权限，通常需要拥有该 sequence 或有相应 ALTER权限。

PaaS应用运行账号可能只能：

```text
SELECT
INSERT
UPDATE
DELETE
```

而不能：

```text
ALTER SEQUENCE
```

如果 Liquibase使用与应用相同的低权限账号，启动时 migration 会失败。

因此要确认：

- Liquibase datasource使用哪个账号。
- 账号是否拥有 `BATCH_JOB_SEQ`。
- 是否需要 DBA执行。
- sequence 是否在正确 schema。

---

## 18. 历史失败 Job必须在升级前处理

这与 sequence rename 是相关但独立的问题。

Spring Batch 6官方指出，JobParameter 的模型和序列化格式发生变化：

- Batch 5中的参数名称在外部 map key。
- Batch 6中参数名称进入新的 `JobParameter` record。
- 持久化/序列化格式不同。

因此：

> Batch 5启动并失败的 JobInstance，不能指望升级到 Batch 6后继续 restart。

正式迁移前需要查询失败 execution：

```text
FAILED
STOPPED
UNKNOWN
```

对需要继续执行的：

```text
在 Batch 5/PF 17环境重启成功
```

对不再需要的：

```text
abandon
```

不能以为执行 sequence rename 后，历史失败 Job就自动兼容 Batch 6。

sequence rename只解决数据库对象名称，不转换旧参数格式。

---

## 19. Liquibase 是怎样保证只执行一次的

Liquibase会创建自己的表：

```text
DATABASECHANGELOG
DATABASECHANGELOGLOCK
```

`DATABASECHANGELOG`记录：

```text
changeset id
author
文件路径
checksum
执行时间
```

例如：

```text
id: spring-batch-6-migration
author: pf
```

成功执行后，下次应用启动时 Liquibase发现它已经执行过，就不会再次 rename。

`DATABASECHANGELOGLOCK` 防止多个 PaaS Pod同时执行 migration。

但即使有 Liquibase lock，也不代表可以让 Batch 5和Batch6业务进程同时工作；lock只保护 Liquibase migration，不保护不同 Batch版本的运行兼容性。

---

## 20. 正确的迁移顺序

### 第一步：确定 database/schema

确认：

```text
objective-batch 的 BATCH_* 表在哪个 PostgreSQL database/schema
```

### 第二步：确定 schema owner

确认是：

```text
应用 Liquibase
```

还是：

```text
DBA/部署 pipeline
```

### 第三步：检查当前 sequence

确认属于以下哪种状态：

```text
旧：BATCH_JOB_SEQ
新：BATCH_JOB_INSTANCE_SEQ
空：全新 DB
异常：两者都有
```

### 第四步：处理 Batch 5失败任务

在旧版本中完成 restart 或 abandon。

### 第五步：停止 Batch 5运行

包括：

- CronJob调度。
- 手工启动。
- 正在运行的旧 Pod。
- 可能自动拉起的 retry。

### 第六步：执行 migration

优先：

```text
公司提供的 changelog-spring-batch-6-0-migration.yaml
```

或经 DBA批准执行官方 PostgreSQL SQL。

### 第七步：部署 PF 18

确保启动时看到：

```text
BATCH_JOB_INSTANCE_SEQ
```

### 第八步：验证

验证：

- 应用启动不报 sequence不存在。
- 能创建新的 JobInstance。
- `BATCH_JOB_INSTANCE` 有新记录。
- `BATCH_JOB_EXECUTION` 有新 execution。
- Step正常完成。
- 历史 metadata 仍存在。
- Liquibase changeset被记录。
- old sequence 不再被访问。

---

## 21. 推荐的验证场景

### 新运行

提供新的 identifying 参数：

```text
run.id=2001
```

验证：

```text
创建新的 BATCH_JOB_INSTANCE
创建新的 BATCH_JOB_EXECUTION
```

### 失败重启

让一个 PF 18 Job失败：

```text
run.id=2002
```

用相同参数或 `restart(executionId)` 重启。

验证：

```text
JobInstance数量不增加
JobExecution数量增加
```

### 新实例

使用：

```text
run.id=2003
```

验证 JobInstance数量增加。

这些测试同时验证：

- metadata schema正确。
- sequence正常。
- run.id身份逻辑正确。
- restart链路正确。

---

## 22. objective-batch 的最终判断

当前可以确定：

1. `initialize-schema: never`，Boot不会替你迁移。
2. pprod/prod 启用了 Liquibase。
3. Liquibase master 当前为空。
4. v1初始化 changelog 存在，但没有被 master引用。
5. 生产 DB大概率已经存在 Batch metadata，但需要实际确认。
6. PF 18要求已有 Batch 5数据库迁移 sequence。
7. 不能直接重新执行完整 `schema-postgresql.sql`。

因此现在不能直接断言“必须修改 master”或者“必须由 DBA执行”。必须先确认 schema ownership。

决策流程是：

```text
已有 Batch 5数据库？
    ├─ 否
    │   └─ 使用 Batch 6完整初始化 schema
    │
    └─ 是
        ├─ 应用 Liquibase管理？
        │   └─ include公司 Batch 6 migration changelog
        │
        └─ DBA/pipeline管理？
            └─ 发布前由DBA/pipeline执行 migration
```

无论选择哪条路径，最终数据库必须达到：

```text
存在：BATCH_JOB_INSTANCE_SEQ
不存在：BATCH_JOB_SEQ
历史 metadata 保留
PF 18可以创建新 JobInstance
```

这才算完成 Spring Batch数据库迁移。


## 用户

不懂liquibase里面的代码

## 助手

我先不继续讲数据库理论，直接把你项目里三个 Liquibase YAML 当作“程序入口、目录索引、具体动作”逐行拆开。也会先搜索公司提供的 `changelog-spring-batch-6-0-migration.yaml` 是否已经在本机；如果能找到，就按真实文件讲，不虚构内容。

## 助手

可以把 Liquibase 理解成一个“数据库版本升级执行器”。

它做的事情类似：

```text
读取升级目录
    ↓
找到还没执行过的数据库变更
    ↓
执行 SQL
    ↓
在数据库中登记“这个变更已经执行过”
    ↓
下次启动不再重复执行
```

下面直接用 objective-batch 的真实文件逐层解释。

## 1. 先看整体文件关系

你的项目中有：

```text
application-pprod.yaml / application-prod.yaml
    ↓ 指向
db/changelog/db.changelog-master.yaml
    ↓ 应该 include
db/changelog/v1/changelog-1.0.yaml
    ↓ include
db/changelog/v1/model/schema_spring_batch_metadata.yaml
    ↓ 执行
Spring Batch依赖包中的 schema-postgresql.sql
```

可以把它想象成一本书：

| 文件 | 作用 |
|---|---|
| `application-prod.yaml` | 告诉应用从哪本书开始读 |
| `db.changelog-master.yaml` | 总目录 |
| `v1/changelog-1.0.yaml` | 第1章目录 |
| `schema_spring_batch_metadata.yaml` | 具体操作说明 |
| `schema-postgresql.sql` | 真正执行的 SQL |

但你当前的问题是：

```text
总目录是空的
```

所以后面的 v1文件虽然存在，却不会被执行。

---

## 2. Spring Boot自动建表和 Liquibase 是两套机制

你有这个配置：

```yaml
spring:
  batch:
    jdbc:
      initialize-schema: never
```

位置：[application.yaml](/<本机工作区>/objective-batch/src/main/resources/application.yaml:10)

意思是：

> 不允许 Spring Boot自动创建 Spring Batch表。

Spring Boot本来可以根据配置自动执行：

```text
schema-postgresql.sql
```

但现在被明确关闭了：

```yaml
initialize-schema: never
```

所以数据库表必须由别的机制负责。

这里“别的机制”可能是：

```text
Liquibase
```

或者：

```text
DBA/数据库部署流水线
```

两套机制不要混淆：

```text
spring.batch.jdbc.initialize-schema
    → Spring Boot自动初始化

spring.liquibase
    → Liquibase数据库版本管理
```

objective-batch 关闭了第一套，但在 pprod/prod 开启了第二套。

---

## 3. `application-prod.yaml` 中的 Liquibase 配置

生产配置：

```yaml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.yaml
```

位置：[application-prod.yaml](/<本机工作区>/objective-batch/src/main/resources/application-prod.yaml:30)

逐行解释。

### `spring.liquibase`

```yaml
spring:
  liquibase:
```

表示下面的配置都属于 Spring Boot的 Liquibase模块。

### `enabled: true`

```yaml
enabled: true
```

表示应用启动时启用 Liquibase。

执行顺序大致是：

```text
应用启动
    ↓
连接数据库
    ↓
运行 Liquibase
    ↓
Liquibase完成后继续创建其他 Spring Bean
    ↓
启动 Spring Batch Job
```

所以正常情况下，数据库升级应在 Job访问 metadata 表之前完成。

### `change-log`

```yaml
change-log: classpath:db/changelog/db.changelog-master.yaml
```

表示 Liquibase的入口文件是：

```text
db/changelog/db.changelog-master.yaml
```

### `classpath:` 是什么意思

`classpath:` 不是电脑上的绝对路径。

它表示：

> 在应用运行时 classpath 中寻找这个文件。

编译打包后，`src/main/resources` 中的内容会进入应用 JAR，例如概念上：

```text
objective-batch.jar
└── BOOT-INF/classes
    └── db
        └── changelog
            └── db.changelog-master.yaml
```

Liquibase会从 JAR内部读取它。

---

## 4. dev 和 prod 的区别

dev 中：

```yaml
spring:
  liquibase:
    enabled: false
```

位置：[application-dev.yaml](/<本机工作区>/objective-batch/src/main/resources/application-dev.yaml:31)

所以使用 dev profile 启动时：

```text
Liquibase完全不运行
```

pprod/prod 中：

```yaml
enabled: true
```

所以：

```text
pprod/prod 会运行 Liquibase
```

这意味着只执行：

```bash
mvn test
```

或者使用 dev profile 本地启动，通常不能验证生产 Liquibase migration。

---

## 5. `db.changelog-master.yaml` 是什么

当前文件：

```yaml
databaseChangeLog:
```

位置：[db.changelog-master.yaml](/<本机工作区>/objective-batch/src/main/resources/db/changelog/db.changelog-master.yaml:1)

这是 Liquibase YAML 的根节点。

完整的 master 通常类似：

```yaml
databaseChangeLog:
  - include:
      file: v1/changelog-1.0.yaml
      relativeToChangelogFile: true

  - include:
      file: v2/changelog-2.0.yaml
      relativeToChangelogFile: true
```

可以理解为：

```text
databaseChangeLog:
    下面列出所有需要按顺序执行的数据库版本
```

但当前只有：

```yaml
databaseChangeLog:
```

下面没有任何内容，等价于：

```text
Liquibase已启用
但是没有任何数据库变更需要执行
```

所以当前 pprod/prod 启动时，Liquibase会打开 master，但找不到 changeset，然后直接结束。

---

## 6. YAML 中的 `-` 是什么意思

例如：

```yaml
databaseChangeLog:
  - include:
      file: v1/changelog-1.0.yaml
```

这里的：

```yaml
-
```

表示列表中的一项。

可以类比 Java：

```java
List<Change> changes = List.of(
    include("v1/changelog-1.0.yaml"),
    include("v2/changelog-2.0.yaml")
);
```

所以：

```yaml
databaseChangeLog:
  - include: ...
  - include: ...
```

表示按顺序处理两个 include。

YAML非常依赖缩进。下面两段含义可能不同，因此空格不能随意调整。

---

## 7. `include` 是什么

[v1/changelog-1.0.yaml](/<本机工作区>/objective-batch/src/main/resources/db/changelog/v1/changelog-1.0.yaml:1) 内容：

```yaml
databaseChangeLog:
  - include:
      file: model/schema_spring_batch_metadata.yaml
      relativeToChangelogFile: true
```

逐行解释。

### `include`

```yaml
- include:
```

表示：

> 不在当前文件直接写数据库操作，而是去读取另一个 changelog 文件。

类似 Java：

```java
import anotherFile;
```

或者目录：

```text
请继续读取下一份文件
```

### `file`

```yaml
file: model/schema_spring_batch_metadata.yaml
```

指定被 include 的文件。

### `relativeToChangelogFile: true`

```yaml
relativeToChangelogFile: true
```

表示相对当前 changelog 的位置解析路径。

当前文件是：

```text
db/changelog/v1/changelog-1.0.yaml
```

所以：

```text
model/schema_spring_batch_metadata.yaml
```

会解析成：

```text
db/changelog/v1/model/schema_spring_batch_metadata.yaml
```

如果这里是：

```yaml
relativeToChangelogFile: false
```

Liquibase会从 classpath根目录或其他配置位置解释路径，而不是从当前文件所在目录开始。

---

## 8. 当前 master 为什么没有执行 v1

你有：

```text
db.changelog-master.yaml
```

也有：

```text
v1/changelog-1.0.yaml
```

但 master 中没有：

```yaml
- include:
    file: v1/changelog-1.0.yaml
    relativeToChangelogFile: true
```

所以执行链在第一步就中断：

```text
application-prod.yaml
    ↓
db.changelog-master.yaml
    ↓
没有 include
    ↓
结束
```

`v1/changelog-1.0.yaml` 不会因为文件存在就自动执行。

Liquibase不会自动扫描整个目录；它只执行从入口文件能逐级引用到的 changelog。

---

## 9. `changeSet` 是什么

[schema_spring_batch_metadata.yaml](/<本机工作区>/objective-batch/src/main/resources/db/changelog/v1/model/schema_spring_batch_metadata.yaml:1) 内容：

```yaml
databaseChangeLog:
  - changeSet:
      id: create-spring-batch-metadata
      context: prod
      author: david
      changes:
        - sqlFile:
            encoding: UTF-8
            path: classpath:/org/springframework/batch/core/schema-postgresql.sql
            relativeToChangelogFile: false
            splitStatements: true
            stripComments: true
```

`changeSet` 是 Liquibase的最小执行单元。

可以理解为：

```text
一次有名字、作者和执行记录的数据库变更
```

例如：

```text
创建Batch metadata表
增加一个字段
创建一个索引
重命名一个sequence
```

都可以各自作为一个 changeset。

---

## 10. `id` 和 `author`

```yaml
id: create-spring-batch-metadata
author: david
```

Liquibase主要通过下面的组合识别一个 changeset：

```text
文件路径 + id + author
```

例如：

```text
文件：
db/changelog/v1/model/schema_spring_batch_metadata.yaml

id：
create-spring-batch-metadata

author：
david
```

这三个组合在一起代表一个唯一变更。

第一次运行时，Liquibase执行它，然后在数据库的：

```text
DATABASECHANGELOG
```

表中登记。

下一次启动，Liquibase看到同一个：

```text
文件 + id + author
```

已经执行过，就不会再执行。

因此不要通过修改旧 changeset 的 `id` 来强迫它重新运行。

---

## 11. Liquibase 自己的两个数据库表

Liquibase通常会在数据库中维护：

```text
DATABASECHANGELOG
DATABASECHANGELOGLOCK
```

### `DATABASECHANGELOG`

记录已经执行过什么：

```text
id
author
文件路径
执行时间
checksum
执行顺序
```

概念记录：

```text
create-spring-batch-metadata
david
db/changelog/v1/model/schema_spring_batch_metadata.yaml
已执行
```

### `DATABASECHANGELOGLOCK`

防止两个应用实例同时修改数据库。

例如 PaaS 同时启动两个 Pod：

```text
Pod A：获得 Liquibase lock，执行 migration
Pod B：等待 lock
```

Pod A完成后释放 lock，Pod B发现 changeset 已执行，就不再重复执行。

---

## 12. 为什么不应该修改已经执行过的 changeset

Liquibase会计算 changeset checksum。

如果生产数据库已经记录：

```text
create-spring-batch-metadata
checksum=ABC
```

你修改同一个 changeset 内容后，新 checksum 可能变成：

```text
checksum=XYZ
```

Liquibase可能报：

```text
Validation Failed
Checksum was ABC but is now XYZ
```

因此数据库版本管理通常遵循：

```text
旧 changeset不修改
新增一个 changeset处理下一次变化
```

比如：

```text
v1：创建 Batch 5 schema
v2：迁移到 Batch 6 schema
```

而不是把 v1直接改成 Batch 6 migration。

---

## 13. `context: prod` 是什么

```yaml
context: prod
```

它给 changeset打了一个 Liquibase context 标签。

概念上：

```text
只有启用了 prod context 时才希望执行这个 changeset
```

但它不是 Spring profile 的自动等价物。

也就是说：

```text
spring.profiles.active=prod
```

不一定自动意味着：

```text
Liquibase context=prod
```

通常需要类似：

```yaml
spring:
  liquibase:
    contexts: prod
```

你当前配置里没有看到：

```yaml
spring.liquibase.contexts
```

当没有显式 context filter 时，Liquibase通常会允许执行带 context 的 changeset。

不过当前 master 没有 include 这个文件，所以现在 `context: prod` 实际上还没机会产生影响。

---

## 14. `changes` 是什么

```yaml
changes:
```

表示这个 changeset 要执行哪些数据库操作。

例如可以有：

```yaml
changes:
  - createTable: ...
  - addColumn: ...
  - createIndex: ...
  - sql: ...
  - sqlFile: ...
```

你的 changeset 使用的是：

```yaml
- sqlFile:
```

意思是：

> 真正的 SQL不直接写在 YAML里，而是从另一个 SQL文件读取。

---

## 15. `sqlFile` 的每一行

```yaml
sqlFile:
  encoding: UTF-8
  path: classpath:/org/springframework/batch/core/schema-postgresql.sql
  relativeToChangelogFile: false
  splitStatements: true
  stripComments: true
```

### `encoding`

```yaml
encoding: UTF-8
```

表示用 UTF-8读取 SQL文件。

### `path`

```yaml
path: classpath:/org/springframework/batch/core/schema-postgresql.sql
```

这个 SQL文件不在 objective-batch 仓库中。

它位于 Spring Batch依赖 JAR 中，概念路径类似：

```text
spring-batch-core-6.x.jar
└── org
    └── springframework
        └── batch
            └── core
                └── schema-postgresql.sql
```

升级依赖后，同一个 classpath路径可能指向新版本 JAR里的脚本：

```text
PF 旧版本 → Spring Batch 5的 schema-postgresql.sql
PF 18      → Spring Batch 6的 schema-postgresql.sql
```

### `relativeToChangelogFile: false`

```yaml
relativeToChangelogFile: false
```

因为 SQL不在当前：

```text
v1/model
```

目录下，而是在整个应用 classpath 中。

所以不能从当前 YAML所在目录相对查找。

### `splitStatements: true`

```yaml
splitStatements: true
```

表示 Liquibase根据 SQL结束符把文件拆成多条 SQL执行。

例如：

```sql
CREATE TABLE A (...);

CREATE TABLE B (...);

CREATE SEQUENCE C;
```

会被拆成三条 statement。

### `stripComments: true`

```yaml
stripComments: true
```

表示执行前去掉 SQL注释，例如：

```sql
-- create batch table
```

不会作为 SQL内容发送给数据库。

---

## 16. 这个 changeset 真正做了什么

如果执行链能够到达它：

```text
master
    ↓
v1/changelog-1.0.yaml
    ↓
schema_spring_batch_metadata.yaml
```

它最终会执行 Spring Batch完整的 PostgreSQL建表脚本。

也就是创建：

```text
BATCH_JOB_INSTANCE
BATCH_JOB_EXECUTION
BATCH_JOB_EXECUTION_PARAMS
BATCH_STEP_EXECUTION
BATCH_STEP_EXECUTION_CONTEXT
BATCH_JOB_EXECUTION_CONTEXT
各种 sequence
```

这是一份“初始化脚本”，不是“升级脚本”。

适合：

```text
数据库里完全没有 Batch metadata
```

不适合：

```text
已有 Batch 5 metadata，需要保留历史数据
```

---

## 17. Batch 6 migration changelog 应该是什么角色

公司要求的：

```text
changelog-spring-batch-6-0-migration.yaml
```

不是完整建表脚本。

它应该承担的是：

```text
把已经存在的 Batch 5数据库改成 Batch 6需要的状态
```

对于标准 PostgreSQL，核心 SQL是：

```sql
ALTER SEQUENCE BATCH_JOB_SEQ
RENAME TO BATCH_JOB_INSTANCE_SEQ;
```

它不会重新创建所有表。

概念上的 Liquibase changeset可能类似：

```yaml
databaseChangeLog:
  - changeSet:
      id: migrate-spring-batch-5-to-6
      author: pf
      changes:
        - sql:
            sql: >
              ALTER SEQUENCE BATCH_JOB_SEQ
              RENAME TO BATCH_JOB_INSTANCE_SEQ;
```

注意：这是为了帮助理解的简化示例，不是公司文件的真实内容。实际应使用公司文档提供的原文件，因为它可能包含 precondition、schema、rollback等额外处理。

---

## 18. `include` migration 后会怎样

假设公司 migration 文件放在：

```text
db/changelog/v2/changelog-spring-batch-6-0-migration.yaml
```

master 概念上可以引用：

```yaml
databaseChangeLog:
  - include:
      file: v2/changelog-spring-batch-6-0-migration.yaml
      relativeToChangelogFile: true
```

应用启动时：

```text
读取 master
    ↓
发现 v2 include
    ↓
读取 migration changeset
    ↓
查询 DATABASECHANGELOG
    ↓
没执行过
    ↓
执行 ALTER SEQUENCE
    ↓
登记到 DATABASECHANGELOG
```

下次启动：

```text
读取 master
    ↓
发现 v2
    ↓
DATABASECHANGELOG显示已经执行
    ↓
跳过
```

---

## 19. 为什么不能同时无条件 include 初始化和 migration

假设全新 PF 18数据库：

1. 先执行 Batch 6完整 schema。
2. 完整 schema 直接创建：

```text
BATCH_JOB_INSTANCE_SEQ
```

3. 然后再执行 migration：

```sql
ALTER SEQUENCE BATCH_JOB_SEQ ...
```

但全新数据库中不存在旧的：

```text
BATCH_JOB_SEQ
```

migration 就会失败。

因此必须区分：

```text
全新数据库
    → 执行完整 Batch 6 schema

已有 Batch 5数据库
    → 执行 Batch 6 migration
```

或者公司 migration changelog 内部必须有 precondition：

```text
只有 BATCH_JOB_SEQ存在时才执行
```

这也是为什么必须取得公司提供的真实 YAML，不能只根据文件名自己猜。

---

## 20. `preConditions` 是什么

虽然当前项目文件里没有，但公司 migration 很可能使用。

它的意思是：

```text
执行 changeset 前先检查数据库状态
```

概念上：

```text
如果旧 sequence存在
并且新 sequence不存在
    → 执行 rename
否则
    → 跳过、标记完成或报错
```

这样可以避免：

```text
已经迁移过却再次执行
全新数据库没有旧 sequence却执行 rename
数据库状态异常还继续执行
```

不同 `onFail` 策略可能是：

```text
HALT：立即停止部署
MARK_RAN：不执行，但记作已完成
CONTINUE：继续处理其他 changeset
```

具体使用哪个必须看公司原文件。

---

## 21. objective-batch 当前代码翻译成人话

当前配置整体等价于：

```text
在 dev：
    不运行 Liquibase
    Spring Boot也不自动建 Batch表

在 pprod/prod：
    启动 Liquibase
    Liquibase打开 master
    但 master 是空的
    所以 Liquibase什么都不做

项目中虽然有 v1初始化文件：
    但 master没有引用它
    因此它不会运行
```

所以目前真正的 Batch表很可能是：

- 历史上由 DBA创建。
- 由外部部署 pipeline创建。
- 由旧版本曾经执行过。
- 或者当前本地资源并非生产实际版本。

必须先确认这一点。

---

## 22. 现在不要直接做什么

不要在没有检查数据库前直接把下面内容加入 master：

```yaml
- include:
    file: v1/changelog-1.0.yaml
```

因为如果生产已经有 `BATCH_*` 表，但 Liquibase不知道 v1执行过，它会尝试重新建表。

也不要直接修改旧文件：

```yaml
id: create-spring-batch-metadata
```

把它变成 migration。旧 changeset 应保持不变。

---

## 23. 你现在只需要先弄清三个问题

### 问题一

生产数据库是否已经存在：

```text
BATCH_JOB_INSTANCE
BATCH_JOB_EXECUTION
BATCH_JOB_SEQ
```

### 问题二

数据库中的：

```text
DATABASECHANGELOG
```

是否有：

```text
create-spring-batch-metadata
```

这能判断 Liquibase是否曾执行过项目中的初始化 changeset。

### 问题三

公司提供的：

```text
changelog-spring-batch-6-0-migration.yaml
```

真实内容是什么，它要求放在哪个目录、如何 include。

本机目前没有这个文件，只有 migration guide 对它的引用。

## 最简单的记忆方式

```text
application-prod.yaml
    = Liquibase开关和入口地址

db.changelog-master.yaml
    = 总目录

v1/changelog-1.0.yaml
    = 版本目录

changeSet
    = 一次数据库变更

sqlFile
    = 真正要执行的SQL文件

DATABASECHANGELOG
    = Liquibase的执行历史

Batch 6 migration changelog
    = 在保留历史数据的情况下，把Batch 5数据库升级到Batch 6
```

当前 objective-batch 最重要的事实是：Liquibase虽然在 pprod/prod 开启，但 master 是空的，所以现有 changelog 链不会执行任何 SQL。


## 用户

把上个对话上传到我的github，创建一个public的repo名叫wuswei，然后把上个对话以md文件的格式上传

## 助手

我会使用 GitHub 发布技能来创建公开仓库 `wuswei`，把本任务中用户与助手可见的对话整理成 Markdown 后提交并推送；不会包含系统提示、开发者指令、内部推理或任何未显示给你的内容。这个操作会在你的 GitHub 账户创建一个公开仓库。

## 助手

GitHub 发布技能要求的 `gh` CLI 在当前机器上未安装。我会先检查已连接的 GitHub 应用是否提供“创建仓库/上传文件”能力；如果连接器也不支持创建新仓库，这项工作会停在需要安装并登录 `gh` 的位置。

## 助手

我继续确认 GitHub 连接器的具体能力，重点看它能否直接创建 public repository；如果可以，就不依赖本机缺失的 `gh`。公开上传前，我会把对话中的本机绝对路径和用户名做脱敏，不包含系统指令或内部推理。

## 助手

GitHub 连接器可以上传文件，但没有“创建仓库”接口；本机又没有 `gh`。为了不中断，我会尝试使用已登录的内置浏览器在 GitHub 页面创建仓库，再用连接器上传 Markdown。这个浏览器操作只执行你明确要求的建库和上传。

## 助手

浏览器目前没有登录 GitHub，而且这是唯一可用的浏览器。仓库暂时无法创建；我先把可公开的 Markdown 整理好，并确认 GitHub 连接器对应的账号与仓库是否已存在，这些都是只读或本地准备工作。

## 助手

已经确认连接器账号是 `Diaaago`，且 `Diaaago/wuswei` 尚不存在。创建仓库只差网页登录；与此同时我会尝试从本机当前会话记录生成尽量完整的逐轮 Markdown，而不是凭记忆缩写。公开版本会排除系统/开发者指令并脱敏本机路径。


## 助手

GitHub 登录页已打开。请登录账号 **Diaaago**，然后回复我“已登录”。

登录后我会：

1. 创建 public 仓库 `Diaaago/wuswei`。
2. 从本机对话记录生成 Markdown。
3. 排除系统指令、内部推理，并脱敏绝对路径。
4. 上传并返回仓库链接。

## 用户

## My request for Codex:
已登录

## 助手

我现在检查登录后的 GitHub 建仓页面，创建 `Diaaago/wuswei` 为 public；成功后会通过已连接的 GitHub 账号提交脱敏后的 Markdown 对话记录。

## 助手

仓库已经创建成功并确认是 **Public**。现在我开始生成并提交 `conversation.md`：保留用户与助手可见的迁移讨论，图片只记录为“附件未上传”，删除环境注入信息、系统内容和本机绝对路径。

## 助手

第一次生成时发现单次读取会截断长对话，因此我不会上传残缺文件。我会按对话轮次分段提取、分别校验，再合并成一个 Markdown 后提交；同时补充清理所有正斜杠形式的本机路径。

## 用户

把这次对话的上传就行。给我详细解答第5点management的设置


## 助手

第 5 点的最小迁移结论是：从 `application-dev.yaml`、`application-pprod.yaml` 和 `application-prod.yaml` 删除下面整个旧 workaround；如果 `management` 下没有别的配置，就连 `management` 一起删除。

```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          exclude: "readinessState"
```

它等价于属性 `management.endpoint.health.group.readiness.exclude=readinessState`。它没有关闭 Actuator，也没有关闭整个 health endpoint；它只把 ID 为 `readinessState` 的健康指示器从名为 `readiness` 的健康组中排除。

在 PF Spring 17 以前，readiness 组会装入所有可用健康指示器。在 Kubernetes 环境中，Spring Boot 又会加入 `readinessState`。PF Batch 在真正运行 Job 前执行诊断时，应用状态可能仍是 `REFUSING_TRAFFIC`，于是诊断会被自身的 readiness 状态挡住。旧配置排除 `readinessState`，是为了解开这个循环。

PF Spring 17 把两个目的拆开了：标准 `readiness` 组默认只包含 `readinessState`，用于表达实例是否可以接收流量；新的 `readiness-batch` 组包含除 `readinessState` 之外的可用健康指示器，用于 Batch 启动前检查数据库等依赖。因此旧 exclusion 留在 PF 18 中，会把标准 readiness 组唯一应保留的状态删掉，使探针不再正确表达应用可用性。

不要把旧配置机械移动成 `readiness-batch.exclude=readinessState`：PF 已经这样定义默认值，重复配置不是迁移必需项。只有项目过去对 readiness 做过真正的 Batch 诊断定制（例如特意排除某个外部依赖），才把那项定制迁到 `readiness-batch`。

还要区分几个相似属性：`management.endpoint.health.group...` 配置 health endpoint 内部的健康组；`management.endpoints.web.exposure...` 决定哪些 Actuator endpoints 通过 HTTP 暴露；`management.endpoint.health.probes.enabled` 控制 Kubernetes probe health groups；`management.endpoint.health.probes.add-additional-paths` 决定是否在主端口增加 `/livez`、`/readyz`。它们不是这次删除旧 exclusion 的替代项。

验证时应确认标准 readiness 响应中能看到 `readinessState`，而 `readiness-batch` 不包含它；Batch 启动日志应显示启动前诊断使用 Batch 专用健康组。Spring Boot 标准路径是 `/actuator/health/readiness`，PF/Helm 环境可能使用公司约定的 `/diagnostic/actuators/health/readiness`。
