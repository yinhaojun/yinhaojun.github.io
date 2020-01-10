---
title: Room预填充数据库
date: 2020-01-07 20:53:39
tags:
- Android
- Jetpack
- Room
---

有时，您可能希望您的应用程序从已经加载了一组特定数据的数据库开始，这称为预填充数据库。在Room2.2.0及更高版本中，可以使用API方法在初始化时使用设备文件系统中预打包的数据库文件中的内容预填充Room数据库。

***注意：通过`Room.inMemoryDatabaseBuilder()`得到的数据库不支持使用`createFromAsset()`或`createFromFile()`预填充数据库。***

<!-- more -->

## 从asset预填充

要从位于应用程序`asset/`目录中任何位置的预打包数据库文件中预填充Room数据库，请在调用build()之前从RoomDatabase.Builder对象中调用`createFromAsset()`方法：

```java
Room.databaseBuilder(appContext, AppDatabase.class, "Sample.db")
    .createFromAsset("database/myapp.db")
    .build();
```

`createFromAsset()`方法接受一个字符串参数，该参数包含从assets /目录到预打包数据库文件的相对路径。

Note: When prepopulating from an asset, Room validates the database to ensure that its schema matches the schema of the prepackaged database. You should export your database's schema to use as a reference as you create the prepackaged database file.

***注意：从`asset`预填充时，Room会验证数据库，以确保其结构与预打包数据库的结构(schema)匹配。 创建预打包的数据库文件时，应导出数据库的结构以用作参考。Schema，指的是一组DDL语句集，该语句集完整地描述了数据库的结构。***

## 从文件系统中预填充

要从位于设备文件系统中除应用程序`asset/`目录之外的任何位置的预打包数据库文件中预填充Room数据库，请在调用build（）之前从RoomDatabase.Builder对象中调用`createFromFile()`方法：

```java
Room.databaseBuilder(appContext, AppDatabase.class, "Sample.db")
    .createFromFile(new File("mypath"))
    .build();
```

`createFromFile()`方法接受预打包数据库文件的File参数。 Room会创建指定文件的副本，而不是直接打开它，因此请确保您的应用对文件具有读取权限。

## 处理包括预打包数据库的迁移

预打包的数据库文件还可以更改Room数据库处理回退迁移的方式。 通常，启用破坏性迁移后，Room必须执行没有迁移路径的迁移，Room会删除数据库中的所有表，并为目标版本创建具有指定架构的空数据库。 但是，如果包括与目标版本号相同的预打包数据库文件，Room会在执行破坏性迁移后尝试使用预打包数据库文件的内容填充新创建的数据库。

下面的内容提供了一些如何在实践中工作的示例。

#### 示例：使用预打包的数据库进行回退迁移

假设以下内容：

- 您在应用上定义了Room数据库，版本为3
- ​     设备上已安装的数据库实例为版本2
- ​     预打包的数据库文件的版本为3
- ​     从版本2到版本3没有实现的迁移路径
- ​     破坏性迁移(**Destructive migrations**)已启用。

```java
// 定义数据库的版本是 3.
@Database(version = 3)
public abstract class AppDatabase extends RoomDatabase {
    ...
}

// 启用Destructive migrations并提供了预打包的数据myapp.db
Room.databaseBuilder(appContext, AppDatabase.class, "Sample.db")
    .createFromAsset("database/myapp.db")
    .fallbackToDestructiveMigration()
    .build();

```

在这种情况下会发生以下情况：

- 由于您的应用程序中定义的数据库为版本3，而设备上已安装的数据库实例为版本2，因此必须进行迁移
- 由于没有从版本2到版本3的已实施迁移计划，因此迁移是回退迁移。
- 因为调用fallbackToDestructiveMigration()构建器方法，所以回退迁移是破坏性的。 Room会删除设备上已安装的数据库实例。
- 由于预打包的数据库文件的版本为3，因此Room会重新创建数据库并使用预打包的数据库文件的内容填充它。 另一方面，如果您预打包的数据库文件位于版本2上，则Room将注意该文件与目标版本不匹配，因此不会将其用作回退迁移的一部分（也就是不会将预打包数据库的内容填充到新的数据库中）。

#### 示例：使用预打包的数据库实施迁移

假设您的应用实现了从版本2到版本3的迁移路径：

```java
// Database class definition declaring version 3.
@Database(version = 3)
public abstract class AppDatabase extends RoomDatabase {
    ...
}

// Migration path definition from version 2 to version 3.
static final Migration MIGRATION_2_3 = new Migration(2, 3) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        ...
    }
};

// A prepackaged database is provided.
Room.databaseBuilder(appContext, AppDatabase.class, "Sample.db")
    .createFromAsset("database/myapp.db")
    .addMigrations(MIGRATION_2_3)
    .build();
```

在这种情况下会发生以下情况：

- 由于您的应用程序中定义的数据库为版本3，而设备上已安装的数据库为版本2，因此必须进行迁移
- 由于存在从版本2到版本3的已实现迁移路径，因此Room运行定义的migration（）方法将设备上的数据库实例更新到版本3，从而保留数据库中已经存在的数据。 Room不使用预打包的数据库文件，因为Room仅在回退迁移的情况下才使用预打包的数据库文件

#### 示例：使用预打包的数据库进行多步迁移

预打包数据库文件还可能影响包含多个步骤的迁移。 考虑以下情况：

- 您在应用上定义了Room数据库版本为4
- 设备上已安装的数据库实例为版本2
- 预打包数据库文件的版本为3
- 从版本3到版本4有一个已实现的迁移路径，但没有从版本2到版本3的迁移路径
- 破坏性迁移已启用

```java
// Database class definition declaring version 4.
@Database(version = 4)
public abstract class AppDatabase extends RoomDatabase {
    ...
}

// Migration path definition from version 3 to version 4.
static final Migration MIGRATION_3_4 = new Migration(3, 4) {
    @Override
    public void migrate(SupportSQLiteDatabase database) {
        ...
    }
};

// Destructive migrations are enabled and a prepackaged database is
// provided.
Room.databaseBuilder(appContext, AppDatabase.class, "Sample.db")
    .createFromAsset("database/myapp.db")
    .addMigrations(MIGRATION_3_4)
    .fallbackToDestructiveMigration()
    .build();
```

在这种情况下会发生以下情况：

- 由于您的应用程序中定义的数据库为版本4，而设备上已安装的数据库实例为版本2，因此必须进行迁移
- 由于没有从版本2到版本3的已实现迁移路径，因此迁移是回退迁移
- 因为调用fallbackToDestructiveMigration（）构建器方法，所以回退迁移是破坏性的。 Room将设备上的数据库实例删除
- 由于预打包数据库文件的版本为3，因此Room会重新创建数据库并使用预打包的数据库文件的内容填充它
- 设备上安装的数据库现在的版本为3。由于它仍低于应用程序中定义的版本，因此必须进行另一次迁移
- 由于存在从版本3到版本4的已实现迁移路径，因此Room运行定义的`migration()`方法将设备上的数据库实例更新到版本4，从而保留从版本3预打包的数据库文件复制过来的数据。

















