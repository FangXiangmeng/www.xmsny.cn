---
layout: post
title: Harbor api使用
subtitle: ""
description: "Harbor api使用"
date: 2019-11-11=7T11:25:13+08:00
author: "FangXiangMeng"
tags:
  - harbor
---

## Harbor api

#### 项目管理

###### 查看仓库中的项目详细信息
```shell
$ curl -k -u "admin:Harbor12345" -X GET -H "Content-Type: application/json" "https://HostName/api/projects/{project_id}"

$ curl -k -u "admin:Harbor12345" -X GET -H "Content-Type: application/json" "https://HostName/api/projects?project_name=library"
```

###### 搜索镜像
```shell
$ curl  -u "admin:Harbor12345"  -X GET -H "Content-Type: application/json" "https://HostName/api/search?q=library"
```

###### 删除项目
```shell
$ curl  -u "admin:Harbor12345"  -X DELETE  -H "Content-Type: application/json" "https://HostName/api/projects/{project_id}"
```

###### 创建项目
```shell
$ curl -u "admin:Harbor12345" -X POST -H "Content-Type: application/json" "https://HostName/api/projects" -d @createproject.json
# cat  createproject.json
{
"project_name": "testrpo",
"public": 0
}
或者
$ curl -u "USER:PASSWORD" -X POST -H "Content-Type:application/json" "https://HostName/api/projects"  --insecure   -d '{"project_name": "testrpo","public": 0}'
```

###### 查看项目日志
```shell
$ curl -u "admin:Harbor12345" -X POST -H "Content-Type: application/json" "https://HostName/api/projects/{project_id}/logs/filter" -d @log.json
# cat log.json
{
  "username": "admin"
}
```

### 账号管理
###### 创建账号
```shell
$ curl -u "admin:Harbor12345" -X POST -H "Content-Type: application/json" "https://HostName/api/users" -d @user.json
$ cat  user.json

{

  "user_id": 5,

  "username": "zhangsan",

  "email": "zhangsan@gmail.com",

  "password": "Zs202005",

  "realname": "zhangsan",

  "role_id": 2

}

```

###### 获取用户信息
```shell
$ curl -u "admin:Harbor12345" -X GET -H "Content-Type: 
application/json" "https://HostName/api/users"
```

###### 获取当前用户信息
```shell
$ curl -u "admin:Harbor12345" -X GET -H "Content-Type: application/json" "https://HostName/api/users/current"
```

###### 删除用户
```shell
$ curl -u "admin:Harbor12345" -X DELETE  -H "Content-Type: application/json" "https://HostName/api/users/{user_id}"
```

###### 修改用户密码
```shell
$ curl -u "admin:Harbor12345" -X PUT -H "Content-Type: application/json" "https://HostName/api/users/{user_id}/password" -d @uppwd.json

$ cat uppwd.json

{

  "old_password": "Harbor123456",

  "new_password": "Harbor888888"

}
```

### 用户权限管理

###### 查看项目相关角色
```shell
$ curl -u "admin:Harbor12345" -X GET -H "Content-Type: application/json" "https://HostName/api/projects/{project_id}/members/"
```

###### 项目添加角色
```shell
$ curl -u "jaymarco:Harbor123456" -X POST  -H "Content-Type: application/json" "https://HostName/api/projects/{project_id}/members/" -d @role.json
$ cat role.json

{

  "roles": [

  ],

  "username": "guest"

}
```

###### 删除项目中用户权限
```shell
$ curl -u "admin:Harbor12345" -X DELETE -H "Content-Type: application/json" "https:/HostName6/api/projects/{project_id}/members/{user_id}"
```

###### 获取与用户相关的项目编号和存储库编号
```shell
$ curl -u "admin:Harbor12345" -X GET -H "Content-Type: application/json" "https://HostName/api/statistics"
```

###### 修改当前用户角色
- has_admin_role ：0 普通用户
- has_admin_role ：1 管理员
```shell
$ curl -u "admin:Harbor12345" -X PUT -H "Content-Type: application/json" "https://HostName/api/users/{user_id}/sysadmin " -d @chgrole.json
$ cat >chgrole.json

{

  "has_admin_role": 1

}
```

### 镜像管理
###### 查询镜像
```shell
$ curl -u "admin:Harbor12345" -X GET -H "Content-Type: application/json" "https://HostName/api/repositories?project_id={project_id}&q=dcos%2Fcentos"
```

###### 删除镜像
```shell
$ curl -u "admin:Harbor12345" -X DELETE -H "Content-Type: application/json" "https://HostName/api/repositories?repo_name=dcos%2Fetcd "
```

###### 获取镜像标签
```shell
$ curl -u "admin:Harbor12345" -X GET -H "Content-Type: application/json" "https://HostName/api/repositories/tags?repo_name=dcos%2Fcentos"
```
