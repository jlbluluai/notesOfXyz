### 安装

&emsp;&emsp;官方提供了安装版以及解压版，我这边直接用的解压版，Windows版本：4.0.9。
&emsp;&emsp;将内容解压到你想存放的地方，切到bin目录下，Shift+右键进入powershell（cmd慢慢切到该目录也一样）。执行如下命令，后面的地址是你数据库想放的目录。

```
.\mongod.exe --dbpath E:\mongodb\data
```

&emsp;&emsp;正常来说，这个dos面板执行后已经启动了mongodb，但是会发现没有命令行让我输入了，这时候我们再启动一个dos面板，到以上同样目录下执行以下命令。

```
.\mongo localhost:27017
```



### 创建数据库

&emsp;&emsp;这时候启动的dos面板我们已经可以操作，第一步我们先来创建数据库。

```
> use test//切至test数据库，若没有直接创建
switched to db test
> show dbs//展示当前有的数据库，可以看到刚创建的test没有，因为没有数据，插入一条就会显示
admin   0.000GB
config  0.000GB
local   0.000GB
> db//查看当前数据库
test
> db.test.insert({"name":"测试"})//插入一条数据
WriteResult({ "nInserted" : 1 })
> show dbs//再次展示当前有的数据库，test已经有了
admin   0.000GB
config  0.000GB
local   0.000GB
test    0.000GB
```



### 删除数据库

```
> db.dropDatabase()//确保当前db为del，否则默认删test
{ "dropped" : "del", "ok" : 1 }
```



### 插入文档

```
> use test
switched to db test
> document=({title:"Hello",content:"你好",count:100})//创建一个文档变量
{ "title" : "Hello", "content" : "你好", "count" : 100 }
> db.test.insert(document)//插入文档变量，文档变量内容直接放这插入效果一样
WriteResult({ "nInserted" : 1 })
> db.test.find()//查询test数据库，连带之前的数据都在
{ "_id" : ObjectId("5cbd84e6287ffd1f5a944795"), "name" : "测试" }
{ "_id" : ObjectId("5cbd8a92287ffd1f5a944797"), "title" : "Hello", "content" : "你好", "count" : 100 }
```

集合中键重复处理方式

```
> document=({title:"Hello",title:"你好",count:100})))//后一个覆盖前一个
{ "title" : "你好", "count" : 100 }
```



### 更新文档

```
> db.test.update({count:100},{$set:{count:200}})//将count的100改为200
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
> db.test.find()//查询后值已经改变
{ "_id" : ObjectId("5cbd84e6287ffd1f5a944795"), "name" : "测试" }
{ "_id" : ObjectId("5cbd8a92287ffd1f5a944797"), "title" : "Hello", "content" : "你好", "count" : 200 }
```



### 删除文档

```
> db.test.remove({"name":"测试"})//删除上面的一条数据
WriteResult({ "nRemoved" : 1 })
> db.test.find()//再次查询就没了
{ "_id" : ObjectId("5cbd8a92287ffd1f5a944797"), "title" : "Hello", "content" : "你好", "count" : 200 }
> db.test.remove({})//整个库的文档全删
```



### 创建索引

```
  > db.test.ensureIndex({"title":1})//指明索引的key，1代表升序，-1代表降序
```

