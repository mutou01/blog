1.注意错误处理；
```go
// 推荐写法: 错误判断与赋值同时进行

if err := c.ShouldBindJSON(&req); err != nil {

    ci.Error(c, 40001, "参数错误")

    return

}

  

if err := account.GetByUsername(c, req.Username); err != nil {

    ci.Error(c, 40002, "账号不存在")

    return

}

  

// 不推荐: 分离赋值和判断

err := c.ShouldBindJSON(&req)

if err != nil {

    ci.Error(c, 40001, "参数错误")

    return

}
```

2.模型结构
```go

package Course

  

import (

    "github.com/gin-gonic/gin"

    "github.com/qinuoyun/caleyi/utils/ci"

    "github.com/qinuoyun/agent-server/utils"

)

  

// Course 课程模型

type Course struct {

    ci.Model  // 嵌入基础模型

    Title       string  `gorm:"type:varchar(255);not null;comment:课程标题" json:"title"`

    Description string  `gorm:"type:text;comment:课程描述" json:"description"`

    Price       float64 `gorm:"type:decimal(10,2);default:0.00;comment:课程价格" json:"price"`

    Status      int     `gorm:"type:tinyint;default:1;comment:状态 1-正常 2-下架" json:"status"`

}

  

// init 初始化模型

func init() {

    path := Course{}

    ci.BinModule(&path)

}

  

// TableName 表名定义

func (Course) TableName() string {

    return utils.GetTableName(Course{})

}

```

3.参数绑定与校验
```go

// 正确写法

var req LoginRequest

if err := c.ShouldBindJSON(&req); err != nil {

    ci.Error(c, 40001, "参数错误")

    return

}

  

// 错误写法 - 禁止使用

username := c.PostForm("username")  // 禁止

password := c.Query("password")    // 禁止

```

4.去除build文件

5.Controller层拆分，实现各个方法名为单个单词

6.Service层文件夹直属子文件夹名首字母为大写，且该文件夹下必须有service.go，且其他文件必须为小写单个单词，但允许子文件夹嵌套后不包含service.go
![[Pasted image 20260320180516.png]]

7.Modules层文件夹直属子文件夹名首字母为大写，且该文件夹下文件名=数据库表名，sql只能在这个层文件中
![[Pasted image 20260320180617.png]]