# Oracle 学习笔记

# 安装部署

摘抄自：[Docker安装Oracle_11g](https://www.jb51.net/article/208976.htm) （[原文地址](https://blog.csdn.net/qq_42021376/article/details/115444547?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522167766259016800227481245%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=167766259016800227481245&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-115444547-null-null.blog_rank_default&utm_term=oracle&spm=1018.2226.3001.4450)）

**1.拉取oracle_11g镜像**

```
docker pull registry.cn-hangzhou.aliyuncs.com``/helowin/oracle_11g
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420617.png)

**2.创建oracle11g容器**

```
docker run -d -p 1521:1521 --name oracle11g registry.cn-hangzhou.aliyuncs.com``/helowin/oracle_11g
```

**3.查看oracle11g容器是否创建成功**

```
docker ``ps` `-a
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420618.png)

**4.启动oracle11g容器**

```
docker start oracle11g
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420619.png)

**5.进入oracle11g容器进行配置**

```
docker ``exec` `-it oracle11g ``bash
```

**6.切换到root用户下进行配置**

```
su` `root
```

密码为：`helowin`

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420720.png)

**7、编辑profile文件配置ORACLE环境变量**

```
vi` `/etc/profile
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420721.png)

**8、最后添加以下3行配置**

```
export` `ORACLE_HOME=``/home/oracle/app/oracle/product/11``.2.0``/dbhome_2``export` `ORACLE_SID=helowin``export` `PATH=$ORACLE_HOME``/bin``:$PATH
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420722.png)

保存 ：`:wq`
让配置生效：`source /etc/profile`

**9、创建软连接**

```
ln` `-s $ORACLE_HOME``/bin/sqlplus` `/usr/bin
```

**10、切换到oracle 用户**

```
su` `- oracle
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420723.png)

**11、登录sqlplus并修改sys、system用户密码**

```
sqlplus ``/nolog
conn ``/as` `sysdba
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420724.png)

**12、修改sys、system用户密码并刷新权限**

```
alter` `user` `system identified ``by` `oracle;``alter` `user` `sys identified ``by` `oracle;``ALTER` `PROFILE ``DEFAULT` `LIMIT PASSWORD_LIFE_TIME UNLIMITED;
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420725.png)

退出：`exit;`
**13、查看一下oracle实例状态**

```
lsnrctl status
```

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420726.png)

**14、用nacivat连接oracle数据库**

服务名：`helowin`（一定要填写helowin）
密码：oracle（第12步设置的密码）

![在这里插入图片描述](https://img.jbzj.com/file_images/article/202104/2021040708420727.png)