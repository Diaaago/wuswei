## 1. 数据库表是什么

可以把一张数据库表理解成 Excel 表格。

例如 `TB_CATALOG`：

| CODE | MARKET | TYPE    | DESCRIPTION    |
| ---- | ------ | ------- | -------------- |
| A001 | FR     | PRODUCT | France product |
| A002 | IT     | SERVICE | Italy service  |

这里：

- 表：`TB_CATALOG`
- 列：`CODE`、`MARKET`、`TYPE`
- 行：一条具体数据，例如 `A001 / FR / PRODUCT`

Java 中的 `CatalogEntity`，就是用来表示其中一行数据的类。

```
@Entity
@Table(name = "TB_CATALOG")
public class CatalogEntity {
}
```

含义是：

```
CatalogEntity 对应数据库中的 TB_CATALOG 表
```

每个 Java 字段再对应数据库的一列：

```
@Column(name = "MARKET")
private String market;
```

意思是：

```
Java 的 market 字段 ↔ 数据库的 MARKET 列
```

## 2. 主键是什么

数据库必须能准确区分每一行。

例如：

| CODE | MARKET |
| ---- | ------ |
| A001 | FR     |
| A002 | IT     |

这里可以用 `CODE` 唯一识别每一行，所以 `CODE` 是主键。

Java 代码中用 `@Id` 表示：

```
@Id
@Column(name = "CODE")
private String code;
```

这里两个注解分工不同：

```
@Id
    表示这个字段是实体主键

@Column(name = "CODE")
    表示它对应数据库的 CODE 列
```

因此，真正让 `CODE` 成为主键的是 `@Id`。

这点非常重要，因为它能解释为什么删除 `@PrimaryKeyJoinColumn` 后，`CODE` 仍然是主键。

## 3. 外键是什么

外键用于表示“两张表之间的关系”。

例如员工表：

| EMPLOYEE_ID | NAME  | DEPARTMENT_ID |
| ----------- | ----- | ------------- |
| 1           | Alice | 100           |
| 2           | Bob   | 200           |

部门表：

| DEPARTMENT_ID | NAME    |
| ------------- | ------- |
| 100           | IT      |
| 200           | Finance |

员工表的 `DEPARTMENT_ID` 指向部门表的 `DEPARTMENT_ID`：

```
EMPLOYEE.DEPARTMENT_ID
        ↓
DEPARTMENT.DEPARTMENT_ID
```

员工表中的 `DEPARTMENT_ID` 就是外键。

它表达的不是“这一行是谁”，而是：

```
这一行和另一张表的哪一行有关系
```

## 4. Java 继承和数据库表的关系

Java 支持继承：

```
class Animal {
    private String name;
}

class Dog extends Animal {
    private String color;
}
```

`Dog` 自动拥有 `name` 和 `color`。

但是数据库没有完全一样的 Java 继承机制。JPA/Hibernate 必须决定：

```
Animal 和 Dog 的字段应该存在哪些表里？
```

其中有两种相关的处理方式。

### 方式一：父类没有自己的表

例如父类只是提供公共字段：

```
@MappedSuperclass
public abstract class AbstractEntity {

    private String tenantId;
}
```

子类：

```
@Entity
@Table(name = "TB_CATALOG")
public class CatalogEntity extends AbstractEntity {

    @Id
    private String code;
}
```

数据库里只有一张表：

| CODE | TENANT_ID | MARKET |
| ---- | --------- | ------ |
| A001 | tenant-1  | FR     |

也就是说：

```
AbstractEntity 没有自己的数据库表
它的字段直接放进 TB_CATALOG
```

这种情况下没有父表，所以也不需要把 `TB_CATALOG` 连接到父表。

### 方式二：父类和子类各有一张表

例如：

```
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public class BaseEntity {

    @Id
    private String id;
}
```

子类：

```
@Entity
@PrimaryKeyJoinColumn(name = "CODE")
public class CatalogEntity extends BaseEntity {
}
```

数据库可能有两张表。

父表：

| ID   | TENANT_ID |
| ---- | --------- |
| A001 | tenant-1  |

子表：

| CODE | MARKET |
| ---- | ------ |
| A001 | FR     |

这两张表通过同一个值 `A001` 连接：

```
BASE_ENTITY.ID = TB_CATALOG.CODE
```

此时，`TB_CATALOG.CODE` 同时承担两个角色：

```
1. 它是 TB_CATALOG 自己的主键
2. 它也是指向父表 BASE_ENTITY.ID 的外键
```

这才是 `@PrimaryKeyJoinColumn` 的用途。

## 5. 这次错误具体在说什么

你当前大概是这样的代码：

```
@Entity
@Table(name = "TB_CATALOG")
@PrimaryKeyJoinColumn(name = "CODE")
public class CatalogEntity extends AbstractEntity {

    @Id
    @Column(name = "CODE")
    private String code;
}
```

`@PrimaryKeyJoinColumn` 告诉 Hibernate：

```
CatalogEntity 是一个 JOINED 继承体系中的子实体。

请使用 TB_CATALOG.CODE，
去连接父实体所在表的主键。
```

但是迁移到新的 jcloud `AbstractEntity` 后，Hibernate 检查发现：

```
CatalogEntity 不是 JOINED 继承体系中的子实体
```

也就是说，从 Hibernate 的角度看：

```
没有需要连接的父实体表
```

代码却仍然要求：

```
用 CODE 连接父实体表
```

这两件事发生矛盾，于是 Hibernate 报错：

```
CatalogEntity may not specify a @PrimaryKeyJoinColumn
```

翻译成比较容易理解的话就是：

> `CatalogEntity` 当前不是需要通过主键连接父表的子实体，所以不能在它上面使用 `@PrimaryKeyJoinColumn`。

## 6. 为什么换父类以后才报错

迁移前后，Java 表面上可能只是改了一个父类：

```
// 迁移前
extends AbstractBaseEntity
```

改成：

```
// 迁移后
extends AbstractEntity
```

但这两个父类的数据库映射方式可能不一样。

迁移前的父类可能参与了实体继承，或者旧版本 Hibernate 没有严格拒绝这个注解。

迁移后的父类不再形成 `JOINED` 实体关系，因此 `@PrimaryKeyJoinColumn` 没有了使用条件。新版本 Hibernate 在启动时检查到它，直接终止启动。

## 7. 为什么删除它不会让 CODE 失去主键

删除的是：

```
@PrimaryKeyJoinColumn(name = "CODE")
```

保留的是：

```
@Id
@Column(name = "CODE")
private String code;
```

两者作用完全不同：

| 注解                    | 作用                                 |
| ----------------------- | ------------------------------------ |
| `@Id`                   | 声明 `CODE` 是当前表的主键           |
| `@Column`               | 声明 Java 字段对应数据库的 `CODE` 列 |
| `@PrimaryKeyJoinColumn` | 声明这个主键还要连接父实体表         |

因此删除后：

```
CODE 仍然是主键
但不再尝试通过 CODE 连接一个不存在的父实体表
```

最终应当是：

```
@Entity
@Table(name = "TB_CATALOG")
public class CatalogEntity extends AbstractEntity {

    @Id
    @Column(name = "CODE")
    private String code;

    // 其他字段
}
```

## 8. 这次不应该做什么

不应该为了保留旧注解而添加：

```
@Inheritance(strategy = InheritanceType.JOINED)
```

因为这可能要求数据库出现一张新的父表，改变原来的数据库结构。

也不应该修改 jcloud 的 `AbstractEntity`，因为你的迁移范围是 `out-batch`，而且这会影响其他项目。

当前最小且符合现有数据库结构的迁移是：

```
 @Entity
 @Table(name = "TB_CATALOG")
-@PrimaryKeyJoinColumn(name = "CODE")
 public class CatalogEntity extends AbstractEntity {
```

然后保留：

```
@Id
@Column(name = "CODE")
private String code;
```

一句话总结：

> `@Id` 表示“CODE 是本表主键”；`@PrimaryKeyJoinColumn` 表示“CODE 还要连接父实体表”。迁移后没有这样的父实体表关系，所以只需要删除后者，不能删除 `@Id`。







----







## 1. 数据库表是什么

可以把一张数据库表理解成 Excel 表格。

例如 `TB_CATALOG`：

| CODE | MARKET | TYPE    | DESCRIPTION    |
| ---- | ------ | ------- | -------------- |
| A001 | FR     | PRODUCT | France product |
| A002 | IT     | SERVICE | Italy service  |

这里：

- 表：`TB_CATALOG`
- 列：`CODE`、`MARKET`、`TYPE`
- 行：一条具体数据，例如 `A001 / FR / PRODUCT`

Java 中的 `CatalogEntity`，就是用来表示其中一行数据的类。

```
@Entity
@Table(name = "TB_CATALOG")
public class CatalogEntity {
}
```

含义是：

```
CatalogEntity 对应数据库中的 TB_CATALOG 表
```

每个 Java 字段再对应数据库的一列：

```
@Column(name = "MARKET")
private String market;
```

意思是：

```
Java 的 market 字段 ↔ 数据库的 MARKET 列
```

## 2. 主键是什么

数据库必须能准确区分每一行。

例如：

| CODE | MARKET |
| ---- | ------ |
| A001 | FR     |
| A002 | IT     |

这里可以用 `CODE` 唯一识别每一行，所以 `CODE` 是主键。

Java 代码中用 `@Id` 表示：

```
@Id
@Column(name = "CODE")
private String code;
```

这里两个注解分工不同：

```
@Id
    表示这个字段是实体主键

@Column(name = "CODE")
    表示它对应数据库的 CODE 列
```

因此，真正让 `CODE` 成为主键的是 `@Id`。

这点非常重要，因为它能解释为什么删除 `@PrimaryKeyJoinColumn` 后，`CODE` 仍然是主键。

## 3. 外键是什么

外键用于表示“两张表之间的关系”。

例如员工表：

| EMPLOYEE_ID | NAME  | DEPARTMENT_ID |
| ----------- | ----- | ------------- |
| 1           | Alice | 100           |
| 2           | Bob   | 200           |

部门表：

| DEPARTMENT_ID | NAME    |
| ------------- | ------- |
| 100           | IT      |
| 200           | Finance |

员工表的 `DEPARTMENT_ID` 指向部门表的 `DEPARTMENT_ID`：

```
EMPLOYEE.DEPARTMENT_ID
        ↓
DEPARTMENT.DEPARTMENT_ID
```

员工表中的 `DEPARTMENT_ID` 就是外键。

它表达的不是“这一行是谁”，而是：

```
这一行和另一张表的哪一行有关系
```

## 4. Java 继承和数据库表的关系

Java 支持继承：

```
class Animal {
    private String name;
}

class Dog extends Animal {
    private String color;
}
```

`Dog` 自动拥有 `name` 和 `color`。

但是数据库没有完全一样的 Java 继承机制。JPA/Hibernate 必须决定：

```
Animal 和 Dog 的字段应该存在哪些表里？
```

其中有两种相关的处理方式。

### 方式一：父类没有自己的表

例如父类只是提供公共字段：

```
@MappedSuperclass
public abstract class AbstractEntity {

    private String tenantId;
}
```

子类：

```
@Entity
@Table(name = "TB_CATALOG")
public class CatalogEntity extends AbstractEntity {

    @Id
    private String code;
}
```

数据库里只有一张表：

| CODE | TENANT_ID | MARKET |
| ---- | --------- | ------ |
| A001 | tenant-1  | FR     |

也就是说：

```
AbstractEntity 没有自己的数据库表
它的字段直接放进 TB_CATALOG
```

这种情况下没有父表，所以也不需要把 `TB_CATALOG` 连接到父表。

### 方式二：父类和子类各有一张表

例如：

```
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public class BaseEntity {

    @Id
    private String id;
}
```

子类：

```
@Entity
@PrimaryKeyJoinColumn(name = "CODE")
public class CatalogEntity extends BaseEntity {
}
```

数据库可能有两张表。

父表：

| ID   | TENANT_ID |
| ---- | --------- |
| A001 | tenant-1  |

子表：

| CODE | MARKET |
| ---- | ------ |
| A001 | FR     |

这两张表通过同一个值 `A001` 连接：

```
BASE_ENTITY.ID = TB_CATALOG.CODE
```

此时，`TB_CATALOG.CODE` 同时承担两个角色：

```
1. 它是 TB_CATALOG 自己的主键
2. 它也是指向父表 BASE_ENTITY.ID 的外键
```

这才是 `@PrimaryKeyJoinColumn` 的用途。

## 5. 这次错误具体在说什么

你当前大概是这样的代码：

```
@Entity
@Table(name = "TB_CATALOG")
@PrimaryKeyJoinColumn(name = "CODE")
public class CatalogEntity extends AbstractEntity {

    @Id
    @Column(name = "CODE")
    private String code;
}
```

`@PrimaryKeyJoinColumn` 告诉 Hibernate：

```
CatalogEntity 是一个 JOINED 继承体系中的子实体。

请使用 TB_CATALOG.CODE，
去连接父实体所在表的主键。
```

但是迁移到新的 jcloud `AbstractEntity` 后，Hibernate 检查发现：

```
CatalogEntity 不是 JOINED 继承体系中的子实体
```

也就是说，从 Hibernate 的角度看：

```
没有需要连接的父实体表
```

代码却仍然要求：

```
用 CODE 连接父实体表
```

这两件事发生矛盾，于是 Hibernate 报错：

```
CatalogEntity may not specify a @PrimaryKeyJoinColumn
```

翻译成比较容易理解的话就是：

> `CatalogEntity` 当前不是需要通过主键连接父表的子实体，所以不能在它上面使用 `@PrimaryKeyJoinColumn`。

## 6. 为什么换父类以后才报错

迁移前后，Java 表面上可能只是改了一个父类：

```
// 迁移前
extends AbstractBaseEntity
```

改成：

```
// 迁移后
extends AbstractEntity
```

但这两个父类的数据库映射方式可能不一样。

迁移前的父类可能参与了实体继承，或者旧版本 Hibernate 没有严格拒绝这个注解。

迁移后的父类不再形成 `JOINED` 实体关系，因此 `@PrimaryKeyJoinColumn` 没有了使用条件。新版本 Hibernate 在启动时检查到它，直接终止启动。

## 7. 为什么删除它不会让 CODE 失去主键

删除的是：

```
@PrimaryKeyJoinColumn(name = "CODE")
```

保留的是：

```
@Id
@Column(name = "CODE")
private String code;
```

两者作用完全不同：

| 注解                    | 作用                                 |
| ----------------------- | ------------------------------------ |
| `@Id`                   | 声明 `CODE` 是当前表的主键           |
| `@Column`               | 声明 Java 字段对应数据库的 `CODE` 列 |
| `@PrimaryKeyJoinColumn` | 声明这个主键还要连接父实体表         |

因此删除后：

```
CODE 仍然是主键
但不再尝试通过 CODE 连接一个不存在的父实体表
```

最终应当是：

```
@Entity
@Table(name = "TB_CATALOG")
public class CatalogEntity extends AbstractEntity {

    @Id
    @Column(name = "CODE")
    private String code;

    // 其他字段
}
```

## 8. 这次不应该做什么

不应该为了保留旧注解而添加：

```
@Inheritance(strategy = InheritanceType.JOINED)
```

因为这可能要求数据库出现一张新的父表，改变原来的数据库结构。

也不应该修改 jcloud 的 `AbstractEntity`，因为你的迁移范围是 `out-batch`，而且这会影响其他项目。

当前最小且符合现有数据库结构的迁移是：

```
 @Entity
 @Table(name = "TB_CATALOG")
-@PrimaryKeyJoinColumn(name = "CODE")
 public class CatalogEntity extends AbstractEntity {
```

然后保留：

```
@Id
@Column(name = "CODE")
private String code;
```

一句话总结：

> `@Id` 表示“CODE 是本表主键”；`@PrimaryKeyJoinColumn` 表示“CODE 还要连接父实体表”。迁移后没有这样的父实体表关系，所以只需要删除后者，不能删除 `@Id`。



----







对，照片里的迁移代码已经存在，而且 master 也正确 include 了它：

```
db.changelog-master.yaml
  → v2/changelog-spring-batch-6-0-migration.yaml
  → BATCH_JOB_SEQ 重命名为 BATCH_JOB_INSTANCE_SEQ
```

所以问题不是“没写迁移代码”，而是“这段代码没有对你当前连接的数据库真正生效”。

## 最可能的原因：local 禁用了 Liquibase

你之前的 `application-local.yaml` 中有：

```
spring:
  liquibase:
    enabled: false
```

这意味着本地启动过程是：

```
读取 application-local.yaml
  ↓
发现 Liquibase enabled=false
  ↓
完全不读取 db.changelog-master.yaml
  ↓
v2 的 renameSequence 不执行
  ↓
数据库仍只有旧的 BATCH_JOB_SEQ
  ↓
Batch 6 查找 BATCH_JOB_INSTANCE_SEQ
  ↓
报错
```

“项目里有 YAML 文件”不等于“数据库已经执行了这个 YAML”。必须由启用状态下的 Liquibase 连接数据库并执行它。

`spring.batch.jdbc.initialize-schema` 和 Liquibase 也是两套不同机制：

```
spring.batch.jdbc.initialize-schema
    用于初始化 Spring Batch 自带 schema

spring.liquibase.enabled
    决定项目 db.changelog 中的 changeSet 是否执行
```

`initialize-schema` 不会主动执行你的 v2 Liquibase 文件。

## 先确认 changeSet 有没有执行

在当前连接的数据库执行：

```
SELECT id, author, filename, exectype
FROM databasechangelog
WHERE id = 'model-rename-sequence-batch-job-seq-to-batch-job-instance-seq';
```

然后检查 sequence：

```
SELECT schemaname, sequencename
FROM pg_sequences
WHERE lower(sequencename) IN (
    'batch_job_seq',
    'batch_job_instance_seq'
);
```

### 结果一：DATABASECHANGELOG 查不到记录

并且数据库里只有：

```
batch_job_seq
```

说明迁移文件存在，但从未对这个数据库执行。

如果这是完全隔离的本地数据库，可以临时启用：

```
spring:
  liquibase:
    enabled: true
```

再执行：

```
mvn clean spring-boot:run -Dspring-boot.run.profiles=local
```

如果你的 `local` 实际连接的是公司 QVAL/共享数据库，不要擅自启用；应该让这条 changeSet 通过正常数据库部署流程执行。

### 结果二：显示 `EXECTYPE = MARK_RAN`

这一点要特别注意。照片中的 changeSet 使用了：

```
onFail: MARK_RAN
```

它的意思是：

```
如果旧的 BATCH_JOB_SEQ 存在
    → 执行重命名

如果旧的 BATCH_JOB_SEQ 不存在或当前 schema 看不到它
    → 不执行重命名
    → 但仍把 changeSet 标记为已经处理
```

所以可能出现：

```
DATABASECHANGELOG 里有记录
但 BATCH_JOB_INSTANCE_SEQ 仍然不存在
```

这时要继续检查：

```
SHOW search_path;
```

以及：

```
SELECT schemaname, sequencename
FROM pg_sequences
WHERE lower(sequencename) LIKE '%batch_job%seq%';
```

可能旧 sequence 在另一个 schema，导致 `sequenceExists` 没看到它，然后被 `MARK_RAN` 跳过。

共享环境下如果已经错误地 `MARK_RAN`，不要直接删除 `DATABASECHANGELOG` 记录；应新增一个修正 changeSet。隔离的本地数据库则可以重新初始化。

### 结果三：显示 `EXECTYPE = EXECUTED`

但新 sequence 仍不存在，通常表示：

- changeSet 在另一个 schema 执行；
- 应用当前连接的数据库不是执行迁移的那个数据库；
- sequence 后来被删除；
- 当前 datasource 的 `search_path` 不包含该 schema。

要对比：

```
SELECT current_database(), current_schema();
SHOW search_path;
```

## 照片里的 YAML 还有一个需要核对的地方

照片看起来写的是：

```
preConditions:
  onFail: MARK_RAN
  sequenceExists:
    sequenceName: BATCH_JOB_SEQ
```

Liquibase 当前官方 YAML 示例使用列表形式：

```
preConditions:
  - onFail: MARK_RAN
  - sequenceExists:
      sequenceName: BATCH_JOB_SEQ
```

[Liquibase 官方 preconditions 语法](https://docs.liquibase.com/community/user-guide-5-0-2/what-are-preconditions)

因此，如果实际文件确实没有两个 `-`，建议按项目所用 Liquibase 版本验证。否则启用 Liquibase 后可能出现 YAML 解析错误。不过这不是当前“sequence 不存在”的直接原因，因为当前看起来 Liquibase根本没有运行。

## 当前最准确的结论

迁移代码本身已经存在；现在需要确认的是下面哪一种：

```
A. local 禁用了 Liquibase，changeSet 从未执行
B. precondition 失败后被 MARK_RAN 跳过
C. changeSet 在其他数据库/schema 执行
```

从你当前配置看，A 是最可能的原因。先运行那两条只读 SQL，就能马上确定是哪一种。







---









这说明问题不只是 `enabled=false`。改成 `true` 后仍报同一个错误，只可能是：

1. Liquibase 实际没有启动；
2. Liquibase 启动了，但这个 changeSet 被 `MARK_RAN` 跳过；
3. migration 在另一个数据库/schema 执行了。

从日志来看，应用最终出现了：

```
Started OutBatchApplication
```

说明 Liquibase没有因为执行 SQL 失败而终止启动。如果 rename 真正执行成功，`batch_job_instance_seq` 就应该存在。因此更可能是“没有执行”或“被跳过”。

## 第一步：确认项目运行时有没有 Liquibase

在公司项目根目录执行：

```
mvn dependency:tree -Dincludes=org.liquibase:liquibase-core
```

正常应该看到：

```
org.liquibase:liquibase-core:jar:...
```

如果没有任何输出，说明 POM 中没有 Liquibase 运行库。此时：

```
spring:
  liquibase:
    enabled: true
```

只是一个没人读取的配置，Spring Boot 不会报错，也不会执行 changelog。

需要在 POM 中明确添加：

```
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

不写版本，由 PF Spring 18 的依赖管理控制。

然后：

```
mvn clean spring-boot:run -Dspring-boot.run.profiles=local
```

启动日志中应该能看到类似：

```
liquibase
Reading from databasechangelog
Running Changeset
UPDATE SUMMARY
```

## 第二步：检查是否被 `MARK_RAN` 跳过

如果依赖树里存在 Liquibase，在当前数据库执行：

```
SELECT id, author, filename, exectype, dateexecuted
FROM databasechangelog
WHERE id =
'model-rename-sequence-batch-job-seq-to-batch-job-instance-seq';
```

同时执行：

```
SELECT schemaname, sequencename
FROM pg_sequences
WHERE lower(sequencename) IN (
    'batch_job_seq',
    'batch_job_instance_seq'
);
```

### 如果 `EXECTYPE` 是 `MARK_RAN`

这就是最关键的原因。

你的 changeSet 是：

```
preConditions:
  onFail: MARK_RAN
  sequenceExists:
    sequenceName: BATCH_JOB_SEQ
```

运行过程可能是：

```
Liquibase 查找 BATCH_JOB_SEQ
  ↓
在当前 schema 中没找到
  ↓
precondition 失败
  ↓
因为 onFail=MARK_RAN
  ↓
不执行 renameSequence
  ↓
却把 changeSet 记为已经处理
  ↓
应用启动
  ↓
Batch 6 找不到 BATCH_JOB_INSTANCE_SEQ
```

所以：

```
Liquibase 启用了
```

不等于：

```
renameSequence 执行了
```

### 如果查询不到 changeSet 记录

说明 Liquibase仍没有执行这份 master changelog。继续检查环境变量是否覆盖了 YAML：

```
printenv | rg -i 'SPRING_LIQUIBASE|LIQUIBASE'
```

特别检查：

```
SPRING_LIQUIBASE_ENABLED=false
```

还要在启动日志中搜索：

```
LiquibaseAutoConfiguration
liquibase
databasechangelog
```

如果完全没有 Liquibase 日志，通常是依赖不存在或自动配置没有启用。

### 如果 `EXECTYPE` 是 `EXECUTED`

但 sequence 仍不存在，则检查实际数据库和 schema：

```
SELECT current_database(), current_schema();
SHOW search_path;
```

以及：

```
SELECT schemaname, sequencename
FROM pg_sequences
WHERE lower(sequencename) LIKE '%batch%seq%';
```

这通常表示 migration 在某个 schema 中执行了，但 Spring Batch 正在另一个 schema 中查找。

## 第三步：根据 sequence 查询结果处理

### 只有 `batch_job_seq`

对隔离的本地数据库，可以执行官方迁移：

```
ALTER SEQUENCE BATCH_JOB_SEQ
RENAME TO BATCH_JOB_INSTANCE_SEQ;
```

共享数据库不要手工执行，应通过 Liquibase部署。

### 两个都不存在

rename changeSet 无法解决，因为没有旧 sequence 可以重命名。需要初始化完整的 Spring Batch 6 metadata schema。

### 新 sequence 存在于其他 schema

不要创建第二个。需要修正 datasource 的 schema/search path，让应用访问正确位置。

## 为什么最后还是 `BUILD SUCCESS`

错误仍然在 `JobRunnerConfig.java:37` 附近被捕获：

```
catch (Exception exception) {
    exception.printStackTrace();
}
```

异常没有重新抛出，所以 Maven显示：

```
BUILD SUCCESS
```

但 Job 仍然没有开始执行业务步骤。

当前最优先执行的是：

```
mvn dependency:tree -Dincludes=org.liquibase:liquibase-core
```

和：

```
SELECT id, exectype
FROM databasechangelog
WHERE id =
'model-rename-sequence-batch-job-seq-to-batch-job-instance-seq';
```

这两个结果可以直接确定是“Liquibase根本没运行”，还是“被 `MARK_RAN` 跳过”。







-----

## objective

现在是：

> 应用启动成功、Maven 显示 `BUILD SUCCESS`，但两个 Batch Job 都没有执行。

因为你把：

```
spring:
  batch:
    job:
      enabled: false
```

设置成了 `false`。

## 发生了什么

之前：

```
enabled: true
```

Spring Boot 自动寻找要运行的 Job，但项目有两个：

```
monthlyExtractionJob
monthlyExtractionPartnerMdsJob
```

它不知道选择哪个，所以报错：

```
Job name must be specified in case of multiple jobs
```

现在改成：

```
enabled: false
```

相当于告诉 Spring Boot：

> 不要自动运行任何 Job。

因此流程变成：

```
加载数据库和 JPA
    ↓
Spring ApplicationContext 启动成功
    ↓
不启动任何 Batch Job
    ↓
正常关闭
    ↓
BUILD SUCCESS
```

截图里的这些内容证明应用框架启动成功：

```
Started ObjectiveBatchApplication
BUILD SUCCESS
```

但截图中没有看到正常 Job 执行时应有的：

```
Job launched
JobExecution
Step execution
monthlyExtraction...
```

所以这只是“应用能启动”，不是“Batch 业务运行成功”。

## 为什么日志里的 exit code 是 0

截图中有：

```
Exit code for batch jobs executed during Spring startup is zero
```

这里的 `0` 只表示 Spring 启动过程没有失败。

由于自动 Job 启动被关闭了，不能把它理解成：

```
两个 Job 都成功处理了数据
```

更准确地说是：

```
没有 Job 执行，也没有 Job 执行失败
```

## 如果你想实际运行第一个 Job

改成：

```
spring:
  batch:
    job:
      enabled: true
      name: monthlyExtractionJob
```

启动时只运行：

```
monthlyExtractionJob
```

## 如果想运行第二个 Job

使用：

```
spring:
  batch:
    job:
      enabled: true
      name: monthlyExtractionPartnerMdsJob
```

只运行：

```
monthlyExtractionPartnerMdsJob
```

这里的名字来自代码中的两个 Job：

```
.get("monthlyExtractionJob", jobRepository)
```

以及：

```
.get("monthlyExtractionPartnerMdsJob", jobRepository)
```

## 更适合部署的方式

如果调度器每次决定运行哪个 Job，不建议在 YAML 中永久写死名字。

配置中保持：

```
spring:
  batch:
    job:
      enabled: true
```

启动第一个时传：

```
--spring.batch.job.name=monthlyExtractionJob
```

启动第二个时传：

```
--spring.batch.job.name=monthlyExtractionPartnerMdsJob
```

这样同一个 jar 可以根据启动参数选择不同 Job。

## 如果一次启动必须运行两个 Job

那就不能靠现在这个自动 runner，因为它要求在多个 Job 中明确选择一个。

需要确认原来的业务要求：

- 每次只运行一个 Job：使用 `spring.batch.job.name`。
- 两个 Job 分别由调度器启动：调度器分别传两个名字。
- 一次启动必须依次执行两个 Job：需要一个明确的编排方式，例如一个 Job 包含两个 Step，或者自定义 runner 显式启动两个 Job。

不能用：

```
enabled: false
```

作为最终解决办法，除非项目中另有代码负责调用：

```
jobLauncher.run(...)
```

objective-batch 当前没有看到这样的自定义启动器，所以保持 `false` 很可能导致生产环境完全不跑业务。

另外，你现在只改了 `application-local.yaml`。如果 dev、pprod、prod 仍然是：

```
enabled: true
```

但没有提供：

```
name: ...
```

部署到这些环境时还会出现原来的“多个 Job 未指定名称”错误。

一句话总结：

> 现在解决的是“应用无法启动”，方法是关闭所有 Job；但“应该运行哪个 Job”这个真正的问题还没有解决。最终应该恢复 `enabled: true`，并通过配置或启动参数明确指定 Job。