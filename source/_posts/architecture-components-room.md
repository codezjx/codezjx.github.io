---
title: Architecture Components之Room初探
date: 2018-07-14 01:22:00
categories: Architecture Components
tags: [Android, Architecture Components, Room]
---

## 简介
`Room`在SQLite之上提供了一层抽象，能让我们在使用SQLite全部功能的同时还能流畅的对数据库进行访问。`Room`基于注解对SQLite进行了大量封装，用法十分简洁且功能强大并能与`RxJava/LiveData`等无缝结合使用，因此官方强烈推荐使用`Room`来替代SQLite。

## 导入依赖
导入依赖只需要前2行就够了，可以选择是否需要搭配`RxJava`使用：
```gradle
implementation "android.arch.persistence.room:runtime:1.1.1"
annotationProcessor "android.arch.persistence.room:compiler:1.1.1"
// optional - RxJava support for Room
implementation "android.arch.persistence.room:rxjava2:1.1.1"
```

## Room的架构
在`Room`里面主要有3个比较重要的组件：

- Database: 数据库的持有者，它是底层数据库连接的主要接入点。
- Entity: 表示数据库中的表。
- DAO: 包含用于访问数据库的方法。

这些组件与程序中其他部分的调用关系如下所示：
![room_architecture](room_architecture.png)

## 基本使用
下面将介绍`Room`中3个重要组件的基本使用，`Room`中大量使用注解来帮我们生成实现代码，能让我们更加专注于业务而不是写各种模板代码。有用过`Retrofit`的童鞋应该对这深有体会~

### Entity
在`Room`中，对于每个Entity对象，最终都对应数据库中的一个表。

（1）下面的代码展示了如何定义一个基本的Entity：
```java
@Entity(tableName = "users")
public class UserInfo {
    @NonNull
    @PrimaryKey
    @ColumnInfo(name = "userId")
    private String mUserId;

    @ColumnInfo(name = "nickName")
    private String mNickName;

    @ColumnInfo(name = "phone")
    private String mPhone;

    @Ignore
    private Bitmap picture;

    // getter and setter
    ...
}
```

- 使用`@Entity`标记这个类为`Room`中的Entity，可以通过`tableName`属性来指定表名，如果不指定则默认使用类名
- 默认情况下，`Room`会为每个实体对象的成员生成数据列，可以用注解`@Ignore`进行排除
- 可以通过`@PrimaryKey`指定主键，使用`autoGenerate`属性可指定是否需要自动生成主键值
- 可以通过`@ColumnInfo`中的`name`属性指定列名，如果不指定则默认使用变量名

（2）通过`@ForeignKey`注解可以实现在Entity之间定义外键约束，其中`entity`属性指定了需要关联的父Entity，`parentColumns`指定了需要关联到父Entity的字段名，`childColumns`则是当前Entity对应外键的字段名。我们还可以通过`onDelete`和`onUpdate`属性来指定当前Entity要如何响应父Entity的变化。
```java
@Entity(foreignKeys = @ForeignKey(entity = UserInfo.class,
                                    parentColumns = "userId",
                                    childColumns = "uId",
                                    onDelete = CASCADE))
public class Book {
    @NonNull
    @PrimaryKey
    @ColumnInfo(name = "bookId")
    private int mBookId;

    @ColumnInfo(name = "title")
    private String mTitle;

    @ColumnInfo(name = "uId")
    private int userId;
}
```

（3）通过`@Embedded`注解，我们可以在Entity中加入嵌套对象，这样子我们就可以在Entity中直接引用整个POJO了。如下面的例子，在users表中，会新增`street, state, city`这3个数据列。当我们插入或查询数据的时候，`Room`将会帮我们处理好字段与数据表的映射关系。
```java
public class Address {
    @ColumnInfo(name = "street")
    private String mStreet;

    @ColumnInfo(name = "state")
    private String mState;
    
    @ColumnInfo(name = "city")
    private String mCity;
    ...
}

@Entity(tableName = "users")
public class UserInfo {
    ...
    @Embedded
    public Address address;
}
```

### DAO（Data Access Object）
在`Room`中，每个DAO对象都包含提供对数据库访问的抽象方法，且DAO对象只能是接口或者抽象类并用`@Dao`注解进行修饰。下面我们来看一段典型的实现代码，定义好增删查改的接口，就像`Retrofit`中通过注解声明接口一样：

```java
@Dao
public interface UserDao {
    @Query("SELECT * FROM users")
    List<UserInfo> getAll();

    @Query("SELECT * FROM users WHERE userId = :userId")
    UserInfo getUser(String userId);

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    void insert(UserInfo user);

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    void insert(List<UserInfo> users);

    @Update
    public void updateUsers(User... users);

    @Delete
    void delete(UserInfo user);

    @Query("DELETE FROM users")
    void deleteAll();
}
```

（1）通过`@Insert`注解可以把方法中所有的参数当做一次事务批量插入到数据库中，通过`onConflict`属性我们可以指定冲突出现时`Room`该如何操作数据。如果`insert()`方法只有一个参数，返回值可以修改为`long`，此时将会返回插入对象的`rowId`。若是插入多个对象，则返回值应该是`long[]`或者`List<Long>`。

（2）`@Update`和`@Delete`比较类似，可以更新/删除多个对象。返回值可以是`int`，代表了更新/删除Item的总数。

（3）`@Query`是最重要的DAO注解，它允许你对数据库执行读/写操作。并且每个query方法都在编译时进行验证，如果出现SQL语法问题则会直接编译错误。在高版本的`Android Studio`中，还提供了表名/字段名的智能补全功能，十分强大。

（4）加入可观察的查询

1. 使用`LiveData`可使我们的数据在更新后能及时的通知到UI，只需简单包装下方法的返回值即可。
```java
@Dao
public interface UserDao {
    @Query("SELECT first_name, last_name FROM users WHERE region IN (:regions)")
    public LiveData<List<User>> loadUsersFromRegionsSync(List<String> regions);
}
```
2. 使用`RxJava`进行响应式查询，并且可以结合Rx强大的操作符做完成各种骚操作~
```java
@Dao
public interface UserDao {
    @Query("SELECT * from users where id = :id LIMIT 1")
    public Flowable<User> loadUserById(int id);
}
```
3. 返回原始的Cursor对象（官方不推荐使用，因为他不保证行是否存在或者行包含了什么数据）
```java
@Dao
public interface UserDao {
    @Query("SELECT * FROM users WHERE age > :minAge LIMIT 5")
    public Cursor loadRawUsersOlderThan(int minAge);
}
```

### Database
Entity与DAO都准备好了，剩下工作就是初始化`Database`了。在`Room`中声明我们自定义的`Database`有几个比较关键的地方：

- 继承`RoomDatabase`并且需要是抽象类。
- 通过`@Database`注解声明数据库类，并且通过`entities`属性指定包含哪些数据表，`version`属性可以指定数据库版本号，用于后续数据库升级。
- 返回`@DAO`标记的DAO对象的方法必须是0参数的抽象方法。

一切就绪，调用`Room.databaseBuilder()`方法即可生成我们的`Database`实例。一般情况下建议采用单例模式初始化`Database`对象，因为初始化的代价十分高，并且很少会用到多个实例的情况，参考代码如下：

```java
@Database(entities = {UserInfo.class}, version = 1)
public abstract class MainDatabase extends RoomDatabase {

    private static final String DB_NAME = "users.db";
    private static volatile MainDatabase sInstance;

    public abstract UserDao userDao();

    public static void init(Context context) {
        if (sInstance == null) {
            synchronized (MainDatabase.class) {
                if (sInstance == null) {
                    sInstance = Room.databaseBuilder(context.getApplicationContext(),
                            MainDatabase.class, DB_NAME)
                            .build();
                }
            }
        }
    }

    public static MainDatabase getInstance() {
        return sInstance;
    }
}
```

> 注意：在DAO中所有的数据库操作都不能在主线程中进行，否则`Room`会直接抛异常。当然，你也可以通过`RoomDatabase.Builder.allowMainThreadQueries()`方法来解除这个限制，但建议还是遵循官方的建议，采用异步的方式进行或者通过`LiveData`及`RxJava`等异步框架实现。
