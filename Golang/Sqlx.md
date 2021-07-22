```shell
# sqlite3 驱动, 需要 gcc(mingw)
go get github.com/mattn/go-sqlite3
# mysql 驱动
go get github.com/jmoiron/sqlx

go get github.com/jmoiron/sqlx
```

## 连接数据库

### sqlite3

```go
var db *sqlx.DB

func initDB() (err error){
	db, err = sqlx.Open("sqlite3", "./test.db")
	if err != nil {
		log.Println(err)
		return
	}
	db.SetMaxOpenConns(20)
	db.SetMaxIdleConns(10)
	return 
}

func main() {
    _ = initDB()
    schema := `CREATE TABLE user (
		id integer PRIMARY KEY AUTOINCREMENT,
		username CHAR(20),
		password CHAR(20),
	    is_delete BOOLEAN DEFAULT FALSE);`

    if result, err := db.Exec(schema); err != nil {
        log.Println(err)
    } else {
        fmt.Println(result)
    }
}
```

## 基本使用

```go
type User struct {
	Id       int    `db:"id"`
	Username string `db:"username""`
	Password string `db:"password"`
	IsDelete bool   `db:"is_delete"`
}
```

### 查询

#### 单条数据

db.Get(&u, sqlStr, 1)

```go
func queryRow() {
	var u User
	// INSERT INTO user (username, password) VALUES ("admin", "admin");
	sqlStr := `SELECT id, username, password, is_delete from user;`
	if err := db.Get(&u, sqlStr, 1); err != nil {
		log.Println(err)
	}
	fmt.Println(u)
}
```

#### 查询多条数据

db.Select(&users, sqlStr, 0)

```go
func queryMultiRow() {
	var users []User
	sqlStr := `SELECT id, username, password, is_delete from user;`
	if err := db.Select(&users, sqlStr, 0); err != nil {
		log.Println(err)
	}
	fmt.Println(users)
}
```

### 插入 更新 删除

#### 插入数据

db.Exec(sqlStr, username, password)

```go
func insertRow(username, password string) {
	sqlStr := `INSERT INTO user (username, password) VALUES (?, ?);`
	result, err := db.Exec(sqlStr, username, password)
	if err != nil {
		log.Println("insert failed: ", err)
	}
	theID, err := result.LastInsertId()	// 获取新插入的数据的ID
	if err != nil {
		log.Println("get last insert ID failed: ", err)
	}
	fmt.Println("inster success, the ID is: ", theID)
}
```

#### 更新数据

db.Exec(sqlStr, password, id)

```go
func updateRows(id int, password string) {
	sqlStr := `UPDATE user SET password=? where id=?;`
	result, err := db.Exec(sqlStr, password, id)
	if err != nil {
		log.Println("update failed: ", err)
	}
	n, err := result.RowsAffected() // 获取操作影响的行数
	if err != nil {
		log.Println("get affected rows failed: ", err)
	}
	fmt.Println("affected rows: ", n)
}

// 逻辑删除
func logicDelete(id int) {
	sqlStr := `UPDATE user SET is_delete=True where id=?;`
	_, _ = db.Exec(sqlStr, id)
}
```

#### 删除数据

db.Exec(sqlStr, id)

```go
func deleteRow(id int) {
	sqlStr := `DELETE FROM user WHERE id=?;`
	result, err := db.Exec(sqlStr, id)
	if err != nil {
		log.Println("delete failed: ", err)
	}
	n, err := result.RowsAffected() // 获取操作影响的行数
	if err != nil {
		log.Println("get affected rows failed: ", err)
	}
	fmt.Println("delete rows: ", n)
}
```

#### NamedExec

`DB.NamedExec`方法用来绑定SQL语句与结构体或map中的同名字段

db.NamedExec(sqlStr, User{Username: username, Password: password})

```go
func insertUser(username, password string) {
	sqlStr := `INSERT INTO user (username, password) VALUES (:Username, :Password);`
	_, _ = db.NamedExec(sqlStr, User{Username: username, Password: password})
}
```

#### NamedQuery

与`DB.NamedExec`同理，这里是支持查询

**使用 map 做命名查询**

db.NamedQuery(sqlStr, map[string]interface{}{"user": "admin"})

```go
func namedQueryMap() {
	sqlStr := `SELECT id, username, password, is_delete FROM user WHERE username=:user;`
	rows, err := db.NamedQuery(sqlStr, map[string]interface{}{"user": "admin"})
	if err != nil {
		log.Println("db.NamedQuery Map failed: ", err)
	}
	defer rows.Close()
	// 遍历查询结果
	for rows.Next() {
		var u User
		if err := rows.StructScan(&u); err != nil {
			log.Println("scan failed: ", err)
		} else {
			fmt.Println("user: ", u)
		}
	}
}
```

**使用 struct 做命名查询**

注意 sql中的占位字段 和 结构体 注解中的名字对应

db.NamedQuery(sqlStr, User{Username: "admin"})

```go
func namedQueryStruct() {
	sqlStr := `SELECT id, username, password, is_delete FROM user WHERE username=:username;`
	rows, err := db.NamedQuery(sqlStr, User{Username: "admin"})
	if err != nil {
		log.Println("db.NamedQuery Struct failed: ", err)
	}
	defer rows.Close()
	// 遍历查询结果
	for rows.Next() {
		var u User
		if err := rows.StructScan(&u); err != nil {
			log.Println("scan failed: ", err)
		} else {
			fmt.Println("user: ", u)
		}
	}
}
```

### 事务操作

对于事务操作，我们可以使用`sqlx`中提供的`db.Beginx()`和`tx.Exec()`方法

```go
func transactionDemo() (err error) {
	tx, err := db.Begin() // 开启事务
	if err != nil {
		log.Println("begin transaction failed: ", err)
		return err
	}
	defer func() {
		if p := recover(); p != nil { // 类似try...catch, panic处理
			tx.Rollback()
			panic(p) // 在 Rollback 后 panic 异常
		} else if err != nil {
			log.Println("roll back")
			tx.Rollback() // err 不是 nil, 执行 rollback
		} else {
			tx.Commit()
			fmt.Println("commit")
		}
	}()

	sqlStr := `UPDATE user SET password=? WHERE id=?;`
    
	result, err := tx.Exec(sqlStr, "111", 3)
	if err != nil {
		return err
	}
	n, err := result.RowsAffected()
	if err != nil {
		return err
	}
	if n != 1 {
		return errors.New("exec sqlStr failed.")
	}

	result, err = tx.Exec(sqlStr, "222", 3)
	if err != nil {
		return err
	}
	n, err = result.RowsAffected()
	if err != nil {
		return err
	}
	if n != 1 {
		return errors.New("exec sqlStr failed.")
	}

	result, err = tx.Exec(sqlStr, "333", 3)
	if err != nil {
		return err
	}
	n, err = result.RowsAffected()
	if err != nil {
		return err
	}
	if n != 1 {
		return errors.New("exec sqlStr failed.")
	}
	return err
}
```

### sqlx.In

复用之前的表和User结构体

```sql
CREATE TABLE user (
    id integer PRIMARY KEY AUTOINCREMENT,
    username CHAR(20),
    password CHAR(20),
    is_delete BOOLEAN DEFAULT FALSE
);
```

```go
type User struct {
	Id       int    `db:"id"`
	Username string `db:"username""`
	Password string `db:"password"`
	IsDelete bool   `db:"is_delete"`
}
```

**bindvars（绑定变量）**

查询占位符`?`在内部称为**bindvars（查询占位符）**,它非常重要。

你应该始终使用它们向数据库发送值，因为它们可以防止SQL注入攻击。`database/sql`不尝试对查询文本进行任何验证；它与编码的参数一起按原样发送到服务器。

除非驱动程序实现一个特殊的接口，否则在执行之前，查询是在服务器上准备的。

因此`bindvars`是特定于数据库的:

- MySQL中使用`?`
- PostgreSQL使用枚举的`$1`、`$2`等bindvar语法
- SQLite中`?`和`$1`的语法都支持
- Oracle中使用`:name`的语法

`bindvars`的一个常见误解是，它们用来在sql语句中插入值。

它们其实仅用于参数化，不允许更改SQL语句的结构。例如，使用`bindvars`尝试参数化列或表名将不起作用：

```go
// ？不能用来插入表名（做SQL语句中表名的占位符）
db.Query("SELECT * FROM ?", "mytable")
 
// ？也不能用来插入列名（做SQL语句中列名的占位符）
db.Query("SELECT ?, ? FROM people", "name", "location")
```

**使用sqlx.In实现批量插入**

