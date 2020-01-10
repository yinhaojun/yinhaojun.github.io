---
title: 使用 Room 实体定义数据
date: 2020-01-07 15:48:57
tags:
- Android
- Jetpack
- Room
---

使用 **Room**时，您可以将相关字段集定义为实体。对于每个实体，系统会在关联的 `Database`对象中创建一个表来存储这些项。您必须通过 `Database`类中的 `entities`数组引用实体类。

以下代码段展示了如何定义实体：

```java
	@Entity
    public class User {
        @PrimaryKey
        public int id;

        public String firstName;
        public String lastName;
    }
```

要保留某个字段，Room 必须拥有该字段的访问权限。您可以将某个字段设为*public*，也可以为其提供 *getter* 和 *setter*。

<!-- more -->

## 使用主键

每个实体必须将至少 1 个字段定义为主键。即使只有 1 个字段，您仍然需要为该字段添加 **@PrimaryKey**注释。此外，如果您想让 Room 为实体分配自动 ID，则可以设置 **@PrimaryKey** 的 *autoGenerate*属性。如果实体具有复合主键，您可以使用 **@Entity** 注释的 *primaryKeys* 属性，如以下代码段所示：

```java
    @Entity(primaryKeys = {"firstName", "lastName"})
    public class User {
        public String firstName;
        public String lastName;
    }
```

默认情况下，Room 将类名称用作数据库表名称。如果您希望表具有不同的名称，请设置 **@Entity** 注释的 *tableName* 属性，如以下代码段所示：

```java
    @Entity(tableName = "users")
    public class User {
        // ...
    }
```

***注意：SQLite 中的表名称不区分大小写。***

与 **tableName** 属性类似，Room 将字段名称用作数据库中的列名称。如果您希望列具有不同的名称，请将 **@ColumnInfo** 注释添加到字段，如以下代码段所示：

```java
	@Entity(tableName = "users")
    public class User {
        @PrimaryKey
        public int id;

        @ColumnInfo(name = "first_name")
        public String firstName;

        @ColumnInfo(name = "last_name")
        public String lastName;
    }

```

## 忽略字段

默认情况下，Room 会为在实体中定义的每个字段创建一个列。如果某个实体中有您不想保留的字段，则可以使用 **@Ignore** 为这些字段注释，如以下代码段所示：

```java
	@Entity
    public class User {
        @PrimaryKey
        public int id;

        public String firstName;
        public String lastName;

        @Ignore
        Bitmap picture;
    }
```

如果实体继承了父实体的字段，则使用 @Entity 属性的 *ignoredColumns* 属性通常会更容易：

```java
    @Entity(ignoredColumns = "picture")
    public class RemoteUser extends User {
        @PrimaryKey
        public int id;

        public boolean hasVpn;
    }
```

## 视图

2.1.0 及更高版本的 Room 为 SQLite 数据库视图提供了支持，从而允许您将查询封装到类中。Room 将这些查询支持的类称为视图，在 DAO 中使用时，它们的行为与简单数据对象的行为相同。

***注意：与实体类似，您可以针对视图运行 SELECT 语句。不过，您无法针对视图运行 INSERT、UPDATE 或 DELETE 语句。***

### 创建视图

要创建视图，请将 `@DatabaseView` 注释添加到类中。将注释的值设为类应该表示的查询。

以下代码段提供了一个视图示例：

```
    @DatabaseView("SELECT user.id, user.name, user.departmentId," +
                  "department.name AS departmentName FROM user " +
                  "INNER JOIN department ON user.departmentId = department.id")
    public class UserDetail {
        public long id;
        public String name;
        public long departmentId;
        public String departmentName;
    }
```

### 将视图与数据库相关联

要将此视图添加为应用数据库的一部分，请在应用的 `@Database` 注释中添加 `views` 属性：

```
    @Database(entities = {User.class}, views = {UserDetail.class},version = 1)
    public abstract class AppDatabase extends RoomDatabase {
        public abstract UserDao userDao();
    }
```

## 提供表搜索支持

### 支持全文搜索

如果您的应用需要通过全文搜索 (FTS) 快速访问数据库信息，请使用虚拟表（使用 FTS3 或 FTS4 SQLite [扩展模块](https://www.sqlite.org/fts3.html)）为您的实体提供支持。要使用 Room 2.1.0 及更高版本中提供的这项功能，请将 **@Fts3** 或 **@Fts4** 注释添加到给定实体，如以下代码段所示：

```java
	// Use `@Fts3` only if your app has strict disk space requirements or if you
    // require compatibility with an older SQLite version.
    @Fts4
    @Entity(tableName = "users")
    public class User {
        // Specifying a primary key for an FTS-table-backed entity is optional, but
        // if you include one, it must use this type and column name.
        @PrimaryKey
        @ColumnInfo(name = "rowid")
        public int id;

        @ColumnInfo(name = "first_name")
        public String firstName;
    }

```

***注意：启用 FTS 的表始终使用 INTEGER 类型的主键且列名称为“rowid”。如果是由 FTS 表支持的实体定义主键，则必须使用相应的类型和列名称。***

如果表支持以多种语言显示的内容，请使用 `languageId` 选项指定用于存储每一行语言信息的列：

```java
	@Fts4(languageId = "lid")
    @Entity(tableName = "users")
    public class User {
        // ...

        @ColumnInfo(name = "lid")
        int languageId;
    }
```

Room 提供了其他几个选项来定义由 FTS 支持的实体，包括结果排序、令牌生成器类型以及作为外部内容管理的表。

### 将特定列编入索引

如果您的应用必须支持不允许使用由 FTS3 或 FTS4 表支持的实体的 SDK 版本，您仍可以将数据库中的某些列编入索引，以加快查询速度。要为实体添加索引，请在 `@Entity `注释中添加` indices` 属性，以列出要在索引或复合索引中包含的列的名称。以下代码段演示了此注释过程：

```java
    @Entity(indices = {@Index("name"),
            @Index(value = {"last_name", "address"})})
    public class User {
        @PrimaryKey
        public int id;

        public String firstName;
        public String address;

        @ColumnInfo(name = "last_name")
        public String lastName;

        @Ignore
        Bitmap picture;
    }
```

有时，数据库中的某些字段或字段组必须是唯一的。您可以通过将` @Index` 注释的 `unique` 属性设为 true 来强制实施此唯一性属性。以下代码示例可防止表格具有包含 firstName 和 lastName 列的同一组值的两行：

```java
    @Entity(indices = {@Index(value = {"first_name", "last_name"},
            unique = true)})
    public class User {
        @PrimaryKey
        public int id;

        @ColumnInfo(name = "first_name")
        public String firstName;

        @ColumnInfo(name = "last_name")
        public String lastName;

        @Ignore
        Bitmap picture;
    }
```







