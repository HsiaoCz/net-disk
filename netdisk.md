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
type User_Basic struct{
    gorm.Model
    Identity string `gorm:"clomun:identity;type:varchar(36);"` // 用户的唯一表示
    Username string `gorm:"clomun:username;type:varchar(60);"` //用户名
    Password string `gorm:"clomun:password;type:varchar(32);"` // 密码
    Email    string `gorm:"clomun:email;type:varchar(100);"`  // 邮箱
}

```

中心存储池

```go
type Repository_Pool struct{
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
type User_Reposiroty struct{
    gorm.Model
    Identity string `gorm:"cloumn:identity;type:varchar(36);"`
    User_Identity string `gorm:"cloumn:user_identity;type:varchar(36);"` // 用户的唯一标识,知道是哪个人的文件
    Parent_ID int `gorm:"cloumn:parent_id;type:int;"` // 父级id 支持无限存储，可以建立n个文件夹
    Repository_Identity string `gorm:"cloumn:repository_identity;type:varchar(36);"` // 关联中心存储池的唯一标识
    Ext string `gorm:"cloumn:ext;type:varchar(255);"` // 文件或文件夹类型，如果为空，说明是一个文件夹，这个字段代表文件的扩展名
    Name string `gorm:"cloumn:name;type:varchar(255);"` // 当前中心存储池的名称
}
```

分享表

```go
type Share_Basic struct{
    gorm.Model
    Identity string `gorm:"clomun:identity;type:varchar(36);"`
    User_Identity string `gorm:"clomun:user_identity;type:varchar(36);"` // 谁分享的，分享者的唯一标识
    Repository_Identity string `gorm:"clomun:repository_identity;type:varchar(36);"` // 分享的是什么内容
    Expire_Time int `gorm:"clomun:expire_identity;type:int;"` // 分享内容的失效时间，单位秒，如果传0，永不失效
}
```
