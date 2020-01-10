---
title: 使用 Room 引用复杂数据
date: 2020-01-09 15:21:51
tags:
- Android
- Jetpack
- Room
---

Room 提供了在基元类型和盒装类型之间进行转换的功能，但不允许实体之间进行对象引用。本文档介绍了如何使用类型转换器，以及 Room 为何不支持对象引用。

## 使用类型转换器

有时，您的应用需要使用自定义数据类型，其中包含您想要存储到单个数据库列中的值。要为自定义类型添加此类支持，您需要提供一个`TypeConverter`，它可以在自定义类与 Room 可以保留的已知类型之间来回转换。

例如，如果我们想保留 `Date` 的实例，则可以编写以下 `TypeConverter`，将等效的 Unix 时间戳存储在数据库中：

```java
    public class Converters {
        @TypeConverter
        public static Date fromTimestamp(Long value) {
            return value == null ? null : new Date(value);
        }

        @TypeConverter
        public static Long dateToTimestamp(Date date) {
            return date == null ? null : date.getTime();
        }
    }
```

<!-- more-->

前面的示例定义了 2 个函数，一个用于将 `Date` 对象转换为 `Long` 对象，另一个用于执行从 `Long` 到 `Date` 的反向转换。由于 Room 已知道如何保留 `Long` 对象，因此可以使用此转换器来保存 `Date` 类型的值。

接着，将 `@TypeConverters`注释添加到`AppDatabase`类中，以便`Room`可以使用您为该`AppDatabase`中的每个实体和 DAO 定义的转换器：

```java
    @Database(entities = {User.class}, version = 1)
    @TypeConverters({Converters.class})
    public abstract class AppDatabase extends RoomDatabase {
        public abstract UserDao userDao();
    }
```

通过使用这些转换器，您就可以在其他查询中使用自定义类型，就像使用基元类型一样，如以下代码段所示：

User

```java
    @Entity
    public class User {
        private Date birthday;
    }
```

UserDao

```java
    @Dao
    public interface UserDao {
        @Query("SELECT * FROM user WHERE birthday BETWEEN :from AND :to")
        List<User> findUsersBornBetweenDates(Date from, Date to);
    }
```

您还可以将`@TypeConverters`限制为不同的范围，包括个别实体、DAO 和 DAO 方法。如需了解详情，请参阅有关`@TypeConverters` 注释的参考文档。

## 了解 Room 为何不允许对象引用

***要点：Room 不允许实体类之间进行对象引用。因此，您必须明确请求您的应用所需的数据。***

映射从数据库到相应对象模型之间的关系是一种常见做法，极其适用于服务器端。即使程序在访问字段时加载字段，服务器仍然可以正常工作。

但在客户端，这种延迟加载是不可行的，因为它通常发生在界面线程上，并且在界面线程上查询磁盘上的信息会导致严重的性能问题。界面线程通常需要大约 16毫秒来计算和绘制 Activity 的更新后的布局，因此，即使查询只用了 5 毫秒，您的应用仍然可能会用尽剩余的时间来绘制框架，从而导致明显的显示故障。如果有一个并行运行的单独事务，或者设备正在运行其他磁盘密集型任务，则查询可能需要更多时间才能完成。不过，如果您不使用延迟加载，则应用会抓取一些不必要的数据，从而导致内存消耗问题。

对象关系型映射通常将决定权留给开发者，以便他们可以针对自己的应用用例执行最合适的操作。开发者通常会决定在应用和界面之间共享模型。不过，这种解决方案并不能很好地扩展，因为界面会不断发生变化，共享模型会出现开发者难以预测和调试的问题。

例如，假设界面加载了`Book`对象的列表，其中每本图书都有一个`Author`对象。您最初可能设计让查询使用延迟加载，从而让 `Book` 实例检索作者。对 `author` 字段的第一次检索会查询数据库。一段时间后，您发现还需要在应用的界面中显示作者姓名。您可以轻松访问此名称，如以下代码段所示：

```java
    authorNameTextView.setText(book.getAuthor().getName());
```

不过，这种看似无害的更改会导致在主线程上查询 `Author` 表。

如果您事先查询作者信息，则在您不再需要这些数据时，就会很难更改数据加载方式。例如，如果应用的界面不再需要显示 `Author` 信息，则应用会有效地加载不再显示的数据，从而浪费宝贵的内存空间。如果 `Author` 类引用其他表（例如 `Books`），则应用的效率会进一步下降。

要使用 Room 同时引用多个实体，请改为创建包含每个实体的 POJO，然后编写用于联接相应表的查询。这种结构合理的模型结合 Room 强大的查询验证功能，可让您的应用在加载数据时消耗较少的资源，从而改善应用的性能和用户体验。

































