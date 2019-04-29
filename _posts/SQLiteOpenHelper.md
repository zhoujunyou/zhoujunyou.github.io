###android.database.sqlite

对照ODBC流程源码分析。未进入native分析

1. **配置数据源**

2. **初始化环境**

   安卓中使用数据库一般先创建一个SQLiteOpenHelper的子类。传入上下文，数据库名，版本号。使用数据库时通过其getWritableDatabase方法获取到SQLiteDatabase对象。分析下面方法  配置数据源

   ```java
   //com.android.database.sqlite.SQLiteOpenHelper
   private SQLiteDatabase getDatabaseLocked(boolean writable) {
       ...
          
                           db = mContext.openOrCreateDatabase(mName, mEnableWriteAheadLogging ?
                                   Context.MODE_ENABLE_WRITE_AHEAD_LOGGING : 0,
                                   mFactory, mErrorHandler);
                  
       ...
             
       }
   
   
   //android.app.ContextImpl  创建或打开数据库文件 这就是数据源配置过程
   @Override
       public SQLiteDatabase openOrCreateDatabase(String name, int mode, CursorFactory factory,
               DatabaseErrorHandler errorHandler) {
           File f = validateFilePath(name, true);
           int flags = SQLiteDatabase.CREATE_IF_NECESSARY;
           if ((mode & MODE_ENABLE_WRITE_AHEAD_LOGGING) != 0) {
               flags |= SQLiteDatabase.ENABLE_WRITE_AHEAD_LOGGING;
           }
           SQLiteDatabase db = SQLiteDatabase.openDatabase(f.getPath(), factory, flags, errorHandler);
           setFilePermissionsFromMode(f.getPath(), mode, 0);
           return db;
       }
   
   ```

3. **建立连接**

   java层最终到SQLiteConnection的open方法，jni调用nativeOpen传入path,openFlags信息 在native层去打开数据库返回一个mConnectionPtr指针。用于操作数据库。

   ```java
   //com.android.database.sqlite.SQLiteDatabase
    public static SQLiteDatabase openDatabase(String path, CursorFactory factory, int flags,
               DatabaseErrorHandler errorHandler) {
           SQLiteDatabase db = new SQLiteDatabase(path, flags, factory, errorHandler);
           db.open();
           return db;
       }
   
   //com.android.database.sqlite.SQLiteConnection
   private void open() {
           mConnectionPtr = nativeOpen(mConfiguration.path, mConfiguration.openFlags,
                   mConfiguration.label,
                   SQLiteDebug.DEBUG_SQL_STATEMENTS, SQLiteDebug.DEBUG_SQL_TIME);
   
           setPageSize();
           setForeignKeyModeFromConfiguration();
           setWalModeFromConfiguration();
           setJournalSizeLimit();
           setAutoCheckpointInterval();
           setLocaleFromConfiguration();
   
           // Register custom functions.
           final int functionCount = mConfiguration.customFunctions.size();
           for (int i = 0; i < functionCount; i++) {
               SQLiteCustomFunction function = mConfiguration.customFunctions.get(i);
               nativeRegisterCustomFunction(mConnectionPtr, function);
           }
       }
   ```

4. 分配语句句柄

   在native层分配

   

5. 执行SQL语句

6. 结果集处理

   上面两步就是我们开发时最主要的两个步骤啦，很多安卓数据库框架就是为了优化这部分行为才出来的比如

   greenDao,room等

   先回过头看创建表的过程。

   ```java
   private android.database.sqlite.SQLiteDatabase getDatabaseLocked(boolean writable) {
           if (mDatabase != null) {
               if (!mDatabase.isOpen()) {
                   // Darn!  The user closed the database by calling mDatabase.close().
                   mDatabase = null;
               } else if (!writable || !mDatabase.isReadOnly()) {
                   // The database is already open for business.
                   return mDatabase;
               }
           }
   
           if (mIsInitializing) {
               throw new IllegalStateException("getDatabase called recursively");
           }
   
           android.database.sqlite.SQLiteDatabase db = mDatabase;
           try {
               mIsInitializing = true;
   
               if (db != null) {
                   if (writable && db.isReadOnly()) {
                       db.reopenReadWrite();
                   }
               } else if (mName == null) {
                   db = android.database.sqlite.SQLiteDatabase.create(null);
               } else {
                   try {
                       if (DEBUG_STRICT_READONLY && !writable) {
                           final String path = mContext.getDatabasePath(mName).getPath();
                           db = android.database.sqlite.SQLiteDatabase.openDatabase(path, mFactory,
                                   android.database.sqlite.SQLiteDatabase.OPEN_READONLY, mErrorHandler);
                       } else {
                           db = mContext.openOrCreateDatabase(mName, mEnableWriteAheadLogging ?
                                   Context.MODE_ENABLE_WRITE_AHEAD_LOGGING : 0,
                                   mFactory, mErrorHandler);
                       }
                   } catch (android.database.sqlite.SQLiteException ex) {
                       if (writable) {
                           throw ex;
                       }
                       Log.e(TAG, "Couldn't open " + mName
                               + " for writing (will try read-only):", ex);
                       final String path = mContext.getDatabasePath(mName).getPath();
                       db = android.database.sqlite.SQLiteDatabase.openDatabase(path, mFactory,
                               android.database.sqlite.SQLiteDatabase.OPEN_READONLY, mErrorHandler);
                   }
               }
   
               onConfigure(db);
   
               final int version = db.getVersion();
               if (version != mNewVersion) {
                   if (db.isReadOnly()) {
                       throw new android.database.sqlite.SQLiteException("Can't upgrade read-only database from version " +
                               db.getVersion() + " to " + mNewVersion + ": " + mName);
                   }
   
                   db.beginTransaction();
                   try {
                       if (version == 0) {
                           onCreate(db);
                       } else {
                           if (version > mNewVersion) {
                               onDowngrade(db, version, mNewVersion);
                           } else {
                               onUpgrade(db, version, mNewVersion);
                           }
                       }
                       db.setVersion(mNewVersion);
                       db.setTransactionSuccessful();
                   } finally {
                       db.endTransaction();
                   }
               }
   
               onOpen(db);
   
               if (db.isReadOnly()) {
                   Log.w(TAG, "Opened " + mName + " in read-only mode");
               }
   
               mDatabase = db;
               return db;
           } finally {
               mIsInitializing = false;
               if (db != null && db != mDatabase) {
                   db.close();
               }
           }
       }
   ```

    上面方法功能点

   1. 如果mDataBase已经创建过则不去创建，已经连接则不去连接直接返回。

   2. 以数据库版本为条件。开启事务处理创建表或修改表结构的一些逻辑。这里就知道了version=0时执行我们在onCreate中写的建表语句，在onUpGrade中去做升级操作。然后设置新版本

    看一下java层处理事务的逻辑：

   [SQLiteSession作用](#SQLiteSession)

   [transactionMode作用]()

   创建SQLiteSession 时将我们打开数据库时得到的SQLiteConnectionPool传入.所以所有Session公用一个连接池。

    1. beginTransaction步骤

       1. 通过SQLiteConnectionPool获取连接SQLiteConnection。[acquireConnection](#acquireConnection)
       2. 通过SQLiteConnection执行BEGIN SQL [connection execute](#connection execute)
       3. 获取一个Transaction对象放在mTransactionStack顶部

   	2.  setTransactionSuccessful步骤

       将栈顶的mTransactionStack.mMarkedSuccessful 标记为true.

   	3.  endTransaction（一般放在finally中 让必需执行）

       1. 事务成功条件(top.mMarkedSuccessful || yielding) && !top.mChildFailed;

          top.mMarkedSuccessful本次事务是否成功，yielding调用了yieldTransaction,top.mChildFailed嵌套事务是否成功。

       2. 回收事务

       3. 如果外部还有事务且本次事务不成功mTransactionStack.mChildFailed = true;

       4. s成功执行COMMIT，否则执行ROLLBACK

       5. 释放当前连接[releaseConnection](#releaseConnection)

   ```java
    private void beginTransactionUnchecked(int transactionMode,
               SQLiteTransactionListener transactionListener, int connectionFlags,
               CancellationSignal cancellationSignal) {
           if (cancellationSignal != null) {
               cancellationSignal.throwIfCanceled();
           }
   
           if (mTransactionStack == null) {
               acquireConnection(null, connectionFlags, cancellationSignal); // might throw
           }
           try {
               // Set up the transaction such that we can back out safely
               // in case we fail part way.
               if (mTransactionStack == null) {
                   // Execute SQL might throw a runtime exception.
                   switch (transactionMode) {
                       case TRANSACTION_MODE_IMMEDIATE:
                           mConnection.execute("BEGIN IMMEDIATE;", null,
                                   cancellationSignal); // might throw
                           break;
                       case TRANSACTION_MODE_EXCLUSIVE:
                           mConnection.execute("BEGIN EXCLUSIVE;", null,
                                   cancellationSignal); // might throw
                           break;
                       default:
                           mConnection.execute("BEGIN;", null, cancellationSignal); // might throw
                           break;
                   }
               }
   
               // Listener might throw a runtime exception.
               if (transactionListener != null) {
                   try {
                       transactionListener.onBegin(); // might throw
                   } catch (RuntimeException ex) {
                       if (mTransactionStack == null) {
                           mConnection.execute("ROLLBACK;", null, cancellationSignal); // might throw
                       }
                       throw ex;
                   }
               }
   
               // Bookkeeping can't throw, except an OOM, which is just too bad...
               Transaction transaction = obtainTransaction(transactionMode, transactionListener);
               transaction.mParent = mTransactionStack;
               mTransactionStack = transaction;
           } finally {
               if (mTransactionStack == null) {
                   releaseConnection(); // might throw
               }
           }
       }
   ```

   ##### 创建表步骤

   ######executeSql

   ```java
   private int executeSql(String sql, Object[] bindArgs) throws SQLException {
           acquireReference();
           try {
               if (DatabaseUtils.getSqlStatementType(sql) == DatabaseUtils.STATEMENT_ATTACH) {
                   boolean disableWal = false;
                   synchronized (mLock) {
                       if (!mHasAttachedDbsLocked) {
                           mHasAttachedDbsLocked = true;
                           disableWal = true;
                       }
                   }
                   if (disableWal) {
                       disableWriteAheadLogging();
                   }
               }
   
               SQLiteStatement statement = new SQLiteStatement(this, sql, bindArgs);
               try {
                   return statement.executeUpdateDelete();
               } finally {
                   statement.close();
               }
           } finally {
               releaseReference();
           }
       }
   ```

   1. x[了解acquireReference](#acquireReference)
   2. Attach 的SQL先不理他
   3. 创建一个SQLiteStatement 创建时父类SQLiteProgram构造方法中调用session.prepare预先获取信息 [connection prepare](#connection prepare)，执行statement.executeUpdateDelete() 通过SQLiteSession去执行，
   4. 释放statement。

   ###### insert 

   1. 将contentValues的key 拼接到sql上，values加到bindArgs中
   2. 同上 调用的是nativeExecuteForLastInsertedRowId

   ######update,delete

   1. 同insert ,调用的是nativeExecuteForChangedRowCount

   #### 查询

   ![SQLiteDatabase query](./assets/SQLiteDatabase query.png)

   来到了最查用的查询步骤。[CursorWindow了解](#CursorWindow),[SQLiteCursor了解](#SQLiteCursor)

   最开始的时候觉得查询为什么不能直接返回一个列表，每次都要返回Cursor自己处理这么麻烦(后面很多数据框架帮忙处理了这一点，但是最终还是通过cursor),现在想想还是有原因的，数据库大多是这样与应用程序交互的，底层数据查询应该更开放，这里就是将查询的结果放在CursorWindow上，自己按需查找需要的数据。

   SQLiteDatabase

   ```java
   public Cursor rawQueryWithFactory(
               CursorFactory cursorFactory, String sql, String[] selectionArgs,
               String editTable, CancellationSignal cancellationSignal) {
          ...
       }
   ```

   通过[SQLiteDirectCursorDriver](#SQLiteDirectCursorDriver)去查询

   ```java
   public Cursor query(CursorFactory factory, String[] selectionArgs) {
           final SQLiteQuery query = new SQLiteQuery(mDatabase, mSql, mCancellationSignal);
           final Cursor cursor;
           try {
               query.bindAllArgsAsStrings(selectionArgs);
   
               if (factory == null) {
                   cursor = new SQLiteCursor(this, mEditTable, query);
               } else {
                   cursor = factory.newCursor(mDatabase, this, mEditTable, query);
               }
           } catch (RuntimeException ex) {
               query.close();
               throw ex;
           }
   
           mQuery = query;
           return cursor;
       }
   ```

   [SQLiteQuery功能](#SQLiteQuery)

   1. 构建一个SQLiteQuery准备一个SQL  PreparedStatement。
   2. 构建一个SQLiteCursor返回应用程序调用。
   3. Cursor.moveToFirst 调用SQLiteCursor.fillWindow->SQLiteQuery.fillWindow->SQLiteSession->SQLiteConnection->native将数据存到CursorWindow中mWindowPtr指针指向的地址
   4. SQLiteCursor.moveToNext.移动Cursor mPos到下一行。
   5. SQLiteCursor.getString 通过mPos和CursorWindow对象获取值
   6. SQLiteCursor.close 释放一次引用releaseReference，dispose mWindowPtr(调用完Cursor记得close,否则只有SQLiteCursor对象被回收才会释放)

   

7. 中止处理

---

##### SQLiteSession

 提供一个会话使用数据库
database的访问通常基于session。session用于管理事务的生命周期和数据库连接
session既可以用于只读也可以用于读写操作。对读操作可以做优化，可以并发执行，写操作串行执行。
当开启WAL模式时可以同时执行多个读和一个写事务。关闭WAL时读事务并发，写事务独享
session对象并非线程安全，session有线程边界，SQLiteDatabse用thread-local获取它，因此事务是线程独立的。

一个线程在数据库上最多有一个session，这就限制了一个线程不能同时使用超过一个连接。因为一个线程使用多个连接容易造成死锁 ，所以做了这个限制

指导：

1. 不要在UI线程上提交事务。（会造成无响应)
2. 保证数据库事务尽可能短。（避免获取数据时间长，容易阻塞其他事务执行）
3. 简单的查询比复杂查询更快（废话）
4. 测试数据库事务提交性能
5. 考虑到数据增长带来的影响（一个查询在100条数据没问题，可能10000条数据就慢了）

##### TransactionMode

BEGIN [ DEFERRED | IMMEDIATE | EXCLUSIVE ] TRANSACTION；
一个deferred事务不获取任何锁，直到它需要锁的时候，而且BEGIN语句本身也不会做什么事情——它开始于UNLOCK状态；默认情况下是这样的。如果仅仅用BEGIN开始一个事务，那么事务就是DEFERRED的，同时它不会获取任何锁，当对数据库进行第一次读操作时，它会获取SHARED LOCK；同样，当进行第一次写操作时，它会获取RESERVED LOCK。
由BEGIN开始的Immediate事务会试着获取RESERVED LOCK。如果成功，BEGIN IMMEDIATE保证没有别的连接可以写数据库。但是，别的连接可以对数据库进行读操作，但是RESERVED LOCK会阻止其它的连接BEGIN IMMEDIATE或者BEGIN EXCLUSIVE命令，SQLite会返回SQLITE_BUSY错误。这时你就可以对数据库进行修改操作，但是你不能提交，当你COMMIT时，会返回SQLITE_BUSY错误，这意味着还有其它的读事务没有完成，得等它们执行完后才能提交事务。
Exclusive事务会试着获取对数据库的EXCLUSIVE锁。这与IMMEDIATE类似，但是一旦成功，EXCLUSIVE事务保证没有其它的连接，所以就可对数据库进行读写操作了。

##### acquireConnection

```java
 private void acquireConnection(String sql, int connectionFlags,
            CancellationSignal cancellationSignal) {
        if (mConnection == null) {
            assert mConnectionUseCount == 0;
            mConnection = mConnectionPool.acquireConnection(sql, connectionFlags,
                    cancellationSignal); // might throw
            mConnectionFlags = connectionFlags;
        }
        mConnectionUseCount += 1;
    }
private void releaseConnection() {
        assert mConnection != null;
        assert mConnectionUseCount > 0;
        if (--mConnectionUseCount == 0) {
            try {
                mConnectionPool.releaseConnection(mConnection); // might throw
            } finally {
                mConnection = null;
            }
        }
    }
```

Session中在mConnectionPool中请求连接，这里mConnectionUseCount用于计数对应releaseConnection，比如在事务中执行一些语句就不必要重新获取连接。
connectionFlags:

* CONNECTION_FLAG_READ_ONLY用于读， 
*  CONNECTION_FLAG_PRIMARY_CONNECTION_AFFINITY ：主连接，串行操作，用于写会对db加锁，直到锁释放后下个请求才能获取到主连接。
*  CONNECTION_FLAG_INTERACTIVE：用于与UI线程交互，连接池会提高这个请求的优先级

```java
private SQLiteConnection waitForConnection(String sql, int connectionFlags,
            CancellationSignal cancellationSignal) {
        final boolean wantPrimaryConnection =
                (connectionFlags & CONNECTION_FLAG_PRIMARY_CONNECTION_AFFINITY) != 0;

        final ConnectionWaiter waiter;
        final int nonce;
        synchronized (mLock) {
            throwIfClosedLocked();

            // Abort if canceled.
            if (cancellationSignal != null) {
                cancellationSignal.throwIfCanceled();
            }

            // Try to acquire a connection.
            SQLiteConnection connection = null;
            if (!wantPrimaryConnection) {
                connection = tryAcquireNonPrimaryConnectionLocked(
                        sql, connectionFlags); // might throw
            }
            if (connection == null) {
                connection = tryAcquirePrimaryConnectionLocked(connectionFlags); // might throw
            }
            if (connection != null) {
                return connection;
            }

            // No connections available.  Enqueue a waiter in priority order.
            final int priority = getPriority(connectionFlags);
            final long startTime = SystemClock.uptimeMillis();
            waiter = obtainConnectionWaiterLocked(Thread.currentThread(), startTime,
                    priority, wantPrimaryConnection, sql, connectionFlags);
            ConnectionWaiter predecessor = null;
            ConnectionWaiter successor = mConnectionWaiterQueue;
            while (successor != null) {
                if (priority > successor.mPriority) {
                    waiter.mNext = successor;
                    break;
                }
                predecessor = successor;
                successor = successor.mNext;
            }
            if (predecessor != null) {
                predecessor.mNext = waiter;
            } else {
                mConnectionWaiterQueue = waiter;
            }

            nonce = waiter.mNonce;
        }

        // Set up the cancellation listener.
        if (cancellationSignal != null) {
            cancellationSignal.setOnCancelListener(new CancellationSignal.OnCancelListener() {
                @Override
                public void onCancel() {
                    synchronized (mLock) {
                        if (waiter.mNonce == nonce) {
                            cancelConnectionWaiterLocked(waiter);
                        }
                    }
                }
            });
        }
        try {
            // Park the thread until a connection is assigned or the pool is closed.
            // Rethrow an exception from the wait, if we got one.
            long busyTimeoutMillis = CONNECTION_POOL_BUSY_MILLIS;
            long nextBusyTimeoutTime = waiter.mStartTime + busyTimeoutMillis;
            for (;;) {
                // Detect and recover from connection leaks.
                if (mConnectionLeaked.compareAndSet(true, false)) {
                    synchronized (mLock) {
                        wakeConnectionWaitersLocked();
                    }
                }

                // Wait to be unparked (may already have happened), a timeout, or interruption.
                LockSupport.parkNanos(this, busyTimeoutMillis * 1000000L);

                // Clear the interrupted flag, just in case.
                Thread.interrupted();

                // Check whether we are done waiting yet.
                synchronized (mLock) {
                    throwIfClosedLocked();

                    final SQLiteConnection connection = waiter.mAssignedConnection;
                    final RuntimeException ex = waiter.mException;
                    if (connection != null || ex != null) {
                        recycleConnectionWaiterLocked(waiter);
                        if (connection != null) {
                            return connection;
                        }
                        throw ex; // rethrow!
                    }

                    final long now = SystemClock.uptimeMillis();
                    if (now < nextBusyTimeoutTime) {
                        busyTimeoutMillis = now - nextBusyTimeoutTime;
                    } else {
                        logConnectionPoolBusyLocked(now - waiter.mStartTime, connectionFlags);
                        busyTimeoutMillis = CONNECTION_POOL_BUSY_MILLIS;
                        nextBusyTimeoutTime = now + busyTimeoutMillis;
                    }
                }
            }
        } finally {
            // Remove the cancellation listener.
            if (cancellationSignal != null) {
                cancellationSignal.setOnCancelListener(null);
            }
        }
```

简单描述上面方法功能

1. 根据connectionFlags判断是否获取主连接
2. 不是主连接，获取非主连接tryAcquireNonPrimaryConnectionLocked。
3. 是主连接或者没获取到非主连接tryAcquirePrimaryConnectionLocked。成功获取直接返回。
4. 获取优先级得到一个ConnectionWaiter对象，obtainConnectionWaiterLocked
5. 将waiter更具优先级插入mConnectionWaiterQueue中
6. 线程park,等待其他连接释放 unpark mConnectionWaiterQueue中waiter的thread.

```java
private SQLiteConnection tryAcquireNonPrimaryConnectionLocked(
            String sql, int connectionFlags) {
        // Try to acquire the next connection in the queue.
        SQLiteConnection connection;
        final int availableCount = mAvailableNonPrimaryConnections.size();
        if (availableCount > 1 && sql != null) {
            // If we have a choice, then prefer a connection that has the
            // prepared statement in its cache.
            for (int i = 0; i < availableCount; i++) {
                connection = mAvailableNonPrimaryConnections.get(i);
                if (connection.isPreparedStatementInCache(sql)) {
                    mAvailableNonPrimaryConnections.remove(i);
                    finishAcquireConnectionLocked(connection, connectionFlags); // might throw
                    return connection;
                }
            }
        }
        if (availableCount > 0) {
            // Otherwise, just grab the next one.
            connection = mAvailableNonPrimaryConnections.remove(availableCount - 1);
            finishAcquireConnectionLocked(connection, connectionFlags); // might throw
            return connection;
        }

        // Expand the pool if needed.
        int openConnections = mAcquiredConnections.size();
        if (mAvailablePrimaryConnection != null) {
            openConnections += 1;
        }
        if (openConnections >= mMaxConnectionPoolSize) {
            return null;
        }
        connection = openConnectionLocked(mConfiguration,
                false /*primaryConnection*/); // might throw
        finishAcquireConnectionLocked(connection, connectionFlags); // might throw
        return connection;
    }
```

获取非主连接步骤

1. 先从mAvailableNonPrimaryConnections找到可用连接。如果有可用且有statement缓存直接返回。

    mAvailableNonPrimaryConnections.remove(i)，finishAcquireConnectionLocked(connection, connectionFlags);就是用于将连接池持有连接 转入被SQLiteSession所请求的连接。

2. 不满足可用statement缓存，返回最后一个可用连接。

3. 获取已被请求连接数 大于最大可连接返回null 否则创建一个新的连接(上面创建连接部分)

```java
// Might throw.
    private SQLiteConnection tryAcquirePrimaryConnectionLocked(int connectionFlags) {
        // If the primary connection is available, acquire it now.
        SQLiteConnection connection = mAvailablePrimaryConnection;
        if (connection != null) {
            mAvailablePrimaryConnection = null;
            finishAcquireConnectionLocked(connection, connectionFlags); // might throw
            return connection;
        }

        // Make sure that the primary connection actually exists and has just been acquired.
        for (SQLiteConnection acquiredConnection : mAcquiredConnections.keySet()) {
            if (acquiredConnection.isPrimaryConnection()) {
                return null;
            }
        }

        // Uhoh.  No primary connection!  Either this is the first time we asked
        // for it, or maybe it leaked?
        connection = openConnectionLocked(mConfiguration,
                true /*primaryConnection*/); // might throw
        finishAcquireConnectionLocked(connection, connectionFlags); // might throw
        return connection;
    }
```

1. 主连接是否存在且可用 则返回给session.(mAvailablePrimaryConnection存在就表示可用主连接)
2. 主连接不可用且被其他session获取返回null。
3. 主连接未打开 去打开返回。

通过上述步骤我们获取到了一个SQLiteConnection。

##### SQLiteConnectionPool

维持一个可用的数据库连接池

任何时候，任何一个连接connection要么被pool所持有要么被SQLiteSession所请求，当session中结束连接时，必需归还给连接池

连接池用强引用持有自己的连接mAvailableNonPrimaryConnections,mAvailablePrimaryConnection，弱引用持有sessions的连接mAcquiredConnections。是因为能在之后能够检查到那些不舍当被抛弃的连接，可以创建新连接去替换。

连接池是线程安全的，但是连接本身不是。

##### SQLiteConnection

代表一个数据库连接，任何一个连接包含一个native sqlite3实例

当数据库连接池打开时允许多个连接到一个数据库，否则通常一个连接一个数据库。

当SQLITE的WAL功能被开启，支持多个读和一个写并发执行。否则读和写都是独享的。

连接对象本身不是线程安全的。需要的时候被请求不需要返回个连接池。用session和pool老保证其线程安全

连接对象持有全部sqlite连接相关的native对象，其他对象没有持有这些native对象，所以当连接关闭时我们只要关注连接对象。

##### connection execute

```java
 public void execute(String sql, Object[] bindArgs,
            CancellationSignal cancellationSignal) {
        if (sql == null) {
            throw new IllegalArgumentException("sql must not be null.");
        }

        final int cookie = mRecentOperations.beginOperation("execute", sql, bindArgs);
        try {
            final PreparedStatement statement = acquirePreparedStatement(sql);
            try {
                throwIfStatementForbidden(statement);
                bindArguments(statement, bindArgs);
                applyBlockGuardPolicy(statement);
                attachCancellationSignal(cancellationSignal);
                try {
                    nativeExecute(mConnectionPtr, statement.mStatementPtr);
                } finally {
                    detachCancellationSignal(cancellationSignal);
                }
            } finally {
                releasePreparedStatement(statement);
            }
        } catch (RuntimeException ex) {
            mRecentOperations.failOperation(cookie, ex);
            throw ex;
        } finally {
            mRecentOperations.endOperation(cookie);
        }
    }
```

1. mRecentOperations用于记录最近的操作完成情况具体有哪些信息可以看OperationLog的dump函数
2. 请求一个PreparedStatement
3. 绑定bindArgs到statement
4.  nativeExecute(mConnectionPtr, statement.mStatementPtr); mConnectionPtr是打开连接时native 对象的指针，mStatementPtr时创建PreparedStatement native statement对象指针。这里就去native中去执行SQL逻辑了。
5. 释放PreparedStatement 

##### acquirePreparedStatement

```java
private PreparedStatement acquirePreparedStatement(String sql) {
        PreparedStatement statement = mPreparedStatementCache.get(sql);
        boolean skipCache = false;
        if (statement != null) {
            if (!statement.mInUse) {
                return statement;
            }
            // The statement is already in the cache but is in use (this statement appears
            // to be not only re-entrant but recursive!).  So prepare a new copy of the
            // statement but do not cache it.
            skipCache = true;
        }

        final long statementPtr = nativePrepareStatement(mConnectionPtr, sql);
        try {
            final int numParameters = nativeGetParameterCount(mConnectionPtr, statementPtr);
            final int type = DatabaseUtils.getSqlStatementType(sql);
            final boolean readOnly = nativeIsReadOnly(mConnectionPtr, statementPtr);
            statement = obtainPreparedStatement(sql, statementPtr, numParameters, type, readOnly);
            if (!skipCache && isCacheable(type)) {
                mPreparedStatementCache.put(sql, statement);
                statement.mInCache = true;
            }
        } catch (RuntimeException ex) {
            // Finalize the statement if an exception occurred and we did not add
            // it to the cache.  If it is already in the cache, then leave it there.
            if (statement == null || !statement.mInCache) {
                nativeFinalizeStatement(mConnectionPtr, statementPtr);
            }
            throw ex;
        }
        statement.mInUse = true;
        return statement;
    }
```

1. 在mPreparedStatementCache(LruCache)中根据sql找到是否有可用的statement，可用直接返回。
2. 通过mConnectionPtr，和sql创建一个native对象statementPtr
3.  obtainPreparedStatement 获取一个statement对象，mPreparedStatementCache中不存在则存进去
4. 标记正在使用中，所以我们执行sql时带bindArgs就可以多命中缓存啦和Retrofit中缓存有点类似。

```java
private void bindArguments(PreparedStatement statement, Object[] bindArgs) {
    ...
}
        
```

主要功能通过statementPtr将参数绑定到native 的statement中,避免每次都创建statement吧。

```java
private void releasePreparedStatement(PreparedStatement statement) {
        statement.mInUse = false;
        if (statement.mInCache) {
            try {
                nativeResetStatementAndClearBindings(mConnectionPtr, statement.mStatementPtr);
            } catch (android.database.sqlite.SQLiteException ex) {
                // The statement could not be reset due to an error.  Remove it from the cache.
                // When remove() is called, the cache will invoke its entryRemoved() callback,
                // which will in turn call finalizePreparedStatement() to finalize and
                // recycle the statement.
                if (DEBUG) {
                    Log.d(TAG, "Could not reset prepared statement due to an exception.  "
                            + "Removing it from the cache.  SQL: "
                            + trimSqlForDisplay(statement.mSql), ex);
                }

                mPreparedStatementCache.remove(statement.mSql);
            }
        } else {
            finalizePreparedStatement(statement);
        }
    }
```

1. 标记未使用。
2. 在缓存中则 nativeResetStatementAndClearBindings否则finalizePreparedStatement



##### releaseConnection

```java
public void releaseConnection(SQLiteConnection connection) {
        synchronized (mLock) {
            AcquiredConnectionStatus status = mAcquiredConnections.remove(connection);
            if (status == null) {
                throw new IllegalStateException("Cannot perform this operation "
                        + "because the specified connection was not acquired "
                        + "from this pool or has already been released.");
            }

            if (!mIsOpen) {
                closeConnectionAndLogExceptionsLocked(connection);
            } else if (connection.isPrimaryConnection()) {
                if (recycleConnectionLocked(connection, status)) {
                    assert mAvailablePrimaryConnection == null;
                    mAvailablePrimaryConnection = connection;
                }
                wakeConnectionWaitersLocked();
            } else if (mAvailableNonPrimaryConnections.size() >= mMaxConnectionPoolSize - 1) {
                closeConnectionAndLogExceptionsLocked(connection);
            } else {
                if (recycleConnectionLocked(connection, status)) {
                    mAvailableNonPrimaryConnections.add(connection);
                }
                wakeConnectionWaitersLocked();
            }
        }
    }
```

1. 先从mAcquiredConnections中remove.
2. 连接池被关了，直接connection.close()
3. 是主连接 recycleConnectionLocked 满足回收到mAvailablePrimaryConnection，调用wakeConnectionWaitersLocked通知其他等待请求的session有连接释放了。
4. 超过连接池大小 直接connection.close()
5. 和3步骤一致，只是回收到mAvailableNonPrimaryConnections中。

```java
private boolean recycleConnectionLocked(SQLiteConnection connection,
            AcquiredConnectionStatus status) {
        if (status == AcquiredConnectionStatus.RECONFIGURE) {
            try {
                connection.reconfigure(mConfiguration); // might throw
            } catch (RuntimeException ex) {
                Log.e(TAG, "Failed to reconfigure released connection, closing it: "
                        + connection, ex);
                status = AcquiredConnectionStatus.DISCARD;
            }
        }
        if (status == AcquiredConnectionStatus.DISCARD) {
            closeConnectionAndLogExceptionsLocked(connection);
            return false;
        }
        return true;
    }
```

1. 判断连接可回收状态，RECONFIGURE，DISCARD都不可回收
2.  RECONFIGURE ，connection.reconfigure(mConfiguration)
3.  DISCARD ，直接close

```java
// Can't throw.
    private void wakeConnectionWaitersLocked() {
        // Unpark all waiters that have requests that we can fulfill.
        // This method is designed to not throw runtime exceptions, although we might send
        // a waiter an exception for it to rethrow.
        ConnectionWaiter predecessor = null;
        ConnectionWaiter waiter = mConnectionWaiterQueue;
        boolean primaryConnectionNotAvailable = false;
        boolean nonPrimaryConnectionNotAvailable = false;
        while (waiter != null) {
            boolean unpark = false;
            if (!mIsOpen) {
                unpark = true;
            } else {
                try {
                    SQLiteConnection connection = null;
                    if (!waiter.mWantPrimaryConnection && !nonPrimaryConnectionNotAvailable) {
                        connection = tryAcquireNonPrimaryConnectionLocked(
                                waiter.mSql, waiter.mConnectionFlags); // might throw
                        if (connection == null) {
                            nonPrimaryConnectionNotAvailable = true;
                        }
                    }
                    if (connection == null && !primaryConnectionNotAvailable) {
                        connection = tryAcquirePrimaryConnectionLocked(
                                waiter.mConnectionFlags); // might throw
                        if (connection == null) {
                            primaryConnectionNotAvailable = true;
                        }
                    }
                    if (connection != null) {
                        waiter.mAssignedConnection = connection;
                        unpark = true;
                    } else if (nonPrimaryConnectionNotAvailable && primaryConnectionNotAvailable) {
                        // There are no connections available and the pool is still open.
                        // We cannot fulfill any more connection requests, so stop here.
                        break;
                    }
                } catch (RuntimeException ex) {
                    // Let the waiter handle the exception from acquiring a connection.
                    waiter.mException = ex;
                    unpark = true;
                }
            }

            final ConnectionWaiter successor = waiter.mNext;
            if (unpark) {
                if (predecessor != null) {
                    predecessor.mNext = successor;
                } else {
                    mConnectionWaiterQueue = successor;
                }
                waiter.mNext = null;

                LockSupport.unpark(waiter.mThread);
            } else {
                predecessor = waiter;
            }
            waiter = successor;
        }
    }
```

1. 尝试去获取连接。根据mConnectionWaiterQueue第一个waiter的mWantPrimaryConnection去获取连接
2. 获取不到直接返回，获取到了奖connection赋值给waiter.mAssignedConnection unpark = true;
3. 调整mConnectionWaiterQueue unpark waiter.thread。然后被阻的[acquireConnection](#acquireConnection)开始执行

---

##### acquireReference

```java
public void acquireReference() {
        synchronized(this) {
            if (mReferenceCount <= 0) {
                throw new IllegalStateException(
                        "attempt to re-open an already-closed object: " + this);
            }
            mReferenceCount++;
        }
    }

 public void releaseReference() {
        boolean refCountIsZero = false;
        synchronized(this) {
            refCountIsZero = --mReferenceCount == 0;
        }
        if (refCountIsZero) {
            onAllReferencesReleased();
        }
    }
  public void close() {
        releaseReference();
    }
```

看一下为什么要有这个方法。刚创建database这个mReferenceCount=1。每次执行SQL或者开启关闭事务都会调用一下acquireReference mReferenceCount++。同时执行完后会调用releaseReference，--mReferenceCount 主动调用close也是调用releaseReference方法。当mReferenceCount=0 才会去调用dispose方法。

那么这个方法就是用来计数用的。就是说我们调用close后，如果还有任务没执行完，就不会dispose等所以任务执行完后才会dispose.

dispose方法会去关闭连接池里的所有native连接，以及清除缓存里的连接

##### SQLiteStatement

表示一个可以对数据执行的声明。不过这个执行只能返回一些简单的结果。这个声明去创建时会预先从数据库获取一些信息。比如mReadOnly，mColumnNames，mNumParameters。

##### connection prepare

```java
public void prepare(String sql, SQLiteStatementInfo outStatementInfo) {
        if (sql == null) {
            throw new IllegalArgumentException("sql must not be null.");
        }

        final int cookie = mRecentOperations.beginOperation("prepare", sql, null);
        try {
            final PreparedStatement statement = acquirePreparedStatement(sql);
            try {
                if (outStatementInfo != null) {
                    outStatementInfo.numParameters = statement.mNumParameters;
                    outStatementInfo.readOnly = statement.mReadOnly;

                    final int columnCount = nativeGetColumnCount(
                            mConnectionPtr, statement.mStatementPtr);
                    if (columnCount == 0) {
                        outStatementInfo.columnNames = EMPTY_STRING_ARRAY;
                    } else {
                        outStatementInfo.columnNames = new String[columnCount];
                        for (int i = 0; i < columnCount; i++) {
                            outStatementInfo.columnNames[i] = nativeGetColumnName(
                                    mConnectionPtr, statement.mStatementPtr, i);
                        }
                    }
                }
            } finally {
                releasePreparedStatement(statement);
            }
        } catch (RuntimeException ex) {
            mRecentOperations.failOperation(cookie, ex);
            throw ex;
        } finally {
            mRecentOperations.endOperation(cookie);
        }
    }
```

准备一个statement用于执行但是 不绑定任何参数及执行它

这个方法用于在编译成执行statement之前检测语法错误。将相关信息保存到outStatementInfo上

一份预先准备好的statement中没有绑定最终的参数，所有他可以缓存某些SELECT,UPDATE,INSERT语句可缓存起来提供后面使用。

nativePrepareStatement会跳用到native sqlite3.c的sqlite3_prepare方法，这个函数将sql文本转换成一个准备语句（prepared statement）对象，同时返回这个对象的指针。这个接口需要一个数据库连接指针以及一个要准备的包含SQL语句的文本。它实际上并不执行这个SQL语句，它仅仅为执行准备这个sql语句。



---

##### CursorWindow

一个包含多个cursor 行数据的缓存区(窗口)

通过这个类与native的数据缓存区交互。

```java
public CursorWindow(String name) {
        mStartPos = 0;
        mName = name != null && name.length() != 0 ? name : "<unnamed>";
        if (sCursorWindowSize < 0) {
            /** The cursor window size. resource xml file specifies the value in kB.
             * convert it to bytes here by multiplying with 1024.
             */
            sCursorWindowSize = Resources.getSystem().getInteger(
                com.android.internal.R.integer.config_cursorWindowSize) * 1024;
        }
        mWindowPtr = nativeCreate(mName, sCursorWindowSize);
        if (mWindowPtr == 0) {
            throw new CursorWindowAllocationException("Cursor window allocation of " +
                    (sCursorWindowSize / 1024) + " kb failed. " + printStats());
        }
        mCloseGuard.open("close");
        recordNewWindow(Binder.getCallingPid(), mWindowPtr);
    }
```

通过数据库名和sCursorWindowSize(窗口大小，默认值)创建CursorWindow 返回native指针地址mWindowPtr

后面一系列查询都通过这个mWindowPtr

##### SQLiteCursor

 AbstractWindowedCursor的子类  AbstractWindowedCursor是Cursors与CursorWindow交互基类，提供一系列通用的方法,比我们常用的getString，getInt方法传入当前位置调用window的native方法获取相应数据，AbstractCursor提供游标处理的通用方法，比如我们常用的moveToFirst，moveToNext最终通过moveToPosition方法记录当前位置，调用字类的onMove判断是否需要重查数据

##### SQLiteQuery
和SQLiteStatement一样是SQLiteProgram的子类，也就是也有预先处理啊SQL语句，绑定参数的功能,SQLiteCursor的fillWindow通过SQLiteQuery的fillWindow与SQLiteSession交互填充数据到CursorWindow。

#####SQLiteDirectCursorDriver
SQLiteCursors的一个驱动用于创建SQLiteCursors，以及会调处理SQLiteCursors的一些重要要生命周期事件。

---

总结，简单分析了android java 层sqlite源码，为后续分析room,greenDao做准备。尝试了解sqlite3 native层