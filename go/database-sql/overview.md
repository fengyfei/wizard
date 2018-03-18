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

**必须调用 Close 将此连接归还给连接池，在此之后，在此连接上的任何操作都会返回错误 ErrConnDone**

```go
// 启动一个事务，提供的 context 在该事务提交或回滚前一直会被使用，如果该 context 被删去，该事务将会回滚，Tx.Commit 将会返回一个错误
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
// 执行查询只返回第一个结果，若无结果，*Row's Scan 将会返回错误 ErrNoRows
func (c *Conn) QueryRowContext(ctx context.Context, query string, args ...interface{}) *Row
```

### DB

**代表了一个数据库。这点和很多其他语言不同，它并不代表一个到数据库的具体连接，而是一个能操作的数据库对象，具体的连接在内部通过连接池来管理，对外不暴露。每一次数据库操作，都产生一个 sql.DB 实例，操作完 Close**

```go
// 以数据库驱动名和数据源为参数，返回一个数据库实例，驱动得另外导入，此方法应只调用一次
func Open(driverName, dataSourceName string) (*DB, error)
```

```go
// 以一个连接为参数，返回数据库实例
func OpenDB(c driver.Connector) *DB
```

```go
// 启动一个事务，默认的隔离级别取决于驱动
func (db *DB) Begin() (*Tx, error)
```

```go
// 启动一个事务，所使用的 context 将会一直被使用直到该事务提交或回滚，如果 context 被删去，该事务将回滚，Tx.Commit 方法将会返回错误
func (db *DB) BeginTx(ctx context.Context, opts *TxOptions) (*Tx, error)
```

```go
// 关闭数据库，释放资源(很少使用)
func (db *DB) Close() error
```

```go
// 返回一个单一的连接，在连接返还或 context 被删去前，它将会一直阻塞，每个连接在使用完后必须调用 Conn.Close 方法将其归还给连接池
func (db *DB) Conn(ctx context.Context) (*Conn, error)
```

```go
// 返回数据库的驱动
func (db *DB) Driver() driver.Driver
```

```go
// 执行查询，不返回任何结果
func (db *DB) Exec(query string, args ...interface{}) (Result, error)
```

```go
// 同上
func (db *DB) ExecContext(ctx context.Context, query string, args ...interface{}) (Result, error)
```

```go
// 判断一个数据库连接是否可用，如果有必要则建立连接
func (db *DB) Ping() error
```

```go
// 同上
func (db *DB) PingContext(ctx context.Context) error
```

```go
// 为之后的查询和执行创造 statement，当该 statement 不再使用时必须调用它的 Close 方法
func (db *DB) Prepare(query string) (*Stmt, error)
```

```go
// 同上
func (db *DB) PrepareContext(ctx context.Context, query string) (*Stmt, error)
```

```go
// 执行查询返回结果，同等于 SELECT
func (db *DB) Query(query string, args ...interface{}) (*Rows, error)
```

```go
// 同上
func (db *DB) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error)
```

```go
// 执行查询只返回第一个结果，若无结果，*Row's Scan 将会返回错误 ErrNoRows
func (db *DB) QueryRow(query string, args ...interface{}) *Row
```

```go
// 执行查询只返回第一个结果，若无结果，*Row's Scan 将会返回错误 ErrNoRows
func (db *DB) QueryRowContext(ctx context.Context, query string, args ...interface{}) *Row
```

```go
// 设置一个连接被重用的最大时间，若 d <= 0，将会一直被重用
func (db *DB) SetConnMaxLifetime(d time.Duration)
```

```go
// 设置连接池中闲置连接的最大数量，若 0 < MaxOpenConns < MaxIdleConns，MaxIdleConns 将会减少至与 MaxOpenConns 相等，若 n <= 0，没有闲置连接会被保留
func (db *DB) SetMaxIdleConns(n int)
```

```go
// 设置数据库连接的最大数量，若 n <= 0，该数量将没有限制
func (db *DB) SetMaxOpenConns(n int)
```

```go
// 返回数据库的统计资料
func (db *DB) Stats() DBStats
```

### Row

**一行结果**

```go
// 将匹配到的第一个数据传给 dest，如果没有匹配的数据，返回错误 ErrNoRows
func (r *Row) Scan(dest ...interface{}) error
```

### Rows

**多行结果**

```go
// 关闭 Rows，防止进一步的枚举
func (rs *Rows) Close() error
```

```go
// 返回列的信息，如：类型、长度、是否为空
func (rs *Rows) ColumnTypes() ([]*ColumnType, error)
```

```go
// 返回列名，当 Rows 关闭时返回错误
func (rs *Rows) Columns() ([]string, error)
```

```go
// 返回错误
func (rs *Rows) Err() error
```

```go
// 判断是否还存在下一个数据
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

```go
func (tx *Tx) Commit() error
```

```go
func (tx *Tx) Exec(query string, args ...interface{}) (Result, error)
```

```go
func (tx *Tx) ExecContext(ctx context.Context, query string, args ...interface{}) (Result, error)
```

```go
func (tx *Tx) Prepare(query string) (*Stmt, error)
```

```go
func (tx *Tx) PrepareContext(ctx context.Context, query string) (*Stmt, error)
```

```go
func (tx *Tx) Query(query string, args ...interface{}) (*Rows, error)
```

```go
func (tx *Tx) QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error)
```

```go
func (tx *Tx) QueryRow(query string, args ...interface{}) *Row
```

```go
func (tx *Tx) QueryRowContext(ctx context.Context, query string, args ...interface{}) *Row
```

```go
func (tx *Tx) Rollback() error
```

```go
func (tx *Tx) Stmt(stmt *Stmt) *Stmt
```

```go
func (tx *Tx) StmtContext(ctx context.Context, stmt *Stmt) *Stmt
```