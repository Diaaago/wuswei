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