[toc]

# 1. 环境配置

## 1.1 配置

```
go get -u gorm.io/gorm

# mysql驱动
go get -u gorm.io/driver/mysql
```

## 1.2 连接数据库

```
func ConnectDB() *gorm.DB {
	dsn := "root:Admin@123@tcp(192.168.88.132:3306)/bookmanager?charset=utf8mb4&parseTime=True&loc=Local"
	mysqlConfig := mysql.Config{
		DSN:                       dsn,
		DefaultStringSize:         256,   // string 类型字段的默认长度
		DisableDatetimePrecision:  true,  // 禁用 datetime 精度，MySQL 5.6 之前的数据库不支持
		DontSupportRenameIndex:    true,  // 重命名索引时采用删除并新建的方式，MySQL 5.7 之前的数据库和 MariaDB 不支持重命名索引
		DontSupportRenameColumn:   true,  // 用 `change` 重命名列，MySQL 8 之前的数据库和 MariaDB 不支持重命名列
		SkipInitializeWithVersion: false, // 根据当前 MySQL 版本自动配置
	}
	db, err := gorm.Open(mysql.New(mysqlConfig), &gorm.Config{
		Logger: logger.New(
			log.New(os.Stdout, "\n", log.LstdFlags),
			logger.Config{
				SlowThreshold:             time.Second, // 慢SQL阈值: time.Second(1秒)
				LogLevel:                  logger.Info, // 日志级别: Silent, Error, Warn, Info
				IgnoreRecordNotFoundError: true,        // 日志里忽略 ErrRecordNotFound 异常
				Colorful:                  true,        // shell终端里按颜色输出
			}),
	})
	if err != nil {
		panic(err)
	} else {
		// 配置连接池
		sqlDB, _ := db.DB()
		// SetMaxIdleConns 用于设置连接池中空闲连接的最大数量。
		sqlDB.SetMaxIdleConns(10)
		// SetMaxOpenConns 设置打开数据库连接的最大数量。
		sqlDB.SetMaxOpenConns(20)
		// SetConnMaxLifetime 设置了连接可复用的最大时间。
		sqlDB.SetConnMaxLifetime(time.Hour)
	}
	return db
}
```

```
// 定义一个模型类
type Product struct {
	gorm.Model
	Code  string
	Price uint
}

func main() {
	db := ConnectDB()
	/*
	根据模型类生成表
	1. 表名是模型类名全小写+s
	2. 会自动添加 id(主键), created_at, updated_at, deleted_at 三个字段
	 */
	_ = db.AutoMigrate(&Product{})
	// 添加数据
	_ = db.Create(&Product{Code: "D42", Price: 100})
	// 查询数据
	var product Product
	db.First(&product, 1) // 根据整形主键查找(pk=1)
	fmt.Printf("%+v\n%", product)
	// 修改数据, 并更新updated_at
	db.Model(&product).Update("Price", 200)
	// 逻辑删除
	db.Delete(&product, 1) // 给第一条数据的deleted_at更新时间
}
```

# 2. 模型

> init.sql

```
CREATE DATABASE IF NOT EXISTS bookmanager CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;

CREATE TABLE IF NOT EXISTS `bookmanager`.`book`
(
    `id`               BIGINT(20) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '书籍id',
    `title`            VARCHAR(20)         NOT NULL COMMENT '书名',
    `publication_date` DATE COMMENT '出版时间',
    `is_delete`        TINYINT(1)          NOT NULL DEFAULT '0' COMMENT '逻辑删除',
    PRIMARY KEY (`id`),
    UNIQUE KEY (`title`)
) ENGINE = Innodb
  DEFAULT CHARSET = utf8mb4 COMMENT ='书籍信息';

CREATE TABLE IF NOT EXISTS `bookmanager`.`hero`
(
    `id`        BIGINT(20) UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '人物id',
    `name`      VARCHAR(20)         NOT NULL COMMENT '姓名',
    `gender`    TINYINT(1)          NOT NULL DEFAULT '0' COMMENT '性别',
    `book_id`   BIGINT(20) UNSIGNED NOT NULL COMMENT '书籍id',
    `is_delete` TINYINT(1)          NOT NULL DEFAULT '0' COMMENT '逻辑删除',
    PRIMARY KEY (`id`),
    CONSTRAINT `book_id` FOREIGN KEY (`book_id`) REFERENCES `book` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE = Innodb
  DEFAULT CHARSET = utf8mb4 COMMENT ='人物信息';

INSERT INTO `bookmanager`.`book` (title, publication_date)
VALUES ('西游记', '1876-03-16'),
       ('水浒传', '1963-08-30'),
       ('射雕英雄传', '1987-07-06');

INSERT INTO `bookmanager`.`hero` (name, gender, book_id)
VALUES ('孙悟空', 0, 1),
       ('白骨精', 1, 1),
       ('哪吒', 0, 1),
       ('李靖', 0, 1),
       ('李逵', 0, 2),
       ('郭靖', 0, 3),
       ('黄蓉', 1, 3);
```

## 2.1 定义模型

```
type Book struct {
	ID              int       `gorm:"primaryKey"`
	Title           string
	PublicationDate time.Time `gorm:"column:publication_date"`
	IsDelete        bool      `gorm:"column:is_delete"`
}

func (Book) TableName() string {
	return "book"
}
```

- GORM 默认会使用结构体名小写的复数形式, 作为表名生成SQL;
    - 要想自定义表名, 可以通过实现 **Tabler** 接口的 **TableName** 方法, 来更改默认表名;
- GORM 默认会使用结构体字段名小写来作为表的列名;
    - 可以通过给结构体增加 `column` 标签或 `命名策略`, 来自定义列名;
- 可以在结构体字段的标签(Tag)中, 约定字段的属性, 比如: primaryKey

### 字段标签

| 标签名                 | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ |
| column                 | 指定 db 列名                                                 |
| type                   | 列数据类型，推荐使用兼容性好的通用类型，例如：所有数据库都支持 bool、int、uint、float、string、time、bytes 并且可以和其他标签一起使用，例如：`not null`、`size`, `autoIncrement`… 像 `varbinary(8)` 这样指定数据库数据类型也是支持的。在使用指定数据库数据类型时，它需要是完整的数据库数据类型，如：`MEDIUMINT UNSIGNED not NULL AUTO_INCREMENT` |
| size                   | 指定列大小，例如：`size:256`                                 |
| **primaryKey**         | 指定列为主键                                                 |
| unique                 | 指定列为唯一                                                 |
| default                | 指定列的默认值                                               |
| precision              | 指定列的精度                                                 |
| scale                  | 指定列大小                                                   |
| not null               | 指定列为 NOT NULL                                            |
| autoIncrement          | 指定列为自动增长                                             |
| autoIncrementIncrement | 自动步长，控制连续记录之间的间隔                             |
| embedded               | 嵌套字段                                                     |
| embeddedPrefix         | 嵌入字段的列名前缀                                           |
| autoCreateTime         | 创建时追踪当前时间，对于 `int` 字段，它会追踪秒级时间戳，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoCreateTime:nano` |
| autoUpdateTime         | 创建/更新时追踪当前时间，对于 `int` 字段，它会追踪秒级时间戳，您可以使用 `nano`/`milli` 来追踪纳秒、毫秒时间戳，例如：`autoUpdateTime:milli` |
| index                  | 根据参数创建索引，多个字段使用相同的名称则创建复合索引，查看 [索引](https://gorm.io/zh_CN/docs/indexes.html) 获取详情 |
| uniqueIndex            | 与 `index` 相同，但创建的是唯一索引                          |
| check                  | 创建检查约束，例如 `check:age > 13`，查看 [约束](https://gorm.io/zh_CN/docs/constraints.html) 获取详情 |
| <-                     | 设置字段写入的权限， `<-:create` 只创建、`<-:update` 只更新、`<-:false` 无写入权限、`<-` 创建和更新权限 |
| ->                     | 设置字段读的权限，`->:false` 无读权限                        |
| -                      | 忽略该字段，`-` 无读写权限                                   |
| comment                | 迁移时为字段添加注释                                         |

## 2.2 增删改查

### 2.2.1 增加

#### 插入一条数据

```
	book := Book{
		Title:           "Python",
		PublicationDate: time.Date(2001, time.January, 30, 0, 0, 0, 0, time.Local),
		IsDelete:        false,
	}
	result := db.Create(&book)
	fmt.Println(book.ID)             // 返回插入数据的主键
	fmt.Println(result.Error)        // 返回error
	fmt.Println(result.RowsAffected) // 返回受影响的行数
	
	// 增加数据, 但只更新给出的字段; is_delete=true不回生效
	db.Select("title", "publication_date").Create(&Book{Title: "天龙八部", PublicationDate: time.Now(), IsDelete: true})
	// 增加数据, 但只更新未给出的字段; is_delete=true不会生效
	db.Omit("is_delete").Create(&Book{Title: "封神演义", PublicationDate: time.Now(), IsDelete: true})
```

#### 插入多条数据

```
	var books = []Book{{Title: "book1", PublicationDate: time.Now()},{Title: "book2", PublicationDate: time.Now()}}
	db.Create(&books)
```

#### 根据Map插入数据

GORM 支持根据 `map[string]interface{}` 和 `[]map[string]interface{}{}` 创建记录

```
	db.Model(&Book{}).Create(map[string]interface{}{"title": "map", "publication_date": time.Now()})
	
	db.Model(&Book{}).Create([]map[string]interface{}{
		{"title":"map01", "publication_date": time.Now()},
		{"title":"map02", "publication_date": time.Now()},
	})
```

### 2.2.2 查询

#### 查询一条数据

```
	var book Book
	db.First(&book) // 获取第一条数据
	db.Take(&book)  // 获取第一条数据, SELECT * FROM book LIMIT 1;
	db.Last(&book)  // 获取最后一条数据
	fmt.Println(book)
```

#### 查询全部数据

```
	var books []Book
	result := db.Find(&books)
	fmt.Println(&books)
	fmt.Println(result.RowsAffected)	// 输出查到的行数
```

#### 条件查询 where

> string 条件

```
	// 获取一条记录
	db.Where("title = ?", "西游记").First(&book)
	// SELECT * FROM book WHERE title = '西游记' ORDER BY id LIMIT 1;
	db.Where("title = ?", "西游记").Take(&book)
	// SELECT * FROM book WHERE title = '西游记' LIMIT 1;
```

```
	// 获取全部匹配的记录
	db.Where("title <> ?", "西游记").Find(&books)
	// SELECT * FROM book WHERE title <> '西游记';
	
	// IN
	db.Where("title IN ?", []string{"西游记", "水浒传", "天龙八部"}).Find(&books)
	// SELECT * FROM book WHERE title IN ('西游记', '水浒传', '天龙八部');
	
	// LIKE
	db.Where("title LIKE ?", "%book%").Find(&books)
	// SELECT * FROM book WHERE title LIKE '%book%';
	
	// AND
	db.Where("title = ? AND is_delete != ?", "西游记", 1).Find(&books)
	// SELECT * FROM book WHERE title = '西游记' AND is_delete != 1;
	
	// Time
	dateTime := time.Date(2021, time.January, 1, 0, 0, 0, 0, time.Local)
	db.Where("publication_date < ?", dateTime).Find(&books)
	// SELECT * FROM `book` WHERE publication_date < '2021-01-01 0:0:0.0';
	
	// BETWEEN
	startDate := time.Date(2021, time.January, 1, 0, 0, 0, 0, time.Local)
	db.Where("publication_date BETWEEN ? AND ?", startDate, time.Now()).Find(&books)
	// SELECT * FROM `book` WHERE publication_date BETWEEN '2021-01-01 0:0:0.0' AND '2021-05-21 01:00:00.0';
```

> Struct和Map条件

```
	// Struct
	db.Where(&Book{Title: "西游记", IsDelete: false}).Find(&books)
	// SELECT * FROM book WHERE title = '西游记' AND is_delete = 0;
	
	// Map
	db.Where(map[string]interface{}{"title": "西游记", "is_delete": false}).Find(&books)
	// SELECT * FROM book WHERE title = '西游记' AND is_delete = 0;
	
	// 主键切片条件
	db.Where([]int{1, 3, 5}).Find(&books)
	// SELECT * FROM book WHERE id IN (1,3,5);
```

#### 内联条件











## 2.3 关联查询





### 关联标签

| 标签             | 描述                                     |
| ---------------- | ---------------------------------------- |
| foreignKey       | 指定当前模型的列作为连接表的外键         |
| references       | 指定引用表的列名，其将被映射为连接表外键 |
| polymorphic      | 指定多态类型，比如模型名                 |
| polymorphicValue | 指定多态值、默认表名                     |
| many2many        | 指定连接表表名                           |
| joinForeignKey   | 指定连接表的外键列名，其将被映射到当前表 |
| joinReferences   | 指定连接表的外键列名，其将被映射到引用表 |
| constraint       | 关系约束，例如：`OnUpdate`、`OnDelete`   |

































