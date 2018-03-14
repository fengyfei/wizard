# database/sql

Go 标准库中的一个包，用于和 SQL 或 SQL-Like 数据库通讯。该包提供的是抽象概念，具体数据库的实现，由驱动来做，这样就可以方便的更换数据库。

---

##简单例子

```go
package main

import (
	"database/sql"
    "fmt"
	"log"

	_ "github.com/go-sql-driver/mysql"
)

func main() {
	db, err := sql.Open("mysql", "user:password@/dbName")  // 创建数据库实例
	if err != nil {
		panic(err)
	}

	defer db.Close()

    if err = db.Ping(); err != nil {    // Ping: 判断实例是否可用
		log.Fatal(err)
	}

    rows, err := db.Query("SELECT name FROM tableName")    // Query: 查询功能
	if err != nil {
		log.Fatal("Query error: ", err)
	}

	for rows.Next() {
		var name string
        if err := rows.Scan(&name); err != nil {    // Scan: 判断值是否存在，存在将其存入 name 中
			log.Fatal("Scan error: ", err)
		}
		fmt.Println(name)
	}
	if err := rows.Err(); err != nil {
		log.Fatal(err)
	}
}

```

---

##Variables

-   **ErrConnDone**

    ```go
    var ErrConnDone = errors.New("database/sql: connection is already closed")
    ```

-   **ErrNoRows**

    ```go
    var ErrNoRows = errors.New("sql: no rows in result set")
    ```

-   **ErrTxDone**

    ```go
    var ErrTxDone = errors.New("sql: Transaction has already been committed or rolled back")
    ```

---

##func Drivers() [\]string

**返回已注册的驱动列表**

---

##func Register(name string, driver driver.Driver)

**注册驱动，对于同一个驱动若注册两次或无此驱动，程序异常**

---

##Struct

###ColumnType

**列的名字和类型**

```go
// 返回数据库系统名和列类型
func (ci *ColumnType) DatabaseTypeName() string
```

```go
// 返回小数类型的的规模和精度
func (ci *ColumnType) DecimalSize() (precision, scale int64, ok bool)
```

```go
// 返回可变长度列类型(如文本和二进制字段类型)的列类型长度，若长度无限，则该值为 math.MaxInt64
func (ci *ColumnType) Length() (length int64, ok bool)
```

```go
// 返回列的名字或者别名
func (ci *ColumnType) Name() string
```

```go
// 判断该列是否为空
func (ci *ColumnType) Nullable() (nullable, ok bool)
```

```go
// 返回一个适合于使用 Rows.Scan 扫描的Go类型
func (ci *ColumnType) ScanType() reflect.Type
```

### Conn

**表示单个数据库连接，而非池**

**必须调用 Close 将此连接归还给连接池，在此之后，在此连接上的任何操作都会返回错误——ErrConnDone**

```go
// 启动一个事务，提供的 context 在该事务提交或回滚前一直会被使用，如果该 context 被取消，该事务将会回滚，Tx.Commit 将会返回一个错误
func (c *Conn) BeginTx(ctx context.Context, opts *TxOptions) (*Tx, error)
```

```go
// 归还连接，可以安全地与其他的操作同时调用，在其他所有操作结束前将会一直阻塞
func (c *Conn) Close() error
```

```go
// 执行查询，但不返回行
func (c *Conn) ExecContext(ctx context.Context, query string, args ...interface{}) (Result, error)
```

```go
// 验证该数据库连接是否可用
func (c *Conn) PingContext(ctx context.Context) error
```

```go
// 为之后的查询和执行创造 statement，当该 statement 不再使用时必须调用它的 Close 方法
func (c *Conn) PrepareContext(ctx context.Context, query string) (*Stmt, error)
```

```go
// 执行查询返回结果，同等于 SELECT
func (c *Conn) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error)
```

```go
// 执行查询只返回第一个结果，若无结果，*Row's Scan 将会返回错误——ErrNoRows
func (c *Conn) QueryRowContext(ctx context.Context, query string, args ...interface{}) *Row
```

### DB

**代表了一个数据库。这点和很多其他语言不同，它并不代表一个到数据库的具体连接，而是一个能操作的数据库对象，具体的连接在内部通过连接池来管理，对外不暴露。每一次数据库操作，都产生一个 sql.DB 实例，操作完 Close**

```go
func Open(driverName, dataSourceName string) (*DB, error)
```

```go
func OpenDB(c driver.Connector) *DB
```

```go
func (db *DB) Begin() (*Tx, error)
```

```go
func (db *DB) BeginTx(ctx context.Context, opts *TxOptions) (*Tx, error)
```

```go
func (db *DB) Close() error
```

```go
func (db *DB) Conn(ctx context.Context) (*Conn, error)
```

```go
func (db *DB) Driver() driver.Driver
```

```go
func (db *DB) Exec(query string, args ...interface{}) (Result, error)
```

```go
func (db *DB) ExecContext(ctx context.Context, query string, args ...interface{}) (Result, error)
```

```go
func (db *DB) Ping() error
```

```go
func (db *DB) PingContext(ctx context.Context) error
```

```go
func (db *DB) Prepare(query string) (*Stmt, error)
```

```go
func (db *DB) PrepareContext(ctx context.Context, query string) (*Stmt, error)
```

```go
func (db *DB) Query(query string, args ...interface{}) (*Rows, error)
```

```go
func (db *DB) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error)
```

```go
func (db *DB) QueryRow(query string, args ...interface{}) *Row
```

```go
func (db *DB) QueryRowContext(ctx context.Context, query string, args ...interface{}) *Row
```

```go
func (db *DB) SetConnMaxLifetime(d time.Duration)
```

```go
func (db *DB) SetMaxIdleConns(n int)
```

```go
func (db *DB) SetMaxOpenConns(n int)
```

```go
func (db *DB) Stats() DBStats
```

### Row

**一行结果**

```go
func (r *Row) Scan(dest ...interface{}) error
```

### Rows

**多行结果**

```go
func (rs *Rows) Close() error
```

```go
func (rs *Rows) ColumnTypes() ([]*ColumnType, error)
```

```go
func (rs *Rows) Columns() ([]string, error)
```

```go
func (rs *Rows) Err() error
```

```go
func (rs *Rows) Next() bool
```

```go
func (rs *Rows) NextResultSet() bool
```

```go
func (rs *Rows) Scan(dest ...interface{}) error
```

### Stmt

**语句，如：DDL、DML 等**

```go
func (s *Stmt) Close() error
```

```go
func (s *Stmt) Exec(args ...interface{}) (Result, error)
```

```go
func (s Stmt) ExecContext(ctx context.Context, args ...interface{}) (Result, error)
```

```go
func (s *Stmt) Query(args ...interface{}) (Rows, error)
```

```go
func (s *Stmt) QueryContext(ctx context.Context, args ...interface{}) (*Rows, error)
```

```go
func (s *Stmt) QueryRow(args ...interface{}) *Row
```

```go
func (s *Stmt) QueryRowContext(ctx context.Context, args ...interface{}) *Row
```

### Tx

**带有特定属性的一个事务**