# Feature Spec Template

Developer: triump2020

Date: 03/01/2024

Release: 1.2.0

Issue: https://github.com/matrixorigin/matrixone/issues/10397

## 1. Introduction for snapshot read

快照是备份数据的一种有效方式, 用户可以通过命令为cluster/account/database/table 创建snapshot.

当用户创建完snapshot之后，事务可以读取用户指定时间点或者快照ID的数据, 即read at snapshot,主要用于数据恢复/回滚场景。

### 1.1 Goals

CN侧事务支持 read at snapshot , 即：事务可以读取用户指定时间戳/快照ID 的数据.

### 1.2 Non Goals

snapshot的创建，查询，删除。

## 2. Design

### 2.1 需要解决的问题

- 涉及到快照读语句的语法
  
    1.用户需要读取某个表的快照/历史数据, 读取表t的快照ID为xxxx 的数据. 

   ```sql
   show snapshots;
   
	select * from t@snapshot1; //if schema changed between snapshot1 and now?
	
   select * from (select * from t@snapshot1)
   
   select * from t1@snapshot1 join t2@snapshot2 on t1.x = t2.y 
   
   //join t1's latest snapshot data with t2's snapshot data at snapshot2.
   insert into t3 select * from t1 join t2@snapshot2 on t1.x = t2.y  
   
   
   
   ```

   2. **读取两个snapshot 之间的 diff**

      ```sql
      select * from t@snapshot1...snapshot2;// read diff between snapshot1 and snapshot2
      ```

   3. 从快照中导入全量/增量数据

      ```sql
       
       begin;
       create table t2;
       insert into t2 select * from t1@snapshot1;// full backup
         
       insert into t2 select * from t1@snapshot1...snapshot2 // incremental backup.
       commit;
      ```

      

- Ast 

​       Ast 需要支持snapshot read 的 相关语法.

- Plan 

- 分布式 Scope 

    将plan 转换为可执行的分布式scope 时，需要处理table scan 节点.

- disttae 从TN 去 snapshot

  1. 快照读事务可能需要从TN 中获取较旧的ckp, 而当前CN 状态机(PartitionState与Catalog)中的数据是基于一个最新的ckp + logtail，所以需要
  从TN 中拉取 旧的ckp, 可以通过RPC 方式，与TN 新建一条tcp链接去获取某个表的指定时间戳的ckp(是否新建链接还是复用已有的push 通道待定)?
  <br>

  2. 快照读事务需要一个独立的状态机，将旧的ckp 数据apply 到此状态机中. 当事务提交后，释放状态机.
  <br>

  3. 如果请求的数据版本在当前CN 状态机中已经存在，可以复用已有状态机(第一版本，可以暂时不复用，减少复杂度)



### 2.2 详细设计

#### 2.2.1 Disttae 设计

​       **default engine** : disttae 中目前已存在的global engine, 其生命周期是全局的.

​       **snapshot engine**:  一个txn 在涉及到snapshot read 时，会创建对应snapshot engine, 这个engine 的生命周期与事务一致.

​       一个txn, 内部会有多个snapshot , 对于每一个snapshot 会创建一个只读的snapshot engine,  其实default engine  本质上也是一个 snapshot engine , 只不过这个engine 服务于latest snapshot , 且是可写的.  

​         front 只要遇到 select/join...... from table@snpashot 子句时，就需要 创建对应的snapshot engine ,  将这个snapshot engine 注册到txn 中,  当txn commit/rollback 时，disttae 会自动销毁这个snapshot engine.

​        front 在 bind 时，需要从 snapshot engine 中open snapshot database ,  然后从snapshot database 中open 一个 snapshot table.

​       compute layer 在 compile  table scan node 时，需要调用 snapshot table 的 ranges 接口.

##### 2.2.1.1 snap engine/db/table 设计
​    每个snapshot/时间戳  对应一个snapshot engine .  **这个engine 是只读的**，会实现  disttae/engine/types.go   Engine interface 里的如下只读接口,  其他接口实现留空.

```go
type SnapEngine struct {
	sync.RWMutex
	mp         *mpool.MPool
	fs         fileservice.FileService
	ls         lockservice.LockService
	qs         queryservice.QueryService
	hakeeper   logservice.CNHAKeeperClient
	us         udf.Service
	cli        client.TxnClient
	idGen      IDGenerator
	catalog    *cache.CatalogCache // ==元数据的快照==
	tnID       string
	partitions map[[2]uint64]*logtailreplay.Partition // ==table data 的快照==
	packerPool *fileservice.Pool[*types.Packer]

	// XXX related to cn push model
	pClient pushClien
}

type Engine interface {
    
    // return all database names
    Databases(ctx context.Context, op client.TxnOperator) (databaseNames []string, err error)

	// Database open a handle for a database
	Database(ctx context.Context, databaseName string, op client.TxnOperator) (Database, error)

	// Nodes returns all nodes for worker jobs. isInternal, tenant, cnLabel are
	// used to filter CN servers.
	Nodes(isInternal bool, tenant string, username string, cnLabel map[string]string) (cnNodes Nodes, err error)

	// Hints returns hints of engine features
	// return value should not be cached
	// since implementations may update hints after engine had initialized
	Hints() Hints

	NewBlockReader(ctx context.Context, num int, ts timestamp.Timestamp, expr *plan.Expr, ranges []byte, tblDef *plan.TableDef, proc any) ([]Reader, error)

	// Get database name & table name by table id
	GetNameById(ctx context.Context, op client.TxnOperator, tableId uint64) (dbName string, tblName string, err error)

	// Get relation by table id
	GetRelationById(ctx context.Context, op client.TxnOperator, tableId uint64) (dbName string, tblName string, rel Relation, err error)

	// AllocateIDByKey allocate a globally unique ID by key.
	AllocateIDByKey(ctx context.Context, key string) (uint64, error)

	// Stats returns the stats info of the key.
	// If sync is true, wait for the stats info to be updated, else,
	// just return nil if the current stats info has not been initialized.
	Stats(ctx context.Context, key pb.StatsInfoKey, sync bool) *pb.StatsInfo
    
}

```

 从snap engine 中 open 一个 snapshot database 之后，就可以调用Database 相关的接口：

```go
type snapDB struct {
	databaseId        uint64
	databaseName      string
	databaseType      string
    snapshot          TS
	databaseCreateSql string
	txn               *Transaction
}

type Database interface {
	Relations(context.Context) ([]string, error)
	Relation(context.Context, string, any) (Relation, error)
	GetDatabaseId(context.Context) string
	IsSubscription(context.Context) bool
	GetCreateSql(context.Context) string
}

```

 从snapshot db 中 可以open 一个 snapshot table, 之后可以调用Relation 相关的接口:

```go
type Statistics interface {
	Stats(ctx context.Context, sync bool) *pb.StatsInfo
	Rows(ctx context.Context) (uint64, error)
	Size(ctx context.Context, columnName string) (uint64, error)
}


type Relation interface {
	Statistics
  
	UpdateObjectInfos(context.Context) error //

	Ranges(context.Context, []*plan.Expr) (Ranges, error)

	TableDefs(context.Context) ([]TableDef, error)

	// Get complete tableDef information, including columns, constraints, partitions, version, comments, etc
	GetTableDef(context.Context) *plan.TableDef
	CopyTableDef(context.Context) *plan.TableDef

	GetPrimaryKeys(context.Context) ([]*Attribute, error)

	GetHideKeys(context.Context) ([]*Attribute, error)

	GetTableID(context.Context) uint64

	// GetTableName returns the name of the table.
	GetTableName() string

	GetDBID(context.Context) uint64

	// second argument is the number of reader, third argument is the filter extend, foruth parameter is the payload required by the engine
	NewReader(context.Context, int, *plan.Expr, []byte, bool) ([]Reader, error)

	TableColumns(ctx context.Context) ([]*Attribute, error)

	//max and min values
	MaxAndMinValues(ctx context.Context) ([][2]any, []uint8, error)

	GetEngineType() EngineType

	GetColumMetadataScanInfo(ctx context.Context, name string) ([]*plan.MetadataScanInfo, error)

	// PrimaryKeysMayBeModified reports whether any rows with any primary keys in keyVector was modified during `from` to `to`
	// If not sure, returns true
	// Initially added for implementing locking rows by primary keys
	PrimaryKeysMayBeModified(ctx context.Context, from types.TS, to types.TS, keyVector *vector.Vector) (bool, error)
}

```



##### 2.2.2.2 reader  相关修改

   Relation.NewReader ，partition reader  , block merge reader 需要修改.

##### 2.2.2.3 distae 从TN获取CKP

  snapshot read txn 可能既会访问全局Engine中的快照数据，全局Engine中的数据是基于最新的ckp；
  也会访问 很旧的ckp 数据，所以需要从TN 拉取老的ckp 数据，基于这个ckp apply 到事务自己独立的Engine 中.

  在函数UpdateObjectInfos 中 判断事务是否为snapshot-read 且 snapshot ts < 全局engine中的 的checkpoint 的时间戳.
  则从TN 拉取ckp 并回放到 snapshot engine 中.

 ```go
 func (tbl *txnTable) UpdateObjectInfos(ctx context.Context) (err error) {
	tbl.tnList = []int{0}

	accountId, err := defines.GetAccountId(ctx)
	if err != nil {
		return err
	}
	_, created := tbl.db.txn.createMap.Load(genTableKey(accountId, tbl.tableName, tbl.db.databaseId))
	// check if the table is not created in this txn, and the block infos are not updated, then update:
	// 1. update logtail
	// 2. generate block infos
	// 3. update the blockInfosUpdated and blockInfos fields of the table
	if !created && !tbl.objInfosUpdated {
		if err = tbl.updateLogtail(ctx); err != nil {
			return
		}
		tbl.objInfosUpdated = true
	}
	return
}
 
 ```

##### 2.2.2.4 disttae 与TN 接口

disttae 需要从TN 拉取 旧的ckp 数据及logtail , 建议用pull 方式, 可以复用以下code:


```go
func updatePartitionOfPull(
	primarySeqnum int,
	tbl *txnTable,
	ctx context.Context,
	op client.TxnOperator,
	engine *Engine,
	partition *logtailreplay.Partition,
	tn DNStore,
	req api.SyncLogTailReq,
) error {
	logDebugf(op.Txn(), "updatePartitionOfPull")
	reqs, err := genLogTailReq(tn, req)
	if err != nil {
		return err
	}
	logTails, err := getLogTail(ctx, op, reqs)
	if err != nil {
		return err
	}

	state, doneMutate := partition.MutateState()

	for i := range logTails {
		if err := consumeLogTailOfPull(primarySeqnum, tbl, ctx, engine, state, logTails[i]); err != nil {
			logutil.Errorf("consume %d-%s logtail error: %v\n", tbl.tableId, tbl.tableName, err)
			return err
		}
	}

	doneMutate()

	return nil
}

```



#### 2.2.2  Parser 设计

#### 2.2.2.1 ast tree 设计

```go
   type TableName struct {
	  TableExpr
	  objName
	  snapshotFlag *SnapShotFlag
   }

   type SnapShotFlag struct {
	   HasSnapShot bool
	   SnapShotTs  string
   }
```

```go
    snapshot_flag_opt:
    {
        $$ = $1,
    }
|   '@' STRING
    {
        $$ = &tree.SnapShotFlag{
            HasSnapShot: true,
            SnapShotTs: $2,
        }
    }

	table_name:
    ident snapshot_flag_opt
    {
        prefix := tree.ObjectNamePrefix{ExplicitSchema: false}
        $$ = tree.NewTableName(tree.Identifier($1.Compare()), prefix, $2)
    }
|   ident '.' ident snapshot_flag_opt
    {
        prefix := tree.ObjectNamePrefix{SchemaName: tree.Identifier($1.Compare()), ExplicitSchema: true}
        $$ = tree.NewTableName(tree.Identifier($3.Compare()), prefix, $2)
    }
```

#### 2.2.3  Front 设计
    // @mochen 是否需要修改内部执行sql？


#### 2.2.4 Plan 设计
    // 主要是修改
	buildFrom:
	   buildTable:
	       *tree.TableName
		   //在Node中增加snapshotTs信息



























 

