## 深入理解SQL驱动
database/sql是go里面操作数据库的，这里我们主要讲解databas/sql和mysql使用

### 数据结构

#### DB
DB是数据库结构，是并发安全的
```go
type DB struct {
    // 等待新连接的总时间(放在头部保证内存对齐)
	waitDuration int64 // Total time waited for new connections.

	connector driver.Connector
	// 连接关闭的次数
	numClosed uint64

	// 锁
	mu           sync.Mutex
	// 空闲的连接
    freeConn     []*driverConn
    // 处理连接请求，用自增id
    connRequests map[uint64]chan connRequest
    // 下一个连接请求使用的id
    nextRequest  uint64 // Next key to use in connRequests.
    // 打开和pending的连接数量
	numOpen      int    // number of opened and pending open connections
	// Used to signal the need for new connections
	// a goroutine running connectionOpener() reads on this chan and
	// maybeOpenNewConnections sends on the chan (one send per needed connection)
	// It is closed during db.Close(). The close tells the connectionOpener
    // goroutine to exit.
    // 打开ch， 
    //connectionOpener 读取数据， maybeOpenNewConnections 发送数据，缓存1000000
    openerCh          chan struct{}
    
    resetterCh        chan *driverConn
    // 是否close
	closed            bool
	dep               map[finalCloser]depSet
    lastPut           map[*driverConn]string // stacktrace of last conn's put; debug only
    // 最大空闲连接数，0和负数都是defaultMaxIdleConns， 默认是2
    maxIdle           int                    // zero means defaultMaxIdleConns; negative means 0
    // 最大连接打开数量， 0是无限制
    maxOpen           int                    // <= 0 means unlimited
    // 连接重用的最大时间
	maxLifetime       time.Duration          // maximum amount of time a connection may be reused
    cleanerCh         chan struct{}
    // 等待连接的总次数
    waitCount         int64 // Total number of connections waited for.
    // 由于空闲而关闭的连接数
    maxIdleClosed     int64 // Total number of connections closed due to idle.
    // 达到最大空闲数量而关闭的连接数
	maxLifetimeClosed int64 // Total number of connections closed due to max free limit.

    // stop会取消
	stop func() // stop cancels the connection opener and the session resetter.
}

```

#### driverConn
driverConn 是driver.Conn的包装，包括对数据库的操作
```go
type driverConn struct {
    db        *DB
    // 创建的时间
	createdAt time.Time

    // 保证下面的并发处理
	sync.Mutex  // guards following
    ci          driver.Conn
    // 是否close
    closed      bool
    // 最终关闭
	finalClosed bool // ci.Close has been called
    openStmt    map[*driverStmt]bool
    // 捕获最后一次执行结果的error
	lastErr     error // lastError captures the result of the session resetter.

    // 通过db.mu 提供并发支持
    inUse      bool
    
	onPut      []func() // code (with db.mu held) run when conn is next returned
	dbmuClosed bool     // same as closed, but guarded by db.mu, for removeClosedStmtLocked
}
```

#### Conn
Conn是一个单独的数据库连接
```go
type Conn struct {
	db *DB

	// closemu prevents the connection from closing while there
	// is an active query. It is held for read during queries
	// and exclusively during close.
	closemu sync.RWMutex

	// dc is owned until close, at which point
	// it's returned to the connection pool.
	dc *driverConn

	// done transitions from 0 to 1 exactly once, on close.
	// Once done, all operations fail with ErrConnDone.
	// Use atomic operations on value when checking value.
	done int32
}
```

#### DBStats
DBStats数据库的状态
```go
type DBStats struct {
    // 最大打开连接数
	MaxOpenConnections int // Maximum number of open connections to the database.

    // 连接池状态
    // 连接使用中和空闲的数量
    OpenConnections int // The number of established connections both in use and idle.
    // 连接使用中的数量
    InUse           int // The number of connections currently in use.
    // 连接的数量
	Idle            int // The number of idle connections.

    // 统计过
    // 等待连接的总数
    WaitCount         int64         // The total number of connections waited for.
    // 等待连接阻塞的总时间
    WaitDuration      time.Duration // The total time blocked waiting for a new connection.
    // 连接空闲被关闭的总数
    MaxIdleClosed     int64         // The total number of connections closed due to SetMaxIdleConns.
    // 由于达到MaxLifetime而关闭的总数
	MaxLifetimeClosed int64         // The total number of connections closed due to SetConnMaxLifetime.
}
```

### 源码解读
下面进入源码解读阶段

#### Open
Open会打开一个数据库， 返回一个DB指针，是并发安全的。
```go
func Open(driverName, dataSourceName string) (*DB, error) {
    // 根据driverName获取driver
	driversMu.RLock()
	driveri, ok := drivers[driverName]
	driversMu.RUnlock()
	if !ok {
		return nil, fmt.Errorf("sql: unknown driver %q (forgotten import?)", driverName)
	}

    // DriverContext是一个实现了OpenConnector方法的接口，返回是一个Connector
	if driverCtx, ok := driveri.(driver.DriverContext); ok {
		connector, err := driverCtx.OpenConnector(dataSourceName)
		if err != nil {
			return nil, err
		}
		return OpenDB(connector), nil
	}

	return OpenDB(dsnConnector{dsn: dataSourceName, driver: driveri}), nil
}
```

#### OpenDB
打开一个数据库链接
```go
func OpenDB(c driver.Connector) *DB {
	ctx, cancel := context.WithCancel(context.Background())
	db := &DB{
		connector:    c,
		openerCh:     make(chan struct{}, connectionRequestQueueSize),
		resetterCh:   make(chan *driverConn, 50),
		lastPut:      make(map[*driverConn]string),
		connRequests: make(map[uint64]chan connRequest),
		stop:         cancel,
	}

    // 启动一个单独的goroutine去处理新建连接请求
	go db.connectionOpener(ctx)
	go db.connectionResetter(ctx)

	return db
}
```

#### connectionOpener

```go
func (db *DB) connectionOpener(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		case <-db.openerCh:
			db.openNewConnection(ctx)
		}
	}
}
```

#### openNewConnection
打开一个连接
```go
func (db *DB) openNewConnection(ctx context.Context) {
	// maybeOpenNewConnctions has already executed db.numOpen++ before it sent
	// on db.openerCh. This function must execute db.numOpen-- if the
    // connection fails or is closed before returning.
    // 获取一个新的连接
	ci, err := db.connector.Connect(ctx)
	db.mu.Lock()
	defer db.mu.Unlock()
	if db.closed {
        // 数据库已经关闭了，需要关闭刚刚创建的连接
		if err == nil {
			ci.Close()
		}
		db.numOpen--
		return
	}
	if err != nil {
        db.numOpen--
        // 处理这个连接
		db.putConnDBLocked(nil, err)
		db.maybeOpenNewConnections()
		return
	}
	dc := &driverConn{
		db:        db,
		createdAt: nowFunc(),
		ci:        ci,
	}
	if db.putConnDBLocked(dc, err) {
		db.addDepLocked(dc, dc)
	} else {
		db.numOpen--
		ci.Close()
	}
}
```

#### putConnDBLocked
如果err!=nil, dc会被忽略，并且返回false
如果err==nil, dc必须不为nil

连接处理有两种情况：
1. 如果同时有连接请求，则直接将dc会发给这个连接请求方
2. 否则如果当前空闲连接小于最大空闲连接，则将连接加入空闲连接，否则将maxIdleClosed++
```go
func (db *DB) putConnDBLocked(dc *driverConn, err error) bool {
	if db.closed {
		return false
    }
    // 超过最大连接
	if db.maxOpen > 0 && db.numOpen > db.maxOpen {
		return false
    }
    // 如果同时有连接请求
	if c := len(db.connRequests); c > 0 {
		var req chan connRequest
		var reqKey uint64
		for reqKey, req = range db.connRequests {
			break
        }
        // 从待连接的map中删除这个key
		delete(db.connRequests, reqKey) // Remove from pending requests.
		if err == nil {
            // 将dc标记为使用中
			dc.inUse = true
        }
        // 直接发送过去
		req <- connRequest{
			conn: dc,
			err:  err,
		}
		return true
	} else if err == nil && !db.closed {
		if db.maxIdleConnsLocked() > len(db.freeConn) {
            // 计入到空闲连接池
            db.freeConn = append(db.freeConn, dc)
            // 开始清理处理
			db.startCleanerLocked()
			return true
		}
		db.maxIdleClosed++
	}
	return false
}
```

#### startCleanerLocked
空闲连接放入free list，判断是否有连接需要清理
```go
func (db *DB) startCleanerLocked() {
	if db.maxLifetime > 0 && db.numOpen > 0 && db.cleanerCh == nil {
		db.cleanerCh = make(chan struct{}, 1)
		go db.connectionCleaner(db.maxLifetime)
	}
}
```

#### connectionCleaner
按照maxLifetime清理过期连接
```go
func (db *DB) connectionCleaner(d time.Duration) {
	const minInterval = time.Second

	if d < minInterval {
		d = minInterval
	}
	t := time.NewTimer(d)

	for {
		select {
		case <-t.C:
		case <-db.cleanerCh: // maxLifetime was changed or db was closed.
		}

		db.mu.Lock()
        d = db.maxLifetime
        // 检查环境
		if db.closed || db.numOpen == 0 || d <= 0 {
			db.cleanerCh = nil
			db.mu.Unlock()
			return
		}
        // 过期时间
		expiredSince := nowFunc().Add(-d)
		var closing []*driverConn
		for i := 0; i < len(db.freeConn); i++ {
			c := db.freeConn[i]
			if c.createdAt.Before(expiredSince) {
                // 早于过期时间，加入到待清理队列
				closing = append(closing, c)
                last := len(db.freeConn) - 1
                // 将最后一个空闲连接放入到索引为i的位置，并且删除
				db.freeConn[i] = db.freeConn[last]
				db.freeConn[last] = nil
				db.freeConn = db.freeConn[:last]
				i--
			}
        }
        // 计数++
		db.maxLifetimeClosed += int64(len(closing))
		db.mu.Unlock()

        // 开始关闭
		for _, c := range closing {
			c.Close()
		}

		if d < minInterval {
			d = minInterval
        }
        // 重置定时器
		t.Reset(d)
	}
}
```

#### Conn
打开一个连接，阻塞获取
```go
func (db *DB) Conn(ctx context.Context) (*Conn, error) {
	var dc *driverConn
	var err error
    // 最大失败次数，默认是2
	for i := 0; i < maxBadConnRetries; i++ {
        // cachedOrNewConn是获取策略，有两种cachedOrNewConn和alwaysNewConn
		dc, err = db.conn(ctx, cachedOrNewConn)
		if err != driver.ErrBadConn {
			break
		}
	}
	if err == driver.ErrBadConn {
		dc, err = db.conn(ctx, alwaysNewConn)
	}
	if err != nil {
		return nil, err
	}

	conn := &Conn{
		db: db,
		dc: dc,
	}
	return conn, nil
}
```

#### conn
conn获取一个新连接或者从idle中获取
```go
func (db *DB) conn(ctx context.Context, strategy connReuseStrategy) (*driverConn, error) {
    db.mu.Lock()
    // 数据库已关闭
	if db.closed {
		db.mu.Unlock()
		return nil, errDBClosed
	}
    // 检查ctx是否过去
	select {
	default:
	case <-ctx.Done():
		db.mu.Unlock()
		return nil, ctx.Err()
	}
	lifetime := db.maxLifetime

    // 优先从空闲队列中读取
	numFree := len(db.freeConn)
	if strategy == cachedOrNewConn && numFree > 0 {
        // 获取空闲连接，并且从freeConn中删除
		conn := db.freeConn[0]
		copy(db.freeConn, db.freeConn[1:])
		db.freeConn = db.freeConn[:numFree-1]
		conn.inUse = true
        db.mu.Unlock()
        // 如果连接过期了，则返回ErrBadConn
		if conn.expired(lifetime) {
			conn.Close()
			return nil, driver.ErrBadConn
		}
        // Lock around reading lastErr to ensure the session resetter finished.
        // 加锁读取lastErr
		conn.Lock()
		err := conn.lastErr
        conn.Unlock()
        // ErrBadConn 返回
		if err == driver.ErrBadConn {
			conn.Close()
			return nil, driver.ErrBadConn
        }
        // 获取成功，返回
		return conn, nil
	}

    // 超过打开最大连接数
	if db.maxOpen > 0 && db.numOpen >= db.maxOpen {
        // 创建连接channel，有缓存
        req := make(chan connRequest, 1)
        // 获取下一个key，自增
		reqKey := db.nextRequestKeyLocked()
		db.connRequests[reqKey] = req
		db.waitCount++
		db.mu.Unlock()

        // 等待开始时间
		waitStart := time.Now()

        // 到达context的时间会超时
		select {
		case <-ctx.Done():
			// Remove the connection request and ensure no value has been sent
            // on it after removing.
            // 超时或者被取消，删除这个连接请求
			db.mu.Lock()
			delete(db.connRequests, reqKey)
			db.mu.Unlock()

            // 更新
			atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))

			select {
			default:
            case ret, ok := <-req:
                // 在获取一下
				if ok && ret.conn != nil {
					db.putConn(ret.conn, ret.err, false)
				}
			}
			return nil, ctx.Err()
        case ret, ok := <-req:
            // 获取成功，更新时间
			atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))
            // 连接关闭, db.Close
			if !ok {
				return nil, errDBClosed
            }
            // 达到maxLifeTime，返回ErrBadConn
			if ret.err == nil && ret.conn.expired(lifetime) {
				ret.conn.Close()
				return nil, driver.ErrBadConn
            }
            // 连接错误
			if ret.conn == nil {
				return nil, ret.err
			}
			// Lock around reading lastErr to ensure the session resetter finished.
			ret.conn.Lock()
			err := ret.conn.lastErr
			ret.conn.Unlock()
			if err == driver.ErrBadConn {
				ret.conn.Close()
				return nil, driver.ErrBadConn
            }
            // 获取成功
			return ret.conn, ret.err
		}
	}
    // 依然没有获取到，到这里都++
	db.numOpen++ // optimistically
    db.mu.Unlock()
    // 连接
	ci, err := db.connector.Connect(ctx)
	if err != nil {
		db.mu.Lock()
		db.numOpen-- // correct for earlier optimism
		db.maybeOpenNewConnections()
		db.mu.Unlock()
		return nil, err
	}
	db.mu.Lock()
	dc := &driverConn{
		db:        db,
		createdAt: nowFunc(),
		ci:        ci,
		inUse:     true,
	}
	db.addDepLocked(dc, dc)
	db.mu.Unlock()
	return dc, nil
}
```

#### maybeOpenNewConnections
如果没有达到maxOpen，打开一个新的连接
```go
func (db *DB) maybeOpenNewConnections() {
	numRequests := len(db.connRequests)
	if db.maxOpen > 0 {
		numCanOpen := db.maxOpen - db.numOpen
		if numRequests > numCanOpen {
			numRequests = numCanOpen
		}
	}
	for numRequests > 0 {
		db.numOpen++ // optimistically
		numRequests--
		if db.closed {
			return
        }
        // 写入连接请求
		db.openerCh <- struct{}{}
	}
}
```

#### connectionOpener
connectionOpener运行一个单独的goroutine
```go
func (db *DB) connectionOpener(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		case <-db.openerCh:
			db.openNewConnection(ctx)
		}
	}
}

```

总结：
数据库连接源码比较简单，接下来我们看下go使用数据库连接另一个要注意的结构


### Rows
Rows 是一个查询的结构，游标从结果的第一行开始，用Next方法可以从一行到下一行
```go
type Rows struct {
    // 当前的dc，关闭后会调用releaseConn释放
    dc          *driverConn // owned; must call releaseConn when closed to release
    // 释放连接函数
	releaseConn func(error)
    rowsi       driver.Rows
    // 当Rows 关闭后调用
	cancel      func()      // called when Rows is closed, may be nil.
	closeStmt   *driverStmt // if non-nil, statement to Close on close

    // closemu 锁
    closemu sync.RWMutex
    // 是否关闭
    closed  bool
    // 只有在closed的情况是非nil
	lasterr error // non-nil only if closed is true

	// lastcols is only used in Scan, Next, and NextResultSet which are expected
	// not to be called concurrently.
	lastcols []driver.Value
}
```

Rows查询完需要Close, 为什么需要Close? 接下来我们看下
#### Close
Rows的Close会调用close去关闭
```go
func (rs *Rows) close(err error) error {
	rs.closemu.Lock()
	defer rs.closemu.Unlock()

	if rs.closed {
		return nil
    }
    // 标记为关闭
	rs.closed = true

	if rs.lasterr == nil {
		rs.lasterr = err
	}

    // 关闭
	withLock(rs.dc, func() {
		err = rs.rowsi.Close()
	})
	if fn := rowsCloseHook(); fn != nil {
		fn(rs, &err)
	}
	if rs.cancel != nil {
		rs.cancel()
	}

	if rs.closeStmt != nil {
		rs.closeStmt.Close()
    }
    // 释放连接
	rs.releaseConn(err)
	return err
}
```

#### releaseConn
releaseConn 就是driverConn.releaseConn函数

```go
func (dc *driverConn) releaseConn(err error) {
	dc.db.putConn(dc, err, true)
}
```

#### putConn
释放连接
```go
func (db *DB) putConn(dc *driverConn, err error, resetSession bool) {
	db.mu.Lock()
	if !dc.inUse {
		if debugGetPut {
			fmt.Printf("putConn(%v) DUPLICATE was: %s\n\nPREVIOUS was: %s", dc, stack(), db.lastPut[dc])
		}
		panic("sql: connection returned that was never out")
	}
	if debugGetPut {
		db.lastPut[dc] = stack()
    }
    // 修改为未使用
	dc.inUse = false

	for _, fn := range dc.onPut {
		fn()
	}
	dc.onPut = nil
    // 当前连接是ErrBadConn， 重新打开一个连接
	if err == driver.ErrBadConn {
		// Don't reuse bad connections.
		// Since the conn is considered bad and is being discarded, treat it
		// as closed. Don't decrement the open count here, finalClose will
		// take care of that.
		db.maybeOpenNewConnections()
		db.mu.Unlock()
		dc.Close()
		return
	}
	if putConnHook != nil {
		putConnHook(db, dc)
	}
	if db.closed {
		// Connections do not need to be reset if they will be closed.
		// Prevents writing to resetterCh after the DB has closed.
		resetSession = false
	}
	if resetSession {
		if _, resetSession = dc.ci.(driver.SessionResetter); resetSession {
			// Lock the driverConn here so it isn't released until
			// the connection is reset.
			// The lock must be taken before the connection is put into
			// the pool to prevent it from being taken out before it is reset.
			dc.Lock()
		}
    }
    // 加入队列
	added := db.putConnDBLocked(dc, nil)
	db.mu.Unlock()

	if !added {
		if resetSession {
			dc.Unlock()
		}
		dc.Close()
		return
	}
	if !resetSession {
		return
	}
	select {
	default:
		// If the resetterCh is blocking then mark the connection
		// as bad and continue on.
		dc.lastErr = driver.ErrBadConn
		dc.Unlock()
	case db.resetterCh <- dc:
	}
}
```
看了上面应该明白为什么Rows需要关闭了，如果不关闭，连接就不会释放，这样打开的连接就是一直增加，如果一旦超过了maxOpen，就会导致后面的数据库操作阻塞。