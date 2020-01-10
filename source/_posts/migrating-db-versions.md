---
title: 迁移 Room 数据库
date: 2020-01-09 14:14:11
tags:
- Android
- Jetpack
- Room
---

当您在应用中添加和更改功能时，需要修改实体类以反映这些更改。当用户更新到您应用的最新版本时，您不想让他们丢失所有现有数据，尤其是在您无法从远程服务器恢复数据时。

借助 `Room `，您可以编写 `Migration`类，以这种方式保留用户数据。每个 `Migration`类均指定一个 startVersion` 和 `endVersion`。在运行时，`Room`会运行每个`Migration`类的`migrate()`方法，以按照正确的顺序将数据库迁移到更高版本。

```java
    static final Migration MIGRATION_1_2 = new Migration(1, 2) {
        @Override
        public void migrate(SupportSQLiteDatabase database) {
            database.execSQL("CREATE TABLE `Fruit` (`id` INTEGER, "
                    + "`name` TEXT, PRIMARY KEY(`id`))");
        }
    };

    static final Migration MIGRATION_2_3 = new Migration(2, 3) {
        @Override
        public void migrate(SupportSQLiteDatabase database) {
            database.execSQL("ALTER TABLE Book "
                    + " ADD COLUMN pub_year INTEGER");
        }
    };

    Room.databaseBuilder(getApplicationContext(), MyDb.class, "database-name")
            .addMigrations(MIGRATION_1_2, MIGRATION_2_3).build();
    
```

迁移过程完成后，Room 会验证架构以确保迁移正确进行。如果 Room 发现问题，则会抛出包含不匹配信息的异常。

<!-- more -->

### 测试迁移

编写迁移非常重要，如果未正确编写，可能会导致应用陷入崩溃循环。为了保持应用的稳定性，您应事先测试迁移。Room 提供了一个测试 Maven 工件来协助完成此测试过程。不过，要使此工件正常工作，您需要导出数据库的架构。

### 导出架构

在编译时，Room 会将数据库的架构信息导出为 JSON 文件。要导出架构，请在 `build.gradle` 文件中设置 `room.schemaLocation` 注释处理器属性，如以下代码段所示：

build.gradle

```groovy
    android {
        ...
        defaultConfig {
            ...
            javaCompileOptions {
                annotationProcessorOptions {
                    arguments = ["room.schemaLocation":
                                 "$projectDir/schemas".toString()]
                }
            }
        }
    }
```

您应将导出的 JSON 文件（代表数据库的架构历史记录）存储在版本控制系统中，因为此系统可让 Room 创建您数据库的较低版本以进行测试。

要测试这些迁移，请将 Room 中的 **Room.man.room:room:testing** Maven 工件添加至测试依赖项中，并将该架构位置添加为资源文件夹，如以下代码段所示：

build.gradle

```
    android {
        ...
        sourceSets {
            androidTest.assets.srcDirs += files("$projectDir/schemas".toString())
        }
    }
```

测试软件包提供了可读取这些架构文件的`MigrationTestHelper` 类。它还实现了 JUnit4`TestRule`接口，因此可以管理创建的数据库。

以下代码段展示了示例迁移测试：

```java
    @RunWith(AndroidJUnit4.class)
    public class MigrationTest {
        private static final String TEST_DB = "migration-test";

        @Rule
        public MigrationTestHelper helper;

        public MigrationTest() {
            helper = new MigrationTestHelper(InstrumentationRegistry.getInstrumentation(),
                    MigrationDb.class.getCanonicalName(),
                    new FrameworkSQLiteOpenHelperFactory());
        }

        @Test
        public void migrate1To2() throws IOException {
            SupportSQLiteDatabase db = helper.createDatabase(TEST_DB, 1);

            // db has schema version 1. insert some data using SQL queries.
            // You cannot use DAO classes because they expect the latest schema.
            db.execSQL(...);

            // Prepare for the next version.
            db.close();

            // Re-open the database with version 2 and provide
            // MIGRATION_1_2 as the migration process.
            db = helper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2);

            // MigrationTestHelper automatically verifies the schema changes,
            // but you need to validate that the data was migrated properly.
        }
    }
    
```

### 测试所有迁移

上述示例展示了如何测试从一个版本到另一个版本的增量迁移。不过，建议您进行一次贯穿所有迁移的测试。这种类型的测试有助于捕获经过迁移路径的数据库与最近创建的数据库之间存在的所有差异。

以下代码段展示了所有迁移测试的示例：

```java
    @RunWith(AndroidJUnit4.class)
    public class MigrationTest {
        private static final String TEST_DB = "migration-test";

        @Rule
        public MigrationTestHelper helper;

        public MigrationTest() {
            helper = new MigrationTestHelper(InstrumentationRegistry.getInstrumentation(),
                    AppDatabase.class.getCanonicalName(),
                    new FrameworkSQLiteOpenHelperFactory());
        }

        @Test
        public void migrateAll() throws IOException {
            // Create earliest version of the database.
            SupportSQLiteDatabase db = helper.createDatabase(TEST_DB, 1);
            db.close();

            // Open latest version of the database. Room will validate the schema
            // once all migrations execute.
            AppDatabase appDb = Room.databaseBuilder(
                    InstrumentationRegistry.getInstrumentation().getTargetContext(),
                    AppDatabase.class,
                    TEST_DB)
                    .addMigrations(ALL_MIGRATIONS).build()
            appDb.getOpenHelper().getWritableDatabase();
            appDb.close();
        }

        // Array of all migrations
        private static final Migration[] ALL_MIGRATIONS = new Migration[]{
                MIGRATION_1_2, MIGRATION_2_3, MIGRATION_3_4};
    }
```

## 妥善处理缺失的迁移路径

更新数据库的架构后，部分设备上的数据库可能仍会使用较低版本的架构。如果 Room 无法找到将该设备的数据库从旧版本升级到当前版本的迁移规则，则会发生`IllegalStateException`。

要防止应用在发生这种情况时崩溃，请在创建数据库时调用 `fallbackToDestructiveMigration()` 构建器方法：

```java
    Room.databaseBuilder(getApplicationContext(), MyDb.class, "database-name")
            .fallbackToDestructiveMigration()
            .build();
```

通过在应用的数据库构建逻辑中添加此子句，您指示 Room 在架构版本之间缺少迁移路径时破坏性地重建应用的数据库表。

***警告：通过在应用的数据库构建器中配置此选项，Room 会在缺少迁移路径时从数据库表中永久删除所有数据。***

破坏性重建回退逻辑包含几个附加选项：

- 如果特定版本的架构历史记录中出现您无法通过迁移路径解决的问题，请使用 `fallbackToDestructiveMigrationFrom()`。此方法表示您希望只有当数据库尝试从某个出现问题的版本迁移时，Room 才使用回退逻辑。
- 要仅在尝试进行架构降级时执行破坏性重建，请改用 `fallbackToDestructiveMigrationOnDowngrade()`。

## 升级到 Room 2.2.0 时处理列默认值

Room 2.2.0 中添加了通过 `@ColumnInfo(defaultValue = "...")` 定义列默认值的支持。列的默认值是数据库架构和实体的重要组成部分，在迁移过程中由 Room 进行验证。如果您的数据库是以前由 Room 2.2.0 之前的版本创建的，那么您可能需要迁移以前添加的 Room 不知道的默认值。

例如，在一个数据库的版本 1 中，存在一个声明如下的 `Song` 实体：

Song Entity, DB Version 1, Room 2.1.0

```java
    @Entity
    public class Song {
        @PrimaryKey
        final long id;
        final String title;
    }
```

对于同一数据库的版本 2，添加了新的 `NOT NULL` 列：

Song Entity, DB Version 2, Room 2.1.0

```java
    @Entity
    public class Song {
        @PrimaryKey
        final long id;
        final String title;
        @NonNull
        final String tag; // added in version 2
    }
```

从版本 1 迁移到 2：

从 1 迁移到 2，Room 2.1.0

```java
    static final Migration MIGRATION_1_2 = new Migration(1, 2) {
        @Override
        public void migrate(SupportSQLiteDatabase database) {
            database.execSQL(
                "ALTER TABLE Song ADD COLUMN tag TEXT NOT NULL DEFAULT ''");
        }
    };
```

此类迁移在 Room 2.2.0 之前的版本中没有什么不良后果，但在升级 Room 并通过 `@ColumnInfo` 将默认值添加到同一列后会导致出现问题。通过使用 `ALTER TABLE`，实体 `Song` 会发生更改，不仅包含新的列 `tag`，还包含默认值。不过，Room 2.2.0 之前的版本并不知道此类更改，从而导致全新安装应用的用户与从版本 1 迁移到版本 2 的用户之间的架构不一致。具体而言，在版本 2 中新建的数据库不包含默认值。

对于此类情况，必须进行迁移，使数据库架构在使用应用的各类用户之间保持一致，因为 Room 2.2.0 现在将在默认值在实体类中定义后对其进行验证。所需的迁移类型包括：

- 使用 `@ColumnInfo` 在实体类中声明默认值。
- 逐一增加数据库版本。
- 进行迁移以实现删除和重建策略（允许向已创建的列添加默认值）。

以下代码段展示了一个删除并重建 `Song` 表的示例迁移实现：

从 2 迁移到 3，Room 2.2.0

```java
    static final Migration MIGRATION_2_3 = new Migration(2, 3) {
        @Override
        public void migrate(SupportSQLiteDatabase database) {
            database.execSQL("CREATE TABLE new_Song (" +
                    "id INTEGER PRIMARY KEY NOT NULL," +
                    "name TEXT," +
                    "tag TEXT NOT NULL DEFAULT '')");
            database.execSQL("INSERT INTO new_Song (id, name, tag) " +
                    "SELECT id, name, tag FROM Song");
            database.execSQL("DROP TABLE Song");
            database.execSQL("ALTER TABLE new_Song RENAME TO Song");
        }
    };
```

***注意：如果您的数据库回退到破坏性迁移，或者没有进行此类可能添加了列及默认值的迁移，则不需要进行此迁移。***







