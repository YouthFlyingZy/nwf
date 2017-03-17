# 针对postgres的数据库访问层
## 配置
在webserver.config.xml中配置连接postgres数据库的参数
```xml
<config>
    <table name='db'>
      <string name='host'>127.0.0.1</string>
      <string name='user_name'>postgres</string>
      <string name='user_password'>123456</string>
      <string name='database'>test</string>
      <string name='port'>5432</string>
    </table>  
  </config>
```
## 生成SQL脚本
可以使用sqlGenerator生成一些简单的脚本,在生成INSERT或者UPDATE语句之前,需要创建与数据库表映射的实体, 
主要用于插入value和set value的时候使用,类似下面的写法
```lua
local student = commonlib.gettable("entity.student");  
student.tbName = "student";  
student.fields = {
	id = {notNil = true, isPrimaryKey = true},
	name = {notNil = true},
	age = {notNil = true},
	class_id = {notNil = false},
	create_time = {notNil = false},
	is_deleted = {notNil = false}
}
```
then
```lua
local tb = {id="nextval('pf_upm_sys_user_sys_user_id_seq')",name="zhangsan",age=10};

local sqlGenerator = commonlib.gettable("nwf.db.sqlGenerator");

local sql = sqlGenerator:insert(commonlib.gettable("entity.student"))
			:value(tb)
			:get();
--更新的时候会忽略tb中的id,即不更新id
sql = sqlGenerator:update(commonlib.gettable("entity.student"))
		  :value(tb)
		  :where("id = ",tb.id)
		  :_and(nil,"create_time = now()")
		  :get();

sql = sqlGenerator:select(" s.name , s.age ")
		  :append("FROM student s LEFT JOIN class c ON c.id = s.id")
		  :where(nil,"create_time = now()")
		  :get();
```
##  使用 DbTemplete
```lua
local dbTemplate = commonlib.gettable("nwf.db.dbTemplate");
```
### 基本用法
```lua
dbTemplate.execute(sql)
dbTemplate.executeWithTransaction(sql1,sql2,...)
```

### 执行sql，同时控制连接对象的释放
```lua
dbTemplate.executeWithReleaseCtrl(sql, conn, release ,openTransaction)

local openTransaction = true
local conn = dbTemplate.executeWithReleaseCtrl(sql1, nil, false, openTransaction)
dbTemplate.executeWithReleaseCtrl(sql2, conn, false, openTransaction)
...
--在执行最后一条sql脚本的时候 设置release = true
dbTemplate.executeWithReleaseCtrl(sqlN, conn, true, openTransaction)
```

### 关联查询
#### Tips
* 为每个表的字段设置别名, [mainTbPrefix_alias], [fromTbPrefix1_alias], [fromTbPrefix2_alias]...
* 数据库的表的主键的别名 mainTbPrefix_id, fromTbPrefix1_id
* tbAliasPrefix = {mainTbPrefix, fromTbPrefix1, fromTbPrefix2, ...}
* 如果不想使用关联查询 , 设置 tbAliasPrefix = nil 

```lua
dbTemplate:queryFirst(sql, tbAliasPrefix)

local sql = SELECT c.id as class_id,
		   c.name as class_name,
		   s.id as student_id,
		   s.name as student_name,
		   s.age as student_age,
		   s.class_id as student_classId ,
		   x.id as xxx_id,
		   x.name as xxx_name,
		   x.class_id as xxx_classId 
	   FROM class c 
	   LEFT JOIN student s ON s.class_id = c.id
	   LEFT JOIN xxx x ON x.class_id = c.id
	   WHERE c.id = 2;
local data,err = dbTemplate:queryFirst(sql, {"class","student","xxx"});
```

### 分页查询
#### Tips
* `countSql` 不能为nil, `pageIndex` 和 `pageSize` 必须大于0
* sql的分页子句: `LIMIT %d OFFSET %d`
```lua
dbTemplate:queryList(sql, tbAliasPrefix, countSql, pageIndex, pageSize)

local sql = SELECT c.id as class_id,
		   c.name as class_name,
		   s.id as student_id,
		   s.name as student_name,
		   s.age as student_age,
		   s.class_id as student_classId ,
		   x.id as xxx_id,
		   x.name as xxx_name,
		   x.class_id as xxx_classId 
           FROM (SELECT id, name FROM class LIMIT %d OFFSET %d) c 
	   LEFT JOIN student s ON s.class_id = c.id
	   LEFT JOIN xxx x ON x.class_id = c.id;
local data,err = dbTemplate:queryList(sql, {"class","student","xxx"}, " select count(1) from class ", 1, 3);
```

