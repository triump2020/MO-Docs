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
  
1. 用户需要读取某个表的快照/历史数据, 读取表t的快照ID为xxxx 的数据. 

   ```sql
   show snapshots;
   
	select * from t@snapshot1; //if schema changed between snapshot1 and now?
	
   select * from (select * from t@snapshot1) // snapshot1 是类型为types.TS 的时间戳
                                             // 同时支持snapshot name/id
   
   select * from t1@snapshot1 join t2@snapshot2 on t1.x = t2.y 
   
   //join t1's latest snapshot data with t2's snapshot data at snapshot2.
   insert into t3 select * from t1 join t2@snapshot2 on t1.x = t2.y  
   
   ```
   
2. **读取两个snapshot 之间的 diff,主要为了支持CDC**
   
   ```sql
      select * (增加列区分 是diff insert or delete ) from t@snapshot1 diff snapshot2;
                                            // read diff between snapshot1 and snapshot2
                                            // 参考 git diff 的语法
   ```
   
3. 从快照中导入全量/增量数据
   
      ```sql
       
       begin;
       create table t2;
       insert into t2 select * from t1@snapshot1;// full backup
         
       insert into t2 select * from t1@snapshot1 diff snapshot2 // incremental backup.
       commit;
   ```


- Ast

   Ast 需要支持snapshot read 相关语法, 处理from

- Plan 

   在bind 时，需要支持 from t1@snapshot 语法, 生成对应的 snapshot da/table 对象.

- 分布式 Scope 

    将plan 转换为可执行的分布式scope 时，需要用snapshot table 对象.

- Disttae 从TN 去 获取 旧的 snapshot data

1. 快照读可能需要从TN 中获取较旧的ckp, 而当前CN 状态机(PartitionState与Catalog)中的数据是基于一个最新的ckp + logtail，所以需要从TN 中拉取 旧的ckp, 可以通过RPC 方式，与TN 新建一条tcp链接去获取某个表的指定时间戳的ckp(是否新建链接还是复用已有的push 通道待定)?
<br>
  
2. 快照读需要一个独立的状态机，将旧的ckp 数据apply 到此状态机中. 当事务提交后，释放状态机.
    <br>
  
3. 如果请求的数据版本在当前CN 状态机中已经存在，可以复用已有状态机


### 2.2 详细设计

#### 2.2.1 Disttae 设计

  **latest snapshot** : distate 中最新的快照数据，是基于最新的checkpoint + logtail apply 之后生成的.

  **global txn op** :  用户每开启一个事务，需要创建一个global txn op, 其负责事务的提交，与TN 之间交互等.
                              如果事务需要进行快照读，需要clone出一个新的snapshot op, 用于读取快照数据, 

​                            一个global txn op 中可以clone多个snapshot op.

  disttae 中对于每个snapshot 会对应一个独立的状态机，用于存储快照数据, latest snapshot 只是snapshot的特例.  如果快照读需要历史数据， distate会从TN中拉取，并apply到每个snapshot对应的状态机中; 
  如果请求的数据版本在当前CN latest snapshot 中已经存在，可以复用已有状态机. disttae 中 snapshot 的释放策略待定.

  当事务进行快照读时，sql  layer应调用txnOp.CloneSnapshotOp() 生成一个新的snapshot op, 用于读取快照数据, 并通过调用  Engine.Database(..., snapshotOp) open 一个snapshot database 对象, distate 会将snapshot op 与 snapshot database 绑定;  调用snapshotDb.Relation(..., tableName) open 一个snapshot table 对象，最后通过调用snapshotTable.NewReader(...)  生成一个snapshot reader对象读取snapshot数据.

  disttae engine 全局只有一个.

  在快照读功能之前，一个事务只有一个 txn op, 且txn op 与 distate.Transaction 对象一一对应;
  在快照读功能之后，一个事务会有多个txn op 对象，distate.Transaction 对象与txn op 为一对多关系.


##### 2.2.1.1 TxnOp 中增加CloneSnapshotOp 接口

```go
type TxnOp interface {
	//clone read-only snapshot op from parent op.
    CloneSnapshotOp(snapshot types.TS) TxnOp
}
```
  CloneSnapshotOp 通过global TxnOp clone出一个新的snapshot op,其snapshot ts 由用户指定，用于读取快照数据, 如果用户指定的snapshot ts与 父txn op的snapshot ts 相同,则其能看到global txn op workspace 中的 数据.

  sql layer通过调用Engine.Database(..., snapshotOp) open 一个snapshot database 对象, distate 会将snapshot op与snapshot database 绑定;

  snapshot op 会与其父op共享同一个Transaction对象, 但是其会有独立的snapshot data, 以便服务历史数据的读取.

##### 2.2.2.2 reader相关修改
   接口暂时不需要改动, 实现上改动比较小.

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

disttae 需要从TN 拉取 旧的ckp 数据及logtail , 建议用pull, non-lazyload 方式, 可以复用之前的code:


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



























 

