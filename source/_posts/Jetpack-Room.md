---
title: Room概览
date: 2020-01-07 12:28:14
tags:
- Android
- Jetpack
- Room
---

Room 在 SQLite 上提供了一个抽象层，以便在充分利用 SQLite 的强大功能的同时，能够流畅地访问数据库。

处理大量结构化数据的应用可极大地受益于在本地保留这些数据，最常见的方法就是缓存相关数据。这样，当设备无法访问网络时，用户仍可在离线状态下浏览相应内容。之后设备重新连接到网络后，用户发起的所有内容更改都会同步到服务器。

<!-- more -->

**注意：要在应用中使用 Room，请在应用的 `build.gradle` 文件中声明 Room 依赖项。**

## 声明依赖项

要添加 Room 的依赖项，您必须将 Google Maven 代码库添加到项目中。

在应用或模块的 `build.gradle` 文件中添加所需工件的依赖项：

```groovy
dependencies {
      def room_version = "2.2.0-rc01"

      implementation "androidx.room:room-runtime:$room_version"
      annotationProcessor "androidx.room:room-compiler:$room_version" // For Kotlin use kapt instead of annotationProcessor

      // optional - Kotlin Extensions and Coroutines support for Room
      implementation "androidx.room:room-ktx:$room_version"

      // optional - RxJava support for Room
      implementation "androidx.room:room-rxjava2:$room_version"

      // optional - Guava support for Room, including Optional and ListenableFuture
      implementation "androidx.room:room-guava:$room_version"

      // Test helpers
      testImplementation "androidx.room:room-testing:$room_version"
    }
    
```

## 配置编译器选项

Room 具有以下注解处理器选项：

- `room.schemaLocation`：配置并启用将数据库架构导出到给定目录中的 JSON 文件的功能。

- `room.incremental`：启用 Gradle 增量注解处理器。

- `room.expandProjection`：配置 Room 以重写查询，使其顶部星形投影在展开后仅包含 DAO 方法返回类型中定义的列。

  以下代码段举例说明了如何配置这些选项：

  ```groovy
  android {
          ...
          defaultConfig {
              ...
              javaCompileOptions {
                  annotationProcessorOptions {
                      arguments = [
                          "room.schemaLocation":"$projectDir/schemas".toString(),
                          "room.incremental":"true",
                          "room.expandProjection":"true"]
                  }
              }
          }
      }
      
  ```


## 组件

Room 包含 3 个主要组件：

1. **Database**：包含数据库持有者，并作为应用已保留的持久关系型数据的底层连接的主要接入点。

   使用`@Database`注释的类应满足以下条件：

   - 是扩展 `RoomDatabase`的抽象类
   - 在注释中添加与数据库关联的实体列表
   - 包含具有 0 个参数且返回使用 `@Dao`注释的类的抽象方法

   在运行时，您可以通过调用 `Room.databaseBuilder()`或 `Room.inMemoryDatabaseBuilder()`获取 `Database`的实例。

2. **Entity**：表示数据库中的表。
3. **DAO**：包含用于访问数据库的方法。

应用使用 Room 数据库来获取与该数据库关联的数据访问对象 (DAO)。然后，应用使用每个 DAO 从数据库中获取实体，然后再将对这些实体的所有更改保存回数据库中。最后，应用使用实体来获取和设置与数据库中的表列相对应的值。

Room 不同组件之间的关系如图 1 所示：

![**图 1.** Room 架构图](room_architecture.png)

以下代码段包含具有一个实体和一个 DAO 的示例数据库配置。

User

```java
    @Entity
    public class User {
        @PrimaryKey
        public int uid;

        @ColumnInfo(name = "first_name")
        public String firstName;

        @ColumnInfo(name = "last_name")
        public String lastName;
    }
```

UserDao

```java
	@Dao
    public interface UserDao {
        @Query("SELECT * FROM user")
        List<User> getAll();

        @Query("SELECT * FROM user WHERE uid IN (:userIds)")
        List<User> loadAllByIds(int[] userIds);

        @Query("SELECT * FROM user WHERE first_name LIKE :first AND " +
               "last_name LIKE :last LIMIT 1")
        User findByName(String first, String last);

        @Insert
        void insertAll(User... users);

        @Delete
        void delete(User user);
    }
```

AppDatabase

```java
    @Database(entities = {User.class}, version = 1)
    public abstract class AppDatabase extends RoomDatabase {
        public abstract UserDao userDao();
    }
```

创建上述文件后，您可以使用以下代码获取已创建的数据库的实例：

```java
    AppDatabase db = Room.databaseBuilder(getApplicationContext(),
            AppDatabase.class, "database-name").build();
```

注意：如果您的应用在单个进程中运行，则在实例化 `AppDatabase` 对象时应遵循单例设计模式。每个 `RoomDatabase`实例的成本相当高，而您几乎不需要在单个进程中访问多个实例。

**单例实现**

```java
@Database(entities = {User.class}, version = 1, exportSchema = false)
public abstract class UserRoomDatabase extends RoomDatabase {

   public abstract UserDao userDao();

   private static volatile UserRoomDatabase INSTANCE;
   private static final int NUMBER_OF_THREADS = 4;
   static final ExecutorService databaseWriteExecutor =
        Executors.newFixedThreadPool(NUMBER_OF_THREADS);

   static UserRoomDatabase getDatabase(final Context context) {
        if (INSTANCE == null) {
            synchronized (UserRoomDatabase.class) {
                if (INSTANCE == null) {
                    INSTANCE = Room.databaseBuilder(context.getApplicationContext(),
                            UserRoomDatabase.class, "user_database")
                        	.allowMainThreadQueries()//允许在UI线程执行数据库操作
                            .build();
                }
            }
        }
        return INSTANCE;
    }
}
```

以上实现了单例模式，可以在应用的Application通过UserRoomDatabase.getDatabase方法获取UserRoomDatabase的实例对象了。

当用户得到UserRoomDatabase对象之后，就可以通过上面的userDao()方法获取UserDao对象，然后就可以执行UserDao中的方法对数据库进行更删改查等操作了。

如果您的应用在多个进程中运行，请在数据库构建器调用中包含 `enableMultiInstanceInvalidation()`。这样，如果您在每个进程中都有一个 `AppDatabase` 实例，就可以在一个进程中使共享数据库文件失效，并且这种失效会自动传播到其他进程中的 `AppDatabase` 实例。