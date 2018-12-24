---
title: 容器化搭建Go-Web应用
date: 2018-12-23 12:13:00
tags:
---

## 改用Mysql数据库

本次作业要求改`BoltDB`为`Mysql`数据库，同时，考虑到后面的作业要求搭建博客。因此将之前写的新闻应用做了重构，改成了一个博客后台。

使用了`Beggo`框架自带的`ORM`。个人感觉这类框架内部封装好了很多逻辑，实现功能会比`boltdb`的键值对操作要容易很多。下面以用户类别为例分别对增删改查四个操作作出解释。

`User`的模型类别如下，我们可以像下面这样指定对应数据库的列要求。如主键`pk`，自增`auto`，可以为空null与不可重复`uniqu`e等。

```go
type User struct {
  Id        int       `orm:"pk;auto"`
  Username  string    `orm:"unique"`
  Password  string
  Token     string    `orm:"unique"`
  Avatar    string
  Email     string    `orm:"null"`
  Url       string    `orm:"null"`
  Signature string    `orm:"null;size(1000)"`
  InTime    time.Time `orm:"auto_now_add;type(datetime)"`
  Roles     []*Role   `orm:"rel(m2m)"`
}
```

+ 增加操作

  在用户注册页面，当我们根据用户的输入构建好了`user`类后，就可以通过如下函数进行用户的创建，注册。

  `orm`是单例模式，如下一行简单的`insert`就做到了。

  ```go
  func SaveUser(user *User) int64 {
    o := orm.NewOrm()
    id, _ := o.Insert(user)
    return id
  }
  
  ```

+ 删除操作

  实际博客应用中很少删除用户，为了演示使用下面的函数。同样的传入`user`类后，一行简单的`delete`就做到了。

  ```go
  func DeleteUser(user *User) {
    o := orm.NewOrm()
    o.Delete(user)
  }
  
  ```

+ 修改操作

  如上，因为指定了`id`为主键，所以`update`的时候会查找主键相同的，再进行修改

  ```go
  func UpdateUser(user *User) {
    o := orm.NewOrm()
    o.Update(user)
  }
  ```

+ 查找操作

  查找的语句较麻烦，我们需要指定一个数据表，指明过滤的字段和条件，并将`user`类型指针传入以获得查找到的结果。

```go
func Login(username string) (bool, User) {
  o := orm.NewOrm()
  var user User
  err := o.QueryTable(user).Filter("Username", username).One(&user)
  return err != orm.ErrNoRows, user
}
```

除了上述的操作外，我们还可以用数据库原生语句来执行命令。如下以增删为例。

```go
func DeleteUserRolesByUserId(user_id int) {
  o := orm.NewOrm()
  o.Raw("delete from user_roles where user_id = ?", user_id).Exec()
}

func SaveUserRole(user_id int, role_id int) {
  o := orm.NewOrm()
  o.Raw("insert into user_roles (user_id, role_id) values (?, ?)", user_id, role_id).Exec()
}

```

我们构建好了`model`后，就可以注册数据库了。

如下我们需要指明数据库的地址，用户名，密码数据库名，通过` orm.RegisterDataBase`注册数据库，通过`orm.RegisterDriver`注册`mysql`驱动。之后我们还需要依次注册各个`model`，并同步数据库` orm.RunSyncdb`。

```go
func init() {
    orm.RegisterDriver("mysql", orm.DR_MySQL)
  orm.RegisterDataBase("default", "mysql", username+":"+password+"@tcp("+url+":"+port+")/pybbs?charset=utf8&parseTime=true&charset=utf8&loc=Asia%2FShanghai", 30)
  orm.RegisterModel(
    new(models.User),
    new(models.Topic),
    new(models.Section),
    new(models.Reply),
    new(models.ReplyUpLog),
    new(models.Role),
    new(models.Permission))
  orm.RunSyncdb("default", false, true)
}

```

![](https://ws2.sinaimg.cn/large/006tNbRwgy1fyi9zyqm2yj31bu0k84a2.jpg)

## Docker构建容器

+ `mysql`镜像

  我们在安装了`docker`后，首先将`mysql`镜像下载下来。

  ```shell
  docker pull mysql:5.6
  ```

  接着我们可以查看到已有的镜像。

  ```shell
  docker images |grep mysql
  ```

  然后开启`mysql`容器，并指定各类配置。

  ```shell
  docker run -p 3306:3306 --name mymysql -v $PWD/conf:/etc/mysql/conf.d -v $PWD/logs:/logs -v $PWD/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.6
  ```

  - **-p 3306:3306**：将容器的 3306 端口映射到主机的 3306 端口。
  - **-v ~/conf:/etc/mysql/conf.d**：将主机当前目录下的 conf/my.cnf 挂载到容器的 /etc/mysql/my.cnf。
  - **-v ~/logs:/logs**：将主机当前目录下的 logs 目录挂载到容器的 /logs。
  - **-v ~/data:/var/lib/mysql** ：将主机当前目录下的data目录挂载到容器的 /var/lib/mysql 。
  - **-e MYSQL_ROOT_PASSWORD=123456** : 初始化 root 用户的密码。

+ `node`镜像

  接下来开启前端镜像，以下我们使用dockerfiles，进入项目目录。

  ```dockerfile
  #安装node镜像
  FROM node:7.8.0
  
  # 创建 app 目录
  WORKDIR /app
  
  # 安装 app 依赖
  
  RUN npm install
  
  #暴露端口
  EXPOSE 8080
  #运行命令
  CMD [ "npm", "run" ,"dev"]
  
  ```

  然后我们创建`docker`镜像

  ```shell
  docker build -t web_front .
  ```

  并运行

  ```shell
  docker run -p 8080:8080 -d web_front
  ```

  ![](https://ws1.sinaimg.cn/large/006tNbRwgy1fygsdr5ec1j30q20cqae0.jpg)

+ go镜像

  接下来我们同样进入`web`后端目录，创建`dockerfile`。

```dockerfile
#安装go镜像
FROM golang:latest
# 创建 应用 目录
WORKDIR $GOPATH/src/web_pro
ADD . $GOPATH/src/web_pro
#安装依赖
RUN go get github.com/astaxie/beego
RUN go get github.com/beego/bee
RUN go build .
#暴露端口
EXPOSE 8080
#运行命令
CMD [ "bee", "run" ]

```

然后我们创建`docker`镜像，并运行

```sh
docker build -t web_pro .
```

```sh
docker run -p 8080:8080 -d web_pro
```

