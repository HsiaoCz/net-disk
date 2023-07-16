## 网盘

分为三个模块:

用户模块:

- 密码登录
- 邮箱注册
- 个人资料详情

存储池模块

中心存储池管理模块

- 文件存储

个人存储池资源管理

- 文件关联存储
- 文件夹层级管理

文件共享模块

- 文件分享

### 1、表设计

用户表

```go
type UserBasic struct{
    gorm.Model
    Identity string `gorm:"clomun:identity;type:varchar(36);"` // 用户的唯一表示
    Username string `gorm:"clomun:username;type:varchar(60);"` //用户名
    Password string `gorm:"clomun:password;type:varchar(32);"` // 密码
    Email    string `gorm:"clomun:email;type:varchar(100);"`  // 邮箱
}

```

中心存储池

```go
type RepositoryPool struct{
    gorm.Model
    Identity string `gorm:"clomun:identity;type:varchar(36);"` // 记录的唯一表示
    Hash  string `gorm:"clomun:hash;type:varchar(32);"` // 文件的唯一标识
    Filename string  `gorm:"clomun:filename;type:varchar(255);"` // 文件的名称
    Ext   string `gorm:"clomun:ext;type:varchar(30);"`  // 文件的扩展名
    Size  float64 `gorm:"clomun:size;type:double;"` // 文件的大小
    Path  string  `gorm:"clomun:path;type:varchar(255);"` //文件的路径
}
```

个人存储池

```go
type UserReposiroty struct{
    gorm.Model
    Identity string `gorm:"cloumn:identity;type:varchar(36);"`
    UserIdentity string `gorm:"cloumn:user_identity;type:varchar(36);"` // 用户的唯一标识,知道是哪个人的文件
    ParentID int `gorm:"cloumn:parent_id;type:int;"` // 父级id 支持无限存储，可以建立n个文件夹
    RepositoryIdentity string `gorm:"cloumn:repository_identity;type:varchar(36);"` // 关联中心存储池的唯一标识
    Ext string `gorm:"cloumn:ext;type:varchar(255);"` // 文件或文件夹类型，如果为空，说明是一个文件夹，这个字段代表文件的扩展名
    Name string `gorm:"cloumn:name;type:varchar(255);"` // 当前中心存储池的名称
}
```

分享表

```go
type Share_Basic struct{
    gorm.Model
    Identity string `gorm:"clomun:identity;type:varchar(36);"`
    UserIdentity string `gorm:"clomun:user_identity;type:varchar(36);"` // 谁分享的，分享者的唯一标识
    RepositoryIdentity string `gorm:"clomun:repository_identity;type:varchar(36);"` // 分享的是什么内容
    ExpireTime int `gorm:"clomun:expire_identity;type:int;"` // 分享内容的失效时间，单位秒，如果传0，永不失效
}
```

这里用的是 gorm，只是标识一下
实际还是用 xorm

### 2、连接数据库

这里使用 xorm

xorm

```bash
go get -u  xorm.io/xorm
```

xorm 的初始化操作

engine 操作单数据库
engineGroup 对读写分离或者负载均衡数据库进行操作

```go
import (
    _ "github.com/go-sql-driver/mysql"
    "xorm.io/xorm"
)

var engine *xorm.Engine

func main() {
    var err error
    engine, err = xorm.NewEngine("mysql", "root:123@/test?charset=utf8")
}
```

xorm 有几个要点:

- 表结构 tag 可以这样定义

```go
type user struct{
    Username string `xorm:"varchar(36) notnull 'username' comment('姓名')"`
}
```

- xorm 也可以使用 TableName() string 指定表名

- 类似于 gorm 的 automegre，xorm 有 sync

```go
err := engine.Sync(new(User), new(Group))
```

- 插入数据

```go
user := new(User)
user.Name = "myname"
affected, err := engine.Insert(user)
// INSERT INTO user (name) values (?)
```

- Alias 给表设置别名

```go
engine.Alias("o").Where("o.name = ?", name).Get(&order)

// where 后面还可以跟and

engine.Where(...).And(...).Get(&order)

// Asc 指定正序排列
// Desc 指定倒序
engine.Asc("id").Find(&orders)

// 指定select语句的字段部分内容，例如
engine.Select("a.*, (select name from b limit 1) as name").Find(&beans)

// SQL可以直接使用sql语句
engine.SQL("select * from table").Find(&beans)

// Where
engine.Where("a = ? AND b = ?", 1, 2).Find(&beans)

// Cols(…string)
// 只查询或更新某些指定的字段，默认是查询所有映射的字段或者根据Update的第一个参数来判断更新的字段
engine.Cols("age", "name").Get(&usr)
// SELECT age, name FROM user limit 1
engine.Cols("age", "name").Find(&users)
// SELECT age, name FROM user
engine.Cols("age", "name").Update(&user)
// UPDATE user SET age=? AND name=?

// Omit
// 指定排除某些状态
// 例1：
engine.Omit("age", "gender").Update(&user)
// UPDATE user SET name = ? AND department = ?
// 例2：
engine.Omit("age, gender").Insert(&user)
// INSERT INTO user (name) values (?) // 这样的话age和gender会给默认值
// 例3：
engine.Omit("age", "gender").Find(&users)
// SELECT name FROM user //只select除age和gender字段的其它字段

```

- get
  查询单条数据使用 Get 方法，在调用 Get 方法时需要传入一个对应结构体的指针，同时结构体中的非空 field 自动成为查询的条件和前面的方法条件组合在一起查询

```go
user := new(User)
has, err := engine.ID(id).Get(user)
// 复合主键的获取方法
// has, errr := engine.ID(xorm.PK{1,2}).Get(user)

user := new(User)
has, err := engine.Where("name=?", "xlw").Get(user)

user := &User{Id:1}
has, err := engine.Get(user)
```

- find 查询多条记录
  查询多条数据使用 Find 方法，Find 方法的第一个参数为 slice 的指针或 Map 指针，即为查询后返回的结果，第二个参数可选，为查询的条件 struct 的指针

```go
// 传入slice
everyone := make([]Userinfo, 0)
err := engine.Find(&everyone)

pEveryOne := make([]*Userinfo, 0)
err := engine.Find(&pEveryOne)

// 传入Map用户返回数据，map必须为map[int64]Userinfo的形式，map的key为id，因此对于复合主键无法使用这种方式。
users := make(map[int64]Userinfo)
err := engine.Find(&users)

pUsers := make(map[int64]*Userinfo)
err := engine.Find(&pUsers)
```

- count 用于计数

```go
user := new(User)
total, err := engine.Where("id >?", 1).Count(user)
```

- Exist 判断某个记录是否存在

如果你的需求是：判断某条记录是否存在，若存在，则返回这条记录。

建议直接使用 Get 方法。

如果仅仅判断某条记录是否存在，则使用 Exist 方法，Exist 的执行效率要比 Get 更高

```go
has, err = testEngine.Where("name = ?", "test1").Exist(&RecordExist{})
// SELECT * FROM record_exist WHERE name = ? LIMIT 1
```
