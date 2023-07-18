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

- lterate Iterate 方法提供逐条执行查询到的记录的方法，他所能使用的条件和 Find 方法完全相同

```go
err := engine.Where("age > ? or name=?)", 30, "xlw").Iterate(new(Userinfo), func(i int, bean interface{})error{
    user := bean.(*Userinfo)
    //do somthing use i and user
})
```

- join 连接查询

```go
Join(string,interface{},string)

第一个参数为连接类型，当前支持INNER, LEFT OUTER, CROSS中的一个值， 第二个参数为string类型的表名，表对应的结构体指针或者为两个值的[]string，表示表名和别名， 第三个参数为连接条件

type Group struct {
	Id int64
	Name string
}

type User struct {
	Id int64
	Name string
	GroupId int64 `xorm:"index"`
}

type UserGroup struct {
    User `xorm:"extends"`
    Name string
}

func (UserGroup) TableName() string {
	return "user"
}

users := make([]UserGroup, 0)
engine.Join("INNER", "group", "group.id = user.group_id").Find(&users)

// join还可以直接使用SQL

users := make([]UserGroup, 0)
engine.Sql("select user.*, group.name from user, group where user.group_id = group.id").Find(&users)
```

- Rows
  Rows 方法和 Iterate 方法类似，提供逐条执行查询到的记录的方法，不过 Rows 更加灵活好用

```go
user := new(User)
rows, err := engine.Where("id >?", 1).Rows(user)
if err != nil {
}
defer rows.Close()
for rows.Next() {
    err = rows.Scan(user)
    //...
}
```

- xorm 事务

```go
func MyTransactionOps() error {
    session := engine.NewSession()
    defer session.Close()

    // add Begin() before any action
    if err := session.Begin(); err != nil {
        return err
    }

    user1 := Userinfo{Username: "xiaoxiao", Departname: "dev", Alias: "lunny", Created: time.Now()}
    if _, err := session.Insert(&user1); err != nil {
        return err
    }
    user2 := Userinfo{Username: "yyy"}
    if _, err = session.Where("id = ?", 2).Update(&user2); err != nil {
        return err
    }

    if _, err = session.Exec("delete from userinfo where username = ?", user2.Username); err != nil {
        return err
    }

    // add Commit() after all actions
    return session.Commit()
}
```

### 3、看看接口

```go
//注册服务
service core-api {
	//用户的登录
	@handler UserLogin
	post /user/login(LoginRequest) returns(LoginResponse)
	//用户的详情
	@handler UserDetail
	get /user/detail(UserDetailRequest) returns(UserDetailResponse)
	//验证码发送
	@handler  MailCodeSend
	post /mail/code/send(MailCodeSendRequest) returns(MailCodeSendResponse)
	//用户注册
	@handler UserRegister
	post /user/register(UserRegisterRequest) returns(UserRegisterResponse)
	//获取资源详细信息
	@handler ShareBasicDetail
	get /share/basic/detail (ShareBasicDetailRequest) returns (ShareBasicDetailResponse)
}

@server(
	middleware: Auth
)

service core-api{
	//文件上传
	@handler FileUpload
	post /file/upload(FileUploadRequest) returns(FileUploadResponse)
	//用户文件的关联存储
	@handler UserRepositorySave
	post /user/repository/save(UserRepositorySaveRequest) returns(UserRepositorySaveResponse)
	//用户文件列表
	@handler UserFileList
	get /uer/file/list(UserFileListRequest) returns(UserFileListResponse)
	//用户的名称修改
	@handler UserFileNameUpdate
	post /user/file/name/update(UserFileNameUpdateRequest) returns(UserFileNameUpdateResponse)
	//用户文件夹创建
	@handler UserFileFolderCreate
	post /user/folder/create (UserFileFolderCreateRequest) returns (UserFileFolderCreateResponse)
	//用户文件的删除
	@handler UserFileDetele
	delete /user/file/delete(UserFileDeleteRequest) returns (UserFileDeleteResponse)
	//文件移动
	@handler UserFileMove
	put /user/file/move (UserFileMoveRequest) returns (UserFileMoveResponse)
	//创建分享记录
	@handler ShareBasicCreate
	post /share/basic/create(ShareBasicCreateRequest) returns(ShareBasicCreateResponse)
	//分享文件资源的保存
	@handler ShareBasicSave
	post /share/basic/save (ShareBasicSaveRequest) returns (ShareBasicSaveResponse)
	// 刷新Authorization
	@handler RefreshAuthorization
	post /refresh/authorization (RefreshAuthorizationRequest) returns (RefreshAuthorizationResponse)
}

//刷新Token的token
type RefreshAuthorizationRequest{}
type RefreshAuthorizationResponse {
	Token        string `json:"token"`
	RefreshToken string `json:"refresh_token"`
}
//分享文件资源保存
type ShareBasicSaveRequest {
	RepositoryIdentity string `json:"repository_identity"`
	ParentId           int64  `json:"parent_id"`
}
type ShareBasicSaveResponse {
	Identity string `json:"identity"`
}
//获取资源详情 这里没有登录的用户也可以获取资源的详情
type ShareBasicDetailRequest {
	Identity string `json:"identity"`
}
type ShareBasicDetailResponse {
	Name               string `json:"name"`
	Ext                string `json:"ext"`
	Size               int64  `json:"size"`
	Path               string `json:"path"`
	RepositoryIdentity string `json:"repository_identity"`
}
//创建文件分享
type ShareBasicCreateRequest {
	UserRepositoryIdentity string `json:"user_repository_identity"`
	ExpiredTime            int64  `json:"expired_time"`
}
type ShareBasicCreateResponse {
	Identity string `json:"identity"`
}
//文件移动
type UserFileMoveRequest {
	Identity       string `json:"identity"`
	ParentIdentity string `json:"parent_identity"`
}
type UserFileMoveResponse {
	Message string `json:"message"`
}
//用户文件删除
type UserFileDeleteRequest {
	Identity string `json:"identity"`
}

type UserFileDeleteResponse {
	Message string `json:"message"`
}

//用户文件夹创建
type UserFileFolderCreateRequest {
	ParentId int64  `json:"parent_id"`
	Name     string `json:"name"`
}

type UserFileFolderCreateResponse {
	Identity string `json:"identity"`
	Message  string `json:"message"`
}
//用户文件名称修改
type UserFileNameUpdateRequest {
	Identity string `json:"identity"`
	Name     string `json:"name"`
}

type UserFileNameUpdateResponse {
	Message string `json:"message"`
}

//用户文件列表
type UserFileListRequest {
	Id   int64 `json:"id,optional"`
	Page int64 `json:"id,optional"`
	Size int64 `json:"size,optional"`
}
type UserFileListResponse {
	List  []*UserFile `json:"list"`
	Count int64       `json:"count"`
}

type UserFile {
	Id                 int64  `json:"id"`
	Name               string `json:"name"`
	Size               int64  `json:"size"`
	Identity           string `json:"identity"`
	RepositoryIdentity string `json:"repository_identity"`
	Ext                string `json:"ext"`
	Path               string `json:"path"`
}
//用户关联存储
type UserRepositorySaveRequest {
	ParentId           int64  `json:"parent_id"`
	RepositoryIdentity string `json:"repository_identity"`
	Ext                string `json:"ext"`
	Name               string `json:"name"`
}

type UserRepositorySaveResponse {
	Message  string `json:"message"`
	Identity string `json:"identity"`
}

//文件上传
type FileUploadRequest {
	Hash string `json:"hash,optional"`
	Name string `json:"name,optional"`
	Ext  string `json:"ext,optional"`
	Size int64  `json:"size,optional"`
	Path string `json:"path,optional"`
}

type FileUploadResponse {
	Message  string `json:"message"`
	Identity string `json:"identity"`
	Ext      string `json:"ext"`
	Name     string `json:"name"`
}
//用户登录
type LoginRequest {
	Name     string `json:"name"`
	Password string `json:"password"`
}

type LoginResponse {
	Token        string `json:"token"`
	RefleshToken string `json:"reflesh_token"`
}

type UserDetailRequest {
	Identity string `json:"identity"`
}

type UserDetailResponse {
	Name  string `json:"name"`
	Email string `json:"email"`
}

//邮箱验证码发送
type MailCodeSendRequest {
	Email string `json:"email"`
}

type MailCodeSendResponse {
	Message string `json:"message"`
}

//用户注册
type UserRegisterRequest {
	//用户名
	Name string `json:"name"`
	//密码
	Password string `json:"password"`
	//邮箱
	Email string `json:"email"`
	//验证码
	Code string `json:"code"`
}

type UserRegisterResponse {
	//消息
	Message string `json:"message"`
}

```
