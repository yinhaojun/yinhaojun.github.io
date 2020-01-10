---
title: Room定义对象之间的关系
date: 2020-01-07 16:24:21
tags:
- Android
- Jetpack
- Room
---

由于 SQLite 是关系型数据库，因此您可以指定各个对象之间的关系。尽管大多数对象关系映射库都允许实体对象互相引用，但 Room 明确禁止这样做。

## 定义一对多关系

即使您不能使用直接关系，Room 仍允许您定义实体之间的外键约束。

例如，如果存在另一个名为 `Book` 的实体，您可以使用 `@ForeignKey`注释定义该实体与 `User` 实体的关系，如以下代码段所示：

```java
@Entity(foreignKeys = @ForeignKey(entity = User.class,
                                      parentColumns = "id",
                                      childColumns = "user_id"))
    public class Book {
        @PrimaryKey public int bookId;

        public String title;

        @ColumnInfo(name = "user_id") public int userId;
    }
```

由于零个或更多个 `Book` 实例可以通过 `user_id` 外键关联到一个 `User` 实例，因此这会在 `User` 和 `Book` 之间构建一对多关系模型。

外键非常强大，可让您指定引用的实体更新后会发生什么。例如，您可以通过在`@ForeignKey`注释中添加`onDelete = CASCADE`，在 `User` 的对应实例删除后告知 SQLite 删除该用户的所有图书。

***注意：SQLite 将 `@Insert(onConflict = REPLACE)`作为一组 `REMOVE` 和 `REPLACE` 操作（而不是单个 `UPDATE` 操作）处理。这种替换冲突值的方法可能会影响您的外键约束。***

<!-- more -->

## 创建嵌套对象

有时，您可能希望在数据库逻辑中将某个实体或数据对象表示为一个紧密的整体，即使该对象包含多个字段也是如此。在这些情况下，您可以使用 `@Embedded`注释表示要解构到表中其子字段的对象。然后，您可以像查询其他各个列一样查询嵌套字段。

例如，您的 `User` 类可以包含类型 `Address` 的字段，该类型表示一组分别名为 `street`、`city`、`state` 和 `postCode` 的字段。要在表中单独存储组成的列，请在 `User` 类（使用`@Embedded`注释）中添加 `Address` 字段，如以下代码段所示：

```java
    public class Address {
        public String street;
        public String state;
        public String city;

        @ColumnInfo(name = "post_code") public int postCode;
    }

    @Entity
    public class User {
        @PrimaryKey public int id;

        public String firstName;

        @Embedded public Address address;
    }
```

然后，表示 `User` 对象的表会包含具有以下名称的列：`id`、`firstName`、`street`、`state`、`city` 和 `post_code`。

**注意：嵌套字段还可以包含其他嵌套字段。**

如果某个实体具有同一类型的多个嵌套字段，您可以通过设置`prefix`属性确保每个列都独一无二。然后，Room 会将提供的值添加到嵌套对象中每个列名称的开头。

```java
For the example above, if we've written:

   @Embedded(prefix = "loc_")
   Coordinates coordinates;
 
The column names for latitude and longitude will be loc_latitude and loc_longitude respectively.

By default, prefix is the empty string.
```

## 定义多对多关系

您通常希望在关系型数据库中构建的另一种关系模型是两个实体之间的多对多关系，其中每个实体都可以关联到另一个实体的零个或更多个实例。例如，假设有一个音乐在线播放应用，用户可以在该应用中将自己喜爱的歌曲整理到播放列表中。每个播放列表都可以包含任意数量的歌曲，每首歌曲都可以包含在任意数量的播放列表中。

要构建这种关系的模型，您需要创建下面三个对象：

- 播放列表的实体类
- 歌曲的实体类
- 用于保存每个播放列表中的歌曲相关信息的中间类

您可以将实体类定义为独立单元：

```java
    @Entity
    public class Playlist {
        @PrimaryKey public int id;

        public String name;
        public String description;
    }

    @Entity
    public class Song {
        @PrimaryKey public int id;

        public String songName;
        public String artistName;
    }
```

然后，将中间类定义为包含对 `Song` 和 `Playlist` 的外键引用的实体：

```java
    @Entity(tableName = "playlist_song_join",
            primaryKeys = { "playlistId", "songId" },
            foreignKeys = {
                    @ForeignKey(entity = Playlist.class,
                                parentColumns = "id",
                                childColumns = "playlistId"),
                    @ForeignKey(entity = Song.class,
                                parentColumns = "id",
                                childColumns = "songId")
                    })
    public class PlaylistSongJoin {
        public int playlistId;
        public int songId;
    }
```

这会生成一个多对多关系模型。借助该模型，您可以使用 DAO 按歌曲查询播放列表和按播放列表查询歌曲：

```java
    @Dao
    public interface PlaylistSongJoinDao {
        @Insert
        void insert(PlaylistSongJoin playlistSongJoin);

        @Query("SELECT * FROM playlist " +
               "INNER JOIN playlist_song_join " +
               "ON playlist.id=playlist_song_join.playlistId " +
               "WHERE playlist_song_join.songId=:songId")
        List<Playlist> getPlaylistsForSong(final int songId);

        @Query("SELECT * FROM song " +
               "INNER JOIN playlist_song_join " +
               "ON song.id=playlist_song_join.songId " +
               "WHERE playlist_song_join.playlistId=:playlistId")
        List<Song> getSongsForPlaylist(final int playlistId);
    }
    
```









