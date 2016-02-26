# 附录一：POC实施前准备


##第一章 OS & Linux

1、检查操作系统及其版本

查看方法：
```
cat etc/issue
```

2、检查hostname,FQDN,DNS

查看方法：
```
hostname

hostname-f

cat /etc/resolv.conf
```

3、更改方法：


```
vim /etc/sysconfig/network

vim /etc/hosts

□ hostname只能是以字母和数字的组合(中间允许’-‘)，不能有“,” / ”.” / “_”等特殊字符


```


    

