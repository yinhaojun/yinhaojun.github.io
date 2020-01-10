---
title: 使用 Room DAO 访问数据
date: 2020-01-07 20:26:37
tags:
- Android
- Jetpack
- Room
---

要使用 Room 访问应用的数据，您需要使用数据访问对象 (DAO)。这些Dao对象构成了Room的主要组件，因为每个DAO都包含一些方法，这些方法提供对应用数据库的抽象访问权限。

通过使用DAO类（而不是查询构建器或直接查询）访问数据库，您可以拆分数据库架构的不同组件。

DAO既可以是接口，也可以是抽象类。如果是抽象类，则该DAO可以选择有一个以`RoomDatabase`为唯一参数的构造函数，Room会在编译时创建每个DAO实现。

***注意：除非您对构建器调用 allowMainThreadQueries()，否则 Room 不支持在主线程上访问数据库，因为它可能会长时间锁定界面。异步查询（返回 LiveData 或 Flowable 实例的查询）无需遵守此规则，因为此类查询会根据需要在后台线程上异步运行查询。***

<!-- more -->

## 定义方法以方便使用

您可以使用 DAO 类表示多个便捷查询。本文档提供了几个常见示例。

### Insert

当您创建 DAO 方法并使用 `@Insert`对其进行注释时，Room 会生成一个实现，该实现在单个事务中将所有参数插入到数据库中。

以下代码段展示了几个示例查询：

```java
    @Dao
    public interface MyDao {
        @Insert(onConflict = OnConflictStrategy.REPLACE)
        public void insertUsers(User... users);

        @Insert
        public void insertBothUsers(User user1, User user2);

        @Insert
        public void insertUsersAndFriends(User user, List<User> friends);
    }
```

如果 `@Insert`方法只接收 1 个参数，则可返回 `long`，这是插入项的新 `rowId`。如果参数是数组或集合，则应返回 `long[]` 或 `List<Long>`。

### Update

`Update`便捷方法会修改数据库中以参数形式给出的一组实体。它使用与每个实体的主键匹配的查询。

以下代码段演示了如何定义此方法：

```java
    @Dao
    public interface MyDao {
        @Update
        public void updateUsers(User... users);
    }
```

虽然通常没有必要，但您可以让此方法返回一个 `int` 值，表示数据库中更新的行数。

### Delete

`Delete`便捷方法会从数据库中删除一组以参数形式给出的实体。它使用主键查找要删除的实体。

以下代码段演示了如何定义此方法：

```java
    @Dao
    public interface MyDao {
        @Delete
        public void deleteUsers(User... users);
    }
```

虽然通常没有必要，但您可以让此方法返回一个 `int` 值，表示从数据库中删除的行数。

## Query

`@Query`是 DAO 类中使用的主要注释。它允许您对数据库执行读/写操作。每个 `@Query`方法都会在编译时进行验证，因此如果查询出现问题，则会发生编译错误，而不是运行时失败。

Room 还会验证查询的返回值，这样的话，当返回的对象中的字段名称与查询响应中的对应列名称不匹配时，Room 会通过以下两种方式之一提醒您：

- 如果只有部分字段名称匹配，则会发出警告
- 如果没有任何字段名称匹配，则会发出错误

### 简单查询

```java
    @Dao
    public interface MyDao {
        @Query("SELECT * FROM user")
        public User[] loadAllUsers();
    }
```

这是一个极其简单的查询，可加载所有用户。在编译时，Room 知道它在查询用户表中的所有列。如果查询包含语法错误，或者数据库中没有用户表格，则 Room 会在您的应用编译时显示包含相应消息的错误。

### 将参数传递给查询

在大多数情况下，您需要将参数传递给查询以执行过滤操作，例如仅显示某个年龄以上的用户。要完成此任务，请在 Room 注释中使用方法参数，如以下代码段所示：

```java
    @Dao
    public interface MyDao {
        @Query("SELECT * FROM user WHERE age > :minAge")
        public User[] loadAllUsersOlderThan(int minAge);
    }
```

在编译时处理此查询时，Room 会将 `:minAge` 绑定参数与 `minAge` 方法参数相匹配。Room 通过参数名称进行匹配。如果有不匹配的情况，则应用编译时会出现错误。

您还可以在查询中传递多个参数或多次引用这些参数，如以下代码段所示：

```java
    @Dao
    public interface MyDao {
        @Query("SELECT * FROM user WHERE age BETWEEN :minAge AND :maxAge")
        public User[] loadAllUsersBetweenAges(int minAge, int maxAge);

        @Query("SELECT * FROM user WHERE first_name LIKE :search " +
               "OR last_name LIKE :search")
        public List<User> findUserWithName(String search);
    }
```

### 返回列的子集

大多数情况下，您只需获取实体的几个字段。例如，您的界面可能仅显示用户的名字和姓氏，而不是用户的每一条详细信息。通过仅提取应用界面中显示的列，您可以节省宝贵的资源，并且您的查询也能更快完成。

借助 Room，您可以从查询中返回任何基于 Java 的对象，前提是结果列集合会映射到返回的对象。例如，您可以创建以下基于 Java 的普通对象 (POJO) 来获取用户的名字和姓氏：

```java
    public class NameTuple {
        @ColumnInfo(name = "first_name")
        public String firstName;

        @ColumnInfo(name = "last_name")
        @NonNull
        public String lastName;
    }
```

现在，您可以在查询方法中使用此 POJO：

```java
    @Dao
    public interface MyDao {
        @Query("SELECT first_name, last_name FROM user")
        public List<NameTuple> loadFullName();
    }
```

Room 知道该查询会返回 `first_name` 和 `last_name` 列的值，并且这些值会映射到 `NameTuple` 类的字段。因此，Room 可以生成正确的代码。如果查询返回太多的列，或者返回 `NameTuple` 类中不存在的列，则 Room 会显示一条警告。

***注意：这些 POJO 也可以使用 @Embedded 注释。***

### 传递参数的集合

您的部分查询可能要求您传入数量不定的参数，参数的确切数量要到运行时才知道。例如，您可能希望从部分区域中检索所有用户的相关信息。Room 知道参数何时表示集合，并根据提供的参数数量在运行时自动将其展开。

```java
    @Dao
    public interface MyDao {
        @Query("SELECT first_name, last_name FROM user WHERE region IN (:regions)")
        public List<NameTuple> loadUsersFromRegions(List<String> regions);
    }
```

### 可观察查询

执行查询时，您通常会希望应用的界面在数据发生变化时自动更新。为此，请在查询方法说明中使用 `LiveData`类型的返回值。当数据库更新时，Room 会生成更新`LiveData`所必需的所有代码。

```java
    @Dao
    public interface MyDao {
        @Query("SELECT first_name, last_name FROM user WHERE region IN (:regions)")
        public LiveData<List<User>> loadUsersFromRegionsSync(List<String> regions);
    }
```

***注意：自版本 1.0 起，Room 会根据在查询中访问的表格列表决定是否更新 LiveData 实例。***

### 使用 RxJava 进行响应式查询

Room 为 RxJava2 类型的返回值提供了以下支持：

- `@Query` 方法：Room 支持 `Publisher`、`Flowable`和`Observable`类型的返回值。
- `@Insert`、`@Update`和 `@Delete` 方法：Room 2.1.0 及更高版本支持 `Completable`、`Single` 和 `Maybe` 类型的返回值。

要使用此功能，请在应用的 `build.gradle` 文件中添加最新版本的 **rxjava2** 工件：

app/build.gradle

```java
    dependencies {
        def room_version = "2.1.0"
        implementation 'androidx.room:room-rxjava2:$room_version'
    }
```

以下代码段展示了几个如何使用这些返回类型的示例：

```java
    @Dao
    public interface MyDao {
        @Query("SELECT * from user where id = :id LIMIT 1")
        public Flowable<User> loadUserById(int id);

        // Emits the number of users added to the database.
        @Insert
        public Maybe<Integer> insertLargeNumberOfUsers(List<User> users);

        // Makes sure that the operation finishes successfully.
        @Insert
        public Completable insertLargeNumberOfUsers(User... users);

        /* Emits the number of users removed from the database. Always emits at
           least one user. */
        @Delete
        public Single<Integer> deleteUsers(List<User> users);
    }

```

### 直接光标访问

如果应用的逻辑需要直接访问返回行，您可以从查询返回 `Cursor` 对象，如以下代码段所示：

```java
    @Dao
    public interface MyDao {
        @Query("SELECT * FROM user WHERE age > :minAge LIMIT 5")
        public Cursor loadRawUsersOlderThan(int minAge);
    }
```

***注意：强烈建议您不要使用 Cursor API，因为它无法保证行是否存在或者行包含哪些值。只有当您已具有需要光标且无法轻松重构的代码时，才使用此功能。***

### 查询多个表格

您的部分查询可能需要访问多个表格才能计算出结果。借助 Room，您可以编写任何查询，因此您也可以联接表格。此外，如果响应是可观察数据类型（如 `Flowable` 或 `LiveData`），Room 会观察查询中引用的所有表格，以确定是否存在无效表格。

以下代码段展示了如何执行表格联接来整合两个表格的信息：一个表格包含当前借阅图书的用户，另一个表格包含当前处于已被借阅状态的图书的数据。

```java
    @Dao
    public interface MyDao {
        @Query("SELECT * FROM book " +
               "INNER JOIN loan ON loan.book_id = book.id " +
               "INNER JOIN user ON user.id = loan.user_id " +
               "WHERE user.name LIKE :userName")
       public List<Book> findBooksBorrowedByNameSync(String userName);
    }
```

您还可以从这些查询中返回 POJO。例如，您可以编写一条加载某位用户及其宠物名字的查询，如下所示：

```java
    @Dao
    public interface MyDao {
       @Query("SELECT user.name AS userName, pet.name AS petName " +
              "FROM user, pet " +
              "WHERE user.id = pet.user_id")
       public LiveData<List<UserPet>> loadUserAndPetNames();

       // You can also define this class in a separate file, as long as you add the
       // "public" access modifier.
       static class UserPet {
           public String userName;
           public String petName;
       }
    }

```

 本指南也适用于带有 `@Transaction` 注释的 DAO 方法。您可以使用此功能通过其他 DAO 方法构建暂停数据库方法。然后，这些方法会在单个数据库事务中运行。

```java
	@Dao
    abstract class UsersDao {
        @Transaction
        open suspend fun setLoggedInUser(loggedInUser: User) {
            deleteUser(loggedInUser)
            insertUser(loggedInUser)
        }

        @Query("DELETE FROM users")
        abstract fun deleteUser(user: User)

        @Insert
        abstract suspend fun insertUser(user: User)
    }

```

***注意：应避免在单个数据库事务中执行额外的应用端工作，因为 Room 会将此类事务视为独占事务，并且按顺序每次仅执行一个事务。也就是说，包含不必要操作的事务很容易锁定您的数据库并影响性能。***



