## Orm 使用方法

beego/orm 的使用例子

后文例子如无特殊说明都以这个为基础。

##### models.go:

```go
package main

import (
	"github.com/astaxie/beego/orm"
)

type User struct {
	Id          int        `orm:"auto"`     // 设置为auto主键
	Name        string
	Profile     *Profile   `orm:"rel(one)"` // OneToOne relation
	orm.Manager // 每个model都需要定义orm.Manager
}

type Profile struct {
	Id          int     `orm:"auto"`
	Age         int16
	User        *User   `orm:"reverse(one)"` // 设置反向关系(可选)
	orm.Manager
}

func init() {
	// 需要在init中注册定义的model
	orm.RegisterModel(new(User), new(Profile))
}
```

##### main.go

```go
package main

import (
	"fmt"
	"github.com/astaxie/beego/orm"
	_ "github.com/go-sql-driver/mysql"
)

func init() {
	orm.RegisterDriver("mysql", orm.DR_MySQL)

	orm.RegisterDataBase("default", "mysql", "root:root@/orm_test?charset=utf8", 30)
}

func main() {
	o := orm.NewOrm()
	o.Using("default") // 默认使用 default，你可以指定为其他数据库

	profile := NewProfile()
	profile.Age = 30

	user := NewUser()
	user.Profile = profile
	user.Name = "slene"

	fmt.Println(o.Insert(profile))
	fmt.Println(o.Insert(user))
}
```

## 数据库的设置

#### RegisterDriver

三种数据库类型

```go
orm.DR_MySQL
orm.DR_Sqlite
orm.DR_Postgres
```

```go
// 参数1   driverName
// 参数2   数据库类型
// 这个用来设置 driverName 对应的数据库类型
// mysql / sqlite3 / postgres 这三种是默认已经注册过的，所以可以无需设置
orm.RegisterDriver("mysql", orm.DR_MySQL)
```

#### RegisterDataBase

orm 必须注册一个名称为 `default` 的数据库，用以作为默认使用。

```go
// 参数1   自定义数据库名称，用来在orm中切换数据库使用
// 参数2   driverName
// 参数3   对应的链接字符串
// 参数4   设置最大的空闲连接数，使用 golang 自己的连接池
orm.RegisterDataBase("default", "mysql", "root:root@/orm_test?charset=utf8", 30)
```

## ORM 接口使用

使用 orm 必然接触的 Ormer 接口，我们来熟悉一下

```go
var o Ormer
o = orm.NewOrm() // 创建一个 Ormer
// NewOrm 的同时会执行 orm.BootStrap (整个 app 只执行一次)，用以验证模型之间的定义并缓存。
```

* type Ormer interface {
	* [Read(Modeler) error](Object.md#read)
	* [Insert(Modeler) (int64, error)](Object.md#insert)
	* [Update(Modeler) (int64, error)](Object.md#update)
	* [Delete(Modeler) (int64, error)](Object.md#delete)
	* [M2mAdd(Modeler, string, ...interface{}) (int64, error)](Object.md#m2madd)
	* [M2mDel(Modeler, string, ...interface{}) (int64, error)](Object.md#m2mdel)
	* [LoadRel(Modeler, string) (int64, error)](Object.md#loadRel)
	* [QueryTable(interface{}) QuerySeter](#querytable)
	* [Using(string) error](#using)
	* [Begin() error](Transaction.md)
	* [Commit() error](Transaction.md)
	* [Rollback() error](Transaction.md)
	* [Raw(string, ...interface{}) RawSeter](#raw)
	* [Driver() Driver](#driver)
* }


#### QueryTable

传入表名，或者 Modeler 对象，返回一个 [QuerySeter](Query.md#queryseter)

```go
o := orm.NewOrm()
var qs QuerySeter
qs = o.QueryTable("user")
// 如果表没有定义过，会立刻 panic
```

#### Using

切换为其他数据库

```go
orm.RegisterDataBase("db1", "mysql", "root:root@/orm_db2?charset=utf8", 30)
orm.RegisterDataBase("db2", "sqlite3", "data.db", 30)

o1 := orm.NewOrm()
o1.Using("db1")

o2 := orm.NewOrm()
o2.Using("db2")

// 切换为其他数据库以后
// 这个 Ormer 对象的其下的 api 调用都将使用这个数据库

```

默认使用 `default` 数据库，无需调用 Using

#### Raw

使用 sql 语句直接进行操作

Raw 函数，返回一个 [RawSeter](Raw.md) 用以对设置的 sql 语句和参数进行操作

```go
o := NewOrm()
var r RawSeter
r = o.Raw("UPDATE user SET name = ? WHERE name = ?", "testing", "slene")
```

#### Driver

返回当前 orm 使用的 db 信息

```go
type Driver interface {
	Name() string
	Type() DriverType
}
```

```go
orm.RegisterDataBase("db1", "mysql", "root:root@/orm_db2?charset=utf8", 30)
orm.RegisterDataBase("db2", "sqlite3", "data.db", 30)

o1 := orm.NewOrm()
o1.Using("db1")
dr := o1.Driver()
fmt.Println(dr.Name() == "db1") // true
fmt.Println(dr.Type() == orm.DR_MySQL) // true

o2 := orm.NewOrm()
o2.Using("db2")
dr = o2.Driver()
fmt.Println(dr.Name() == "db2") // true
fmt.Println(dr.Type() == orm.DR_Sqlite) // true

```
