# Room

Room是一个对象关系映射(ORM)库。Room抽象了SQLite的使用，可以在充分利用SQLite的同时访问流畅的数据库。

 Room由三个重要的组件组成：`Database`、`Entity`、`DAO`。



## Entity

**实体类都需要添加@Entity注解。而且Entity类中需要映射到表中的字段需要保证外部能访问到这些字段(你要么把字段什么为public、要不实现字段的getter和setter方法)**

###### 表名

默认情况下Entity类的名字就是表的名字(不区分大小写)。但是我们也可以通过@Entity的tableName属性来自定义表名字

```
@Entity(tableName = "users")
public class User {
    ...
}
```

###### 设置列的名字

我们也是可以通过@ColumnInfo注解来自定义表中列的名字。如下代码users表中first_name列对应firstName字段，last_name列对应lastName字段

```
@Entity(tableName = "user")
public class User {
    @PrimaryKey
    public int id;

    @ColumnInfo(name = "first_name")
    public String firstName;

    @ColumnInfo(name = "last_name")
    public String lastName;
}
```

###### 主键

   每个Entity都需要至少一个字段设置为主键。即使这个Entity只有一个字段也需要设置为主键。Entity设置主键的方式有两种

通过@Entity的primaryKeys属性来设置主键(primaryKeys是数组可以设置单个主键，也可以设置复合主键)。

```
@Entity(primaryKeys = {"firstName",
                       "lastName"})
public class User {

    public String firstName;
    public String lastName;
}
```

或者@PrimaryKey注解来设置主键，同时可以设置自增autoGenerate属性



###### 索引

数据库索引用于提高数据库表的数据访问速度的。数据库里面的索引有单列索引和组合索引。Room里面可以通过@Entity的indices属性来给表格添加索引。

```
@Entity(indices = {@Index("firstName"),
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

索引也是分两种的唯一索引和非唯一索引。唯一索引就像主键一样重复会报错的。可以通过的@Index的unique数学来设置是否唯一索引

```
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

###### 嵌套对象

会需要将多个对象组合成一个对象。对象和对象之间是有嵌套关系的。Room中你就可以使用`@Embedded`注解来表示嵌入。然后，您可以像查看其他单个列一样查询嵌入字段



## DAO

 这个组件代表了作为DAO的类或者接口。DAO是Room的主要组件，负责定义访问数据库的方法。Room使用过程中一般使用抽象DAO类来定义数据库的CRUD操作。DAO可以是一个接口也可以是一个抽象类。如果它是一个抽象类，它可以有一个构造函数，它将RoomDatabase作为其唯一参数。Room在编译时创建每个DAO实。

DAO里面所有的操作都是依赖方法来实现的。



@Insert注解可以设置一个属性

onConflict：默认值是OnConflictStrategy.ABORT，表示当插入有冲突的时候的处理策略。OnConflictStrategy封装了Room解决冲突的相关策略：

- OnConflictStrategy.REPLACE：冲突策略是取代旧数据同时继续事务。
- OnConflictStrategy.ROLLBACK：冲突策略是回滚事务。
- OnConflictStrategy.ABORT：冲突策略是终止事务。
- OnConflictStrategy.FAIL：冲突策略是事务失败。
- OnConflictStrategy.IGNORE：冲突策略是忽略冲突。

```
@Dao
public interface UserDao {

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    void insertUsers(User... users);
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    void insertAllUsers(List<User> lists);


    @Update(onConflict = OnConflictStrategy.REPLACE)
    int updateUsers(User... users);  //可以返回int变量。表示更新了多少行。
    
    @Delete
    void deleteUsers(User... users);//可以设置int返回值来表示删除了多少行。

}
```



查询语句

```
@Dao
public interface UserDao {

    @Query("SELECT * FROM user")
    User[] loadAllUsers();

    @Query("SELECT * FROM user WHERE firstName == :name")
    User[] loadAllUsersByFirstName(String name);
    
    @Query("SELECT * FROM user WHERE age BETWEEN :minAge AND :maxAge")
    User[] loadAllUsersBetweenAges(int minAge, int maxAge);

    @Query("SELECT * FROM user WHERE firstName LIKE :search " + "OR lastName LIKE :search")
    List<User> findUserWithName(String search);
    
    //查询的时候您可能需要传递一组(数组或者List)参数进去
    @Query("SELECT firstName, lastName FROM user WHERE region IN (:regions)")
    public List<NameTuple> loadUsersFromRegions(List<String> regions);
}
```

## Database

@Database注解可以用来创建数据库的持有者。该注解定义了实体列表，该类的内容定义了数据库中的DAO列表。这也是访问底层连接的主要入口点。注解类应该是抽象的并且扩展自RoomDatabase

- entities：数据库相关的所有Entity实体类，他们会转化成数据库里面的表。
- version：数据库版本。
- exportSchema：默认true，也是建议传true，这样可以把Schema导出到一个文件夹里面。同时建议把这个文件夹上次到VCS。

```
@Database(entities = {User.class, Book.class}, version = 3,exportSchema = false)
public abstract class AppDatabase extends RoomDatabase {

    public abstract UserDao userDao();

    public abstract BookDao bookDao();

}
```

在运行时，你可以通过调用Room.databaseBuilder()或者Room.inMemoryDatabaseBuilder()获取实例。因为每次创建Database实例都会产生比较大的开销，所以应该将Database设计成单例的，或者直接放在Application中创建。

```
        mAppDatabase = Room.databaseBuilder(getApplicationContext(), AppDatabase.class, "android_room_dev.db")
                           .allowMainThreadQueries()
                           .build();
```



## 迁移

Room里面以Migration类的形式提供可一个简化SQLite迁移的抽象层。Migration提供了从一个版本到另一个版本迁移的时候应该执行的操作

- 如果database的版本号不变。app操作数据库表的时候会直接crash掉。(错误的做法)
- 如果增加database的版本号。但是不提供Migration。app操作数据库表的时候会直接crash掉。（错误的做法）
- 如果增加database的版本号。同时启用fallbackToDestructiveMigration。这个时候之前的数据会被清空掉。如下fallbackToDestructiveMigration()设置。(不推荐的做法)

- 增加database的版本号，同时提供Migration。这要是Room数据迁移的重点。(最正确的做法)

```
public class AppApplication extends Application {

    private AppDatabase mAppDatabase;

    @Override
    public void onCreate() {
        super.onCreate();
        mAppDatabase = Room.databaseBuilder(getApplicationContext(), AppDatabase.class, "android_room_dev.db")
                           .allowMainThreadQueries()
                           .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
                           .build();
    }

    public AppDatabase getAppDatabase() {
        return mAppDatabase;
    }

    /**
     * 数据库版本 1->2 user表格新增了age列
     */
    static final Migration MIGRATION_1_2 = new Migration(1, 2) {
        @Override
        public void migrate(SupportSQLiteDatabase database) {
            database.execSQL("ALTER TABLE User ADD COLUMN age integer");
        }
    };

    /**
     * 数据库版本 2->3 新增book表格
     */
    static final Migration MIGRATION_2_3 = new Migration(2, 3) {
        @Override
        public void migrate(SupportSQLiteDatabase database) {
            database.execSQL(
                "CREATE TABLE IF NOT EXISTS `book` (`uid` INTEGER PRIMARY KEY autoincrement, `name` TEXT , `userId` INTEGER, 'time' INTEGER)");
        }
    };
}
```

