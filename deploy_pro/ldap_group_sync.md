# 同步 AD 群组

在 4.1.0 版本之后，专业版开始支持从 LDAP 或者 AD 导入(同步)群组到 seafile。

## 工作原理

导入或同步的过程是从 LDAP 服务器上的组映射到 seafile 的内部数据库中的组。这个过程是单向的。

* 数据库中对组的任何改变都不会回传到 LDAP；
* 除了“设置成员为组管理员” 之外，数据库中对组的任何更改将被下一个LDAP同步操作覆盖。如果要添加或删除成员则只能在LDAP服务器上执行此操作。
* 导入组的创建者将会被设置为系统管理员。

一些LDAP服务器(如ad)允许将组设置为另一个组的成员。这被称为“嵌套”。我们的程序支持同步嵌套组。假设B组是A组成员，结果将是：B组的每个成员均为A组和B组的成员。

有两种运作模式：

* 周期：同步过程将在固定的时间间隔内执行；
* 手动：可以通过执行一个脚本来触发同步过程；

## 前提条件

您已经在系统中安装了 python-ldap 库。

在 Debian 或 Ubuntu 下：

```
sudo apt-get install python-ldap
```

在 CentOS 或 RedHat 下

```
sudo yum install python-ldap
```

## 配置

在启用 LDAP 组同步之前，您应该已经配置好 LDAP 身份验证。有关详细信息请参考[Seafile 企业版 LDAP 和 Active Directory 配置](using_ldap.md)。

以下是 LDAP 组同步想相关选项。它们定义在 [ccnet.conf](../config/ccent-conf.md) 的“[LDAP_SYNC]”配置段中。

* **ENABLE_GROUP_SYNC**：如果要启用 LDAP 组同步请设置为 “true”。
* **SYNC_INTERVAL**：同步周期，单位是分钟，默认设置为60分钟。
* **GROUP_OBJECT_CLASS**：这是用于搜素组对象的类的名称。在 Active directory 中，它通常是"group";在OpenLDAP或其他中，可以使用"groupOfNames","groupOfUniqueNames" 或者 "posixGroup",这取决于你使用的LDAP服务器。默认设置为"group"。
* **GROUP_FILTER**：在搜素组对象时使用的附加筛选器。如果设置了，最终用于搜素的筛选器是"(&(objectClass=GROUP_OBJECT_CLASS)(GROUP_FILTER))"，否则使用的筛选器将是"(objectClass=GROUP_OBJECT_CLASS)"。
* **GROUP_MEMBER_ATTR**：在加载组的成员时使用的属性字段。对于大多数directory服务器，属性是“member”，这也是默认值。对于"posixGroup"，它应该被设置为"memberUid"。
* **USER_ATTR_IN_MEMBERUID**：“memberuid”选项中的用户属性集,用于“posixgroup”。默认值为“uid”。

组的搜索基础是 `ccnet.conf` 中设置在"[LDAP]"配置段的"BASE_DN"。

这有一个关于 Active Directory 的配置示例：

```
[LDAP]
HOST = ldap://192.168.1.123/
BASE = cn=users,dc=example,dc=com
USER_DN = administrator@example.local
PASSWORD = secret
LOGIN_ATTR = mail

[LDAP_SYNC]
ENABLE_GROUP_SYNC = true
SYNC_INTERVAL = 60
```

对于AD，除了"ENABLE_GROUP_SYNC"之外，通常不需要配置其他选项。因为其他选项的默认值是AD的常用值。如果LDAP服务器中有特殊设置，则只设置相应的选项。

这有一个关于 OpenLDAP 的配置示例：

```
[LDAP]
HOST = ldap://192.168.1.123/
BASE = ou=users,dc=example,dc=com
USER_DN = cn=admin,dc=example,dc=com
PASSWORD = secret
LOGIN_ATTR = mail

[LDAP_SYNC]
ENABLE_GROUP_SYNC = true
SYNC_INTERVAL = 60
GROUP_OBJECT_CLASS = groupOfNames
```

**注意** 在您重启seafile服务器后，不会立即同步，它在第一次同步周期后进行同步。例如，如果将同步周期设置为30分钟，则第一次自动同步将在您重启后的30分钟后发生。要立即同步，您需要手动触发。下一节将介绍这一情况。

运行同步后，您应该在日志 `logs/seafevents` 中看到如下所示的日志信息。并且在系统管理页面中应该能看到那些组。

```
[2015-03-30 18:15:05,109] [DEBUG] create group 1, and add dn pair CN=DnsUpdateProxy,CN=Users,DC=Seafile,DC=local<->1 success.
[2015-03-30 18:15:05,145] [DEBUG] create group 2, and add dn pair CN=Domain Computers,CN=Users,DC=Seafile,DC=local<->2 success.
[2015-03-30 18:15:05,154] [DEBUG] create group 3, and add dn pair CN=Domain Users,CN=Users,DC=Seafile,DC=local<->3 success.
[2015-03-30 18:15:05,164] [DEBUG] create group 4, and add dn pair CN=Domain Admins,CN=Users,DC=Seafile,DC=local<->4 success.
[2015-03-30 18:15:05,176] [DEBUG] create group 5, and add dn pair CN=RAS and IAS Servers,CN=Users,DC=Seafile,DC=local<->5 success.
[2015-03-30 18:15:05,186] [DEBUG] create group 6, and add dn pair CN=Enterprise Admins,CN=Users,DC=Seafile,DC=local<->6 success.
[2015-03-30 18:15:05,197] [DEBUG] create group 7, and add dn pair CN=dev,CN=Users,DC=Seafile,DC=local<->7 success.
```

## 手动触发同步

手动触发 LDAP 组同步。

```
cd seafile-server-lastest
./pro/pro.py ldapsync
```



