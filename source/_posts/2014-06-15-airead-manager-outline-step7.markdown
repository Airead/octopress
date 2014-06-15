---
layout: post
title: "airead-manager-outline-step7"
date: 2014-06-15 21:52
comments: true
categories: airead-manager
---

添加权限限制

<!-- more -->

## 后端添加新加权限
在 `utils/permission.py` 中添加三行，增加 `hello` 权限。
```
RoleList = [
    ...
    'groupManager',
    'hello'  # add this line, hello role
]


class Roles(object):
    ...
    groupManager = RoleNeed('groupManager')
    hello = RoleNeed('hello')  # add this


class Permissions(object):
    ...
    groupManager = Permission(Roles.groupManager)
    hello = Permission(Roles.hello)  # add this

```

## 在 create_db.py 中初始化新权限
权限目前只能通过初始化获得，管理员不能添加新的权限。所以需要在 create_db.py 中初始化新权限。

```
def permission_init():
    permissions = [
        {'name': u'管理员', 'tag': 'admin'},
        {'name': u'游客', 'tag': 'guest'},
        {'name': u'数据库管理', 'tag': 'databaseManager'},
        {'name': u'数据库设置', 'tag': 'databaseSetting'},
        {'name': u'系统设置', 'tag': 'systemSetting'},
        {'name': u'节假日', 'tag': 'holidayManager'},
        {'name': u'用户管理', 'tag': 'userManager'},
        {'name': u'管理组设置', 'tag': 'groupManager'},
        {'name': u'world', 'tag': 'hello'},  # add this
    ]
```

然后运行 `python aireadManager/create_db.py` 初始化权限。

这时，应该能够在 `用户组管理` 的编辑界面看到 `world` 权限。

## 让删除接口受 world 权限限制
在 `blueprints/rest/persons.py` 添加权限支持，修改如下：

```
from aireadManager.utils.permissions import Permissions

...

    @Permissions.hello.require(403)  # add this line
    def delete(self, pid):
        db.session.query(PersonModel).filter_by(id=pid).delete()
        db.session.commit()

        return {'code': Code.SUCCESS}
```

现在我们再去删除 person, 将会收到 403 (FORBIDDEN) 错误了。

## 让前端不显示没有权限的菜单
- 添加 Roles 定义，修改 `static/coffee/utils.coffee` 如下：

```
define [], () ->
  Roles =
    admin: 'admin'
    guest: 'guest'
    databaseManager: 'databaseManager'
    databaseSetting: 'databaseSetting'
    systemSetting: 'systemSetting'
    holidayManager: 'holidayManager'
    userManager: 'userManager'
    groupManager: 'groupManager'
    hello: 'hello'  # add this line

  return {
    Roles: Roles
  }
```

- 为 `world.coffee` 加 role 需求

```
## 使用 utils.coffee
define ['./base', 'jQuery', 'static/js/utils'], (indexCtlModule, $, utils) ->

...

  ret =  # static/coffee/routes 跟据这个返回值注册前端路由
    group: '学习'  # 定义这个模块包含于哪个大组
    item: 'item Hello'  # item 名称
    url: moduleName  # 注册的 url
    role: utils.Roles.hello  # hello 角色需求, add this line
```

再次刷新页面，`item hello` 菜单应该已经隐藏了。

## 重现 item hello 菜单
为当前登陆用户的用户组添加 `world` 权限，就可以让 `item hello` 菜单重现，并拥有删除权限

`git checkout step7` 查看源码