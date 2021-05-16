---
title: Go快速上手--数据存储（redis,mysql,mongodb）
date: 2021-05-16 19:16:08
tags: [go]	
categories: 技术
---

# Redis

安装和启动Redis
```
sudo apt install redis-server

sudo service redis-server start

go get github.com/go-redis/redis/v8
```
通过cli连接redis
```
redis-cli -h 127.0.0.1 -p 6379 
```
redis 五大数据结构:

![image](./1.png)

redis五大数据结构的使用实践案例如下：
```go
package main

import (
	"context"
	"fmt"
	"time"
	"github.com/go-redis/redis/v8"
)

var ctx = context.Background()

func main() {
	rdb := redis.NewClient(&redis.Options{
		Addr:     "127.0.0.1:6379",
		Password: "",
		DB:       0,
	})

	pong, err := rdb.Ping(ctx).Result()
	if err != nil {
		fmt.Printf("连接redis出错，错误信息：%v", err)
		return
	} else {
		fmt.Println("成功连接redis,", pong)
	}


	// ======================key val get set============================================

	// set key val操作
	err = rdb.Set(ctx, "guid", "89_1909979_2232118", 0).Err()
	if err != nil {
		fmt.Printf("redis set失败，错误信息：%v", err)
	}

	// get key val操作
	val, err := rdb.Get(ctx, "guid").Result()
	if err != nil {
		fmt.Printf("redis get失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis get成功，key：%s, val:%s\n", "guid", val)
	}

	// 删除一个key
	err = rdb.Del(ctx, "guid").Err()
	if err != nil {
		fmt.Printf("redis Del失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis Del成功，key：%s\n", "guid")
	}

	val, err = rdb.Get(ctx, "guidxx").Result()
	if err != nil {
		fmt.Printf("redis get失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis get成功，key：%s, val:%s\n", "guidxx", val)
	}

	// set key val NX, key不存在才set, 并设置指定过期时间
	err = rdb.SetNX(ctx, "task", "123", 10*time.Second).Err()
	if err != nil {
		fmt.Printf("redis SetNX失败，错误信息：%v\n", err)
	}

	time.Sleep(time.Duration(1)*time.Second)

	// 获取过期剩余时间
	tm, err := rdb.TTL(ctx, "task").Result()
	if err != nil {
		fmt.Printf("redis TTL失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis TTL成功，key：%s, tm:%v\n", "task", tm)
	}

	val, err = rdb.Get(ctx, "task").Result()
	if err != nil {
		fmt.Printf("redis get失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis get成功，key：%s, val:%s\n", "task", val)
	}

	time.Sleep(time.Duration(11)*time.Second)

	val, err = rdb.Get(ctx, "task").Result()
	if err != nil {
		fmt.Printf("redis get失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis get成功，key：%s, val:%s\n", "task", val)
	}

	// 对key设置newValue这个值，并且返回key原来的旧值。原子操作。
	oldVal, err := rdb.GetSet(ctx, "task", "456").Result()
	if err != nil {
		fmt.Printf("redis GetSet失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis GetSet成功，key：%s, oldVal:%s\n", "task", oldVal)
	}

	// 批量get set
	err = rdb.MSet(ctx, "key1", "val1", "key2", "val2", "key3", "val3").Err()
	if err != nil {
		fmt.Printf("redis MSet失败，错误信息：%v\n", err)
	} 

	vals, err := rdb.MGet(ctx, "key1", "key2", "key3").Result()
	if err != nil {
		fmt.Printf("redis MGet失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis MGet成功，vals：%v\n", vals)
	}

	// 自增自减，原子操作
	err = rdb.Incr(ctx, "age").Err()
	if err != nil {
		fmt.Printf("redis Incr失败，错误信息：%v\n", err)
	}
	// +5
	err = rdb.IncrBy(ctx, "age", 5).Err()
	if err != nil {
		fmt.Printf("redis IncrBy失败，错误信息：%v\n", err)
	}

	err = rdb.Decr(ctx, "age").Err()
	if err != nil {
		fmt.Printf("redisDecr失败，错误信息：%v\n", err)
	}
	// -3
	err = rdb.DecrBy(ctx, "age", 3).Err()
	if err != nil {
		fmt.Printf("redis DecrBy失败，错误信息：%v\n", err)
	} 

	// 2
	val, err = rdb.Get(ctx, "age").Result()
	if err != nil {
		fmt.Printf("redis get失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis get成功，key：%s, val:%s\n", "age", val)
	}

	// 对key设置过期时间
	rdb.Expire(ctx, "key1", 3*time.Second)

	tm, err = rdb.TTL(ctx, "key1").Result()
	if err != nil {
		fmt.Printf("redis TTL失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis TTL成功，key：%s, tm:%v\n", "key1", tm)
	}

	//批量删除
	err = rdb.Del(ctx, "key1", "key2", "key3").Err()
	if err != nil {
		fmt.Printf("redis Del失败，错误信息：%v\n", err)
	}

	// =====================list=============================

	// LPushX 从左往右插入
	// //仅当列表存在的时候才插入数据,此时列表不存在，无法插入
	err = rdb.LPushX(ctx, "uids", 120).Err()
	if err != nil {
		fmt.Printf("redis LPushX失败，错误信息：%v\n", err)
	}

	// /此时列表不存在，依然可以插入
	err = rdb.LPush(ctx, "uids", 130).Err()
	if err != nil {
		fmt.Printf("redis LPush失败，错误信息：%v\n", err)
	}

	// 批量插入
	err = rdb.LPushX(ctx, "uids", 130, 140, 154, 132).Err()
	if err != nil {
		fmt.Printf("redis LPush失败，错误信息：%v\n", err)
	}

	// 返回全部数据
	vals2, err := rdb.LRange(ctx, "uids", 0, -1).Result()
	if err != nil {
		fmt.Printf("redis LRange失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis LRange成功，key：%s, vals:%v\n", "uids", vals2)  // redis LRange成功，key：uids, vals:[132 154 140 130 130]
	}

	// 取队列[2,4]的数据，即第3，4，5位置数据
	vals2, err = rdb.LRange(ctx, "uids", 2, 4).Result()
	if err != nil {
		fmt.Printf("redis LRange失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis LRange成功，key：%s, vals:%v\n", "uids", vals2)  // redis LRange成功，key：uids, vals:[140 130 130]
	}

	// 返回队列长度
	llen, err := rdb.LLen(ctx, "uids").Result()
	if err != nil {
		fmt.Printf("redis LLen失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis LLen成功，key：%s, llen:%v\n", "uids", llen)  // redis LLen成功，key：uids, llen:5
	}

	// 修改list中指定位置的值
	err = rdb.LSet(ctx, "uids", 2, 1000).Err()
	if err != nil {
		fmt.Printf("redis LSet失败，错误信息：%v\n", err)
	}

	// 返回全部数据
	vals2, err = rdb.LRange(ctx, "uids", 0, -1).Result()
	if err != nil {
		fmt.Printf("redis LRange失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis LRange成功，key：%s, vals:%v\n", "uids", vals2)  //redis LRange成功，key：uids, vals:[132 154 1000 130 130]
	}

	// 出队列，左进右出
	err = rdb.RPop(ctx, "uids").Err()
	if err != nil {
		fmt.Printf("redis RPop失败，错误信息：%v\n", err)
	}

	vals2, err = rdb.LRange(ctx, "uids", 0, -1).Result()
	if err != nil {
		fmt.Printf("redis LRange失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis LRange成功，key：%s, vals:%v\n", "uids", vals2) 
	}


	// 出队列，左进右出。没有就会阻塞，可以设置阻塞超时值
	err = rdb.BRPop(ctx, time.Duration(1)*time.Second, "uids").Err()
	if err != nil {
		fmt.Printf("redis BRPop失败，错误信息：%v\n", err)
	}

	// 删除一定位置范围内的值。删除count个key的list中值为value 的元素。如果出现重复元素，仅删除1次，也就是删除第一个
	err = rdb.LRem(ctx, "uids", 3, 130).Err()
	if err != nil {
		fmt.Printf("redis LRem失败，错误信息：%v\n", err)
	}

	// 返回全部数据
	vals2, err = rdb.LRange(ctx, "uids", 0, -1).Result()
	if err != nil {
		fmt.Printf("redis LRange失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis LRange成功，key：%s, vals:%v\n", "uids", vals2)  //redis LRange成功，key：uids, vals:[132 154 1000]
	}

	// =====================================集合操作=============================
	//redis集合特性：元素无序且唯一

	// 批量入集合
	err = rdb.SAdd(ctx, "students", "Alice", "James", "James").Err()
	if err != nil {
		fmt.Printf("redis SAdd失败，错误信息：%v\n", err)
	}

	// 获取集合大小
	size, err := rdb.SCard(ctx, "students").Result()
	if err != nil {
		fmt.Printf("redis SCard失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis SCard成功，key：%s, size:%v\n", "students", size)  //redis SCard成功，key：students, size:2
	}

	// 返回集合所有元素
	sMem, err := rdb.SMembers(ctx, "students").Result()
	if err != nil {
		fmt.Printf("redis SMembers失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis SMembers成功，key：%s, size:%v\n", "students", sMem)  //redis SMembers成功，key：students, size:[James Alice]
	}

	// 判断元素是否在集合中
	flag, err := rdb.SIsMember(ctx, "students", "James").Result()
	if err != nil {
		fmt.Printf("redis SIsMember失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis SIsMember成功，key：%s, size:%v\n", "students", flag)  //redis SIsMember成功，key：students, size:true
	}

	// 删除集合元素
	err = rdb.SRem(ctx, "students", "Alice").Err()
	if err != nil {
		fmt.Printf("redis SRem失败，错误信息：%v\n", err)
	}

	// =============================hash操作========================
	// 多级嵌套HASH  China 是hash Guangdong 是字段名, Tencent是字段值
	err = rdb.HSet(ctx, "China", "Guangdong", "Tencent").Err()
	if err != nil {
		fmt.Printf("redisHSet失败，错误信息：%v\n", err)
	}
	
	hvar, err := rdb.HGet(ctx, "China", "Guangdong").Result()
	if err != nil {
		fmt.Printf("redis HGet失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis HGet成功，hvar:%v\n", hvar) 
	}

	err = rdb.HSet(ctx, "China", "Hanzhou", "Alibaba").Err()
	if err != nil {
		fmt.Printf("redis HSet失败，错误信息：%v\n", err)
	}

	// 返回的是个map
	hvarAll, err := rdb.HGetAll(ctx, "China").Result()
	if err != nil {
		fmt.Printf("redis HGetAll失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis HGetAll成功，hvarAll:%v\n", hvarAll)    //redis HGetAll成功，hvarAll:map[Guangdong:Tencent Hanzhou:Alibaba]
	}

	// 将map塞进hash
	batchData := make(map[string]interface{})
	batchData["username"] = "test"
	batchData["password"] = 123456
	err = rdb.HMSet(ctx, "users", batchData).Err()
	if err != nil {
		fmt.Printf("redis HMSet失败，错误信息：%v\n", err)
	}

	hvarAll, err = rdb.HGetAll(ctx, "users").Result()
	if err != nil {
		fmt.Printf("redis HGetAll失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis HGetAll成功，hvarAll:%v\n", hvarAll)   //redis HGetAll成功，hvarAll:map[password:123456 username:test]
	}

	// "Hanzhou"字段不存在才Set
	err = rdb.HSetNX(ctx, "China", "Hanzhou", "Netease").Err()
	if err != nil {
		fmt.Printf("redis HSetNX失败，错误信息：%v\n", err)
	}

	// 对key的值+n
	count, err := rdb.HIncrBy(ctx, "users", "password", 10).Result()
	if err != nil {
		fmt.Printf("redis HIncrBy失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis HIncrBy成功，count:%v\n", count)   // redis HIncrBy成功，count:123466
	}

	// 返回所有key,返回值是个string数组
	keys, err := rdb.HKeys(ctx, "China").Result()
	if err != nil {
		fmt.Printf("redisHKeys失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis HKeys成功，keys:%v\n", keys)   //redis HKeys成功，keys:[Guangdong Hanzhou]
	}

	// 根据key，查询hash的字段数量
	hlen, err := rdb.HLen(ctx, "China").Result()
	if err != nil {
		fmt.Printf("redis HLen失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis HLen成功，hlen:%v\n", hlen)   // redis HLen成功，hlen:2
	}

	//删除多个key
	err = rdb.HDel(ctx, "China", "Hanzhou", "Guangdong").Err()
	if err != nil {
		fmt.Printf("redis HDel失败，错误信息：%v\n", err)
	}

	//================================有序集合sort set================================================

	/*
		// Z 表示已排序的集合成员
		type Z struct {
			Score  float64  // 分数
			Member interface{} // 元素名
		}
	*/
	zsetKey := "companys_rank"
	companys := []*redis.Z{
		{Score: 100.0, Member: "Apple"},
		{Score: 90.0, Member: "MicroSoft"},
		{Score: 70.0, Member: "Amazon"},
		{Score: 87.0, Member: "Google"},
		{Score: 77.0, Member: "Facebook"},
		{Score: 67.0, Member: "Tesla"},
	}

	err = rdb.ZAdd(ctx, zsetKey, companys...).Err()
	if err != nil {
		fmt.Printf("redis ZAdd失败，错误信息：%v\n", err)
	}

	err = rdb.ZIncrBy(ctx, zsetKey, 2, "Amazon").Err()
	if err != nil {
		fmt.Printf("redis ZIncrBy失败，错误信息：%v\n", err)
	}

	//返回从0到-1位置的集合元素， 元素按分数从小到大排序 
	rank, err := rdb.ZRange(ctx, zsetKey, 0, -1).Result()
	if err != nil {
		fmt.Printf("redis ZRange失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis ZRange成功，rank:%v\n", rank)   //redis ZRange成功，rank:[Tesla Amazon Facebook Google MicroSoft Apple]
	}  

	//返回top3
	rank, err = rdb.ZRange(ctx, zsetKey, -3, -1).Result()
	if err != nil {
		fmt.Printf("redis ZRange失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis ZRange成功，rank:%v\n", rank)   //redis ZRange成功，rank:[Google MicroSoft Apple]
	}  

	op := &redis.ZRangeBy{
		Min:"80", // 最小分数
		Max:"100", // 最大分数
		Offset:0, // 类似sql的limit, 表示开始偏移量
		Count:2, // 一次返回多少数据
	}

	///根据分数范围返回集合元素，实现排行榜，取top n
	rank, err = rdb.ZRangeByScore(ctx, zsetKey, op).Result()
	if err != nil {
		fmt.Printf("redis ZRangeByScore失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis ZRangeByScore成功，rank:%v\n", rank)   //redis ZRangeByScore成功，rank:[Google MicroSoft]
	} 

	//根据元素名，查询集合元素在集合中的排名，从0开始算，集合元素按分数从小到大排序
	rk, err := rdb.ZRank(ctx, zsetKey, "Apple").Result()
	if err != nil {
		fmt.Printf("redis ZRank失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis ZRank成功，rk:%v\n", rk)  // redis ZRank成功，rk:5
	} 

	// 一次删除多个key
	err = rdb.ZRem(ctx, zsetKey, "Apple", "Amazon").Err()
	if err != nil {
		fmt.Printf("redis ZRem失败，错误信息：%v\n", err)
	}

	//集合元素按分数排序，从最低分到高分，删除第0个元素到第1个元素。 这里相当于删除最低分的2个元素
	err = rdb.ZRemRangeByRank(ctx, zsetKey, 0, 1).Err()
	if err != nil {
		fmt.Printf("redis ZRemRangeByRank失败，错误信息：%v\n", err) 
	}

	rank, err = rdb.ZRange(ctx, zsetKey, 0, -1).Result()
	if err != nil {
		fmt.Printf("redis ZRange失败，错误信息：%v\n", err)
	} else {
		fmt.Printf("redis ZRange成功，rank:%v\n", rank)     //redis ZRange成功，rank:[Google MicroSoft]
	}  

}
```

输出
```shell
成功连接redis, PONG
redis get成功，key：guid, val:89_1909979_2232118
redis Del成功，key：guid
redis get失败，错误信息：redis: nil
redis TTL成功，key：task, tm:-1ns
redis get成功，key：task, val:456
redis get成功，key：task, val:456
redis GetSet成功，key：task, oldVal:456
redis MGet成功，vals：[val1 val2 val3]
redis get成功，key：age, val:60
redis TTL成功，key：key1, tm:3s
redis LRange成功，key：uids, vals:[132 154 140 130 130]
redis LRange成功，key：uids, vals:[140 130 130]
redis LLen成功，key：uids, llen:5
redis LRange成功，key：uids, vals:[132 154 1000 130 130]
redis LRange成功，key：uids, vals:[132 154 1000 130]
redis LRange成功，key：uids, vals:[132 154 1000]
redis SCard成功，key：students, size:2
redis SMembers成功，key：students, size:[Alice James]
redis SIsMember成功，key：students, size:true
redis HGet成功，hvar:Tencent
redis HGetAll成功，hvarAll:map[Guangdong:Tencent Hanzhou:Alibaba]
redis HGetAll成功，hvarAll:map[password:123456 username:test]
redis HIncrBy成功，count:123466
redis HKeys成功，keys:[Guangdong Hanzhou]
redis HLen成功，hlen:2
redis ZRange成功，rank:[Tesla Amazon Facebook Google MicroSoft Apple]
redis ZRange成功，rank:[Google MicroSoft Apple]
redis ZRangeByScore成功，rank:[Google MicroSoft]
redis ZRank成功，rk:5
redis ZRange成功，rank:[Google MicroSoft]

```



# Mysql


命令行连接msyql
```
mysql -h mhxy-dev-130073-m.hz.dumbo.nie.netease.com -P 3306 -uxy1 -pmhxygamekingxy -A
```

mysql的CRUD实践案例：
```go
package main

import (
    "fmt"

    _ "github.com/go-sql-driver/mysql"
    "github.com/jmoiron/sqlx"
)

type Person struct {
    Cguid	string    `db:"cguid"`
    Username string `db:"name"`
}

var Db *sqlx.DB

//init()函数会在每个包完成初始化后自动执行，并且执行优先级比main函数高。
func init() {

    database, err := sqlx.Open("mysql", "root:@tcp(inner.mhxy.nie.netease.com)/pyc2021")
    if err != nil {
        fmt.Println("open mysql failed,", err)
        return
    }

    Db = database
}

func main() {

	// 查询
    var person []Person
    err := Db.Select(&person, "select cguid, name from pyc2021_match limit 10;")
    if err != nil {
        fmt.Println("exec failed, ", err)
        return
    }

    fmt.Println("select succ:", person)

	//通过Db.Select方法将查询的多行数据保存在一个切片中，然后就可以通过循环的方式获取每行数据
	for _, info := range person {
		cguid := info.Cguid
		name := info.Username
		fmt.Println("mysql select, ", cguid, name)
	}

	// 插入
    r, err := Db.Exec("insert into pyc2021_match(cguid, vote, name, hostnum)values('781122223', 200, '这是一个测试', 90)")
    if err != nil {
        fmt.Println("exec failed, ", err)
        return
    }
    id, err := r.LastInsertId()
    if err != nil {
        fmt.Println("exec failed, ", err)
        return
    }

    fmt.Println("insert succ:", id)

	// 更新
	res, err := Db.Exec("update pyc2021_match set name='修改测试' where cguid='98977999'")
    if err != nil {
        fmt.Println("exec failed, ", err)
        return
    }
    row, err := res.RowsAffected()
    if err != nil {
        fmt.Println("rows failed, ",err)
    }
    fmt.Println("update succ:",row)

	// 删除
	res, err = Db.Exec("delete from pyc2021_match where cguid='98977999'")
    if err != nil {
        fmt.Println("exec failed, ", err)
        return
    }

    row,err = res.RowsAffected()
    if err != nil {
        fmt.Println("rows failed, ",err)
    }

    fmt.Println("delete succ: ",row)

	Db.Close()
}
```

输出：
```shell
select succ: [{7822223 xxxx1} {7832223 xsxxx1} {7832233 xsx99} {22223 西红柿} {98977999 修改测试} {78222223 这是一个测试} {78122223 这是一个测试}]
mysql select,  7822223 xxxx1
mysql select,  7832223 xsxxx1
mysql select,  7832233 xsx99
mysql select,  22223 西红柿
mysql select,  98977999 修改测试
mysql select,  78222223 这是一个测试
mysql select,  78122223 这是一个测试
insert succ: 12
update succ: 0
delete succ:  1

```

# Mongodb

命令行连接mongodb
```
mongo mongodb://root:123@127.0.0.1:30002/xyq
```



mongodb的CRUD实践如下:
```go
package main

//https://github.com/tfogo/mongodb-go-tutorial

import (
   "go.mongodb.org/mongo-driver/bson"    //BOSN解析包
   "go.mongodb.org/mongo-driver/mongo"    //MongoDB的Go驱动包
   "go.mongodb.org/mongo-driver/mongo/options"
   "fmt"
   "log"
   "context"
)

type Users struct {
	Cguid string   `bson:"cguid"`
   Uid   int      `bson:"uid"`
   Text  string   `bson:"text"`
   Name  string   `bson:"name"`
}


func main() {

   //var ctx context.Context
   clientOptions := options.Client().ApplyURI("mongodb://root:mhxygameking@192.168.44.105:30002/xyq")

   // 建立客户端连接
   client, err := mongo.Connect(context.TODO(), clientOptions)
   if err != nil {
      log.Fatal(err)
      fmt.Println(err)
   }
   
   // 检查连接情况
   err = client.Ping(context.TODO(), nil)
   if err != nil {
      log.Fatal(err)
      fmt.Println(err)
   }
   fmt.Println("Connected to MongoDB!")
   
   //指定要操作的数据集
  // collection := client.Database("xyq").Collection("ljs_test")
   collection := client.Database("xyq").Collection("pyc2021_rank")
 
   //执行增删改查操作

   //插入一条数据
   newUser := Users{"89_34122417_1642765091", 129829, "新的文本", "sorrymaker"}
   res, err := collection.InsertOne(context.TODO(), newUser) 
   if err != nil { 
         log.Fatal(err) 
   } 
   fmt.Println("Inserted document: ", res.InsertedID) 

   //插入多条数据
   
   newUser1 := Users{"89_21932437_1643320091", 139829, "新的文本1", "sorrymaker"}
   newUser2 := Users{"89_31933227_164442021", 139129, "新的文本2", "sorrymaker"}
   newUser3 := Users{"89_41931237_1642121091", 139429, "新的文本3", "sorrymaker"}
   news := []interface{}{newUser1, newUser2, newUser3}
   res1, err := collection.InsertMany(context.TODO(), news) 
   if err != nil { 
         log.Fatal(err) 
   } 
   fmt.Println("Inserted document: ", res1.InsertedIDs) 

   // 查找一个数据
   var user Users
   filter := bson.D{{"cguid", "89_22932237_1649720099"}}
	if err = collection.FindOne(context.TODO(),filter).Decode(&user); err != nil{
		fmt.Println(err)
		return
	} 
   fmt.Printf("result:%+v\n", user)

   //查找多个符合条件的数据
   findOptions := options.Find()
   findOptions.SetLimit(10) //限制返回的条目
   var results []*Users
   filter = bson.D{{"name", "sorrymaker"}}
   cur, err := collection.Find(context.TODO(), filter, findOptions)
   if err != nil {
      fmt.Println(err)
		return
   }

   for cur.Next(context.TODO()) {
      var elem Users
      err := cur.Decode(&elem)
      if err != nil {
         fmt.Println(err)
         return
      }
      fmt.Printf("elem:%+v\n", elem)
      results = append(results, &elem)
   }

   if err := cur.Err(); err != nil {
      fmt.Println(err)
      return
   }
   cur.Close(context.TODO()) 


   //更新数据
   filter = bson.D{{"name", "sorrymaker"}} 
   update := bson.D{{"$set", bson.D{{"text", "修改成功"}}}} 
   updateResult, err := collection.UpdateOne(context.TODO(), filter, update) 
    if err != nil { 
         log.Fatal(err) 
    }
    fmt.Printf("Updated documents: %+v\n", updateResult) 


   //更新多条数据
   filter = bson.D{{"name", "sorrymaker"}} 
   update = bson.D{{"$set", bson.D{{"text", "udatemany修改成功"}}}} 
   updateResults, err := collection.UpdateMany(context.TODO(), filter, update) 
    if err != nil { 
         log.Fatal(err) 
    }
    fmt.Printf("UpdateMany documents: %+v\n", updateResults) 


   //更新数据，不存在就插入。upsert
   filter = bson.D{{"cguid", "89_22932437_1643120029"}} 
   update = bson.D{{"$set", bson.D{{"text", "修改成功upsert"}, {"cguid", "89_22932437_1643120029"}, {"uid",1982938}, {"name", "losjo"}}}} 
   updateOpts := options.Update().SetUpsert(true)  // 设置upsert模式
   updateResult, err = collection.UpdateOne(context.TODO(), bson.M{}, update, updateOpts)
   if err != nil {
       log.Fatal(err)
   }
   fmt.Printf("Upsert documents: %+v\n", updateResult) 

   // 删除一条数据
   filter = bson.D{{"cguid", "89_34922417_1641725099"}} 
   deleteResult, err := collection.DeleteOne(context.TODO(), filter)
   if err != nil {
      log.Fatal(err)
   }
   fmt.Printf("deleteone documents: %+v\n", deleteResult) 


   //给字段加索引
   indexModel := mongo.IndexModel{
      Keys: bson.D{
         {"name", 1},
      },
   }
   _, err = collection.Indexes().CreateOne(context.TODO(), indexModel)
   if err != nil {
      log.Fatal(err)
   }
   
   // 断开客户端连接
   err = client.Disconnect(context.TODO())
   if err != nil {
      log.Fatal(err)
   }
   fmt.Println("Connection to MongoDB closed.")
}
```

输出：
```shell
Connected to MongoDB!
Inserted document:  ObjectID("60a0a872d7ed08cd629ddabd")
Inserted document:  [ObjectID("60a0a872d7ed08cd629ddabe") ObjectID("60a0a872d7ed08cd629ddabf") ObjectID("60a0a872d7ed08cd629ddac0")]
result:{Cguid:89_22932237_1649720099 Uid:139829 Text:udatemany修改成功 Name:sorrymaker}
elem:{Cguid:89_32933237_1644720299 Uid:139129 Text:udatemany修改成功 Name:sorrymaker}
elem:{Cguid:89_42931237_1642721099 Uid:139429 Text:udatemany修改成功 Name:sorrymaker}
elem:{Cguid:89_42931237_1642121099 Uid:139429 Text:udatemany修改成功 Name:sorrymaker}
elem:{Cguid:89_42931237_1649721099 Uid:139429 Text:udatemany修改成功 Name:sorrymaker}
elem:{Cguid:89_32932417_1649725099 Uid:129829 Text:udatemany修改成功 Name:sorrymaker}
elem:{Cguid:89_22932237_1643720099 Uid:139829 Text:udatemany修改成功 Name:sorrymaker}
elem:{Cguid:89_32932237_1649720099 Uid:129829 Text:udatemany修改成功 Name:sorrymaker}
elem:{Cguid:89_32932237_1619720099 Uid:129829 Text:udatemany修改成功 Name:sorrymaker}
elem:{Cguid:89_32933237_1649720299 Uid:139129 Text:udatemany修改成功 Name:sorrymaker}
elem:{Cguid:89_22933237_1649720299 Uid:139129 Text:udatemany修改成功 Name:sorrymaker}
Updated documents: &{MatchedCount:1 ModifiedCount:1 UpsertedCount:0 UpsertedID:<nil>}
UpdateMany documents: &{MatchedCount:31 ModifiedCount:5 UpsertedCount:0 UpsertedID:<nil>}
Upsert documents: &{MatchedCount:1 ModifiedCount:1 UpsertedCount:0 UpsertedID:<nil>}
deleteone documents: &{DeletedCount:1}
Connection to MongoDB closed.
```