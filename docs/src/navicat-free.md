# 脱离 Navicat 依赖

在后续的操作过程中, 有同学提出了一个问题: `我用的是 MacOS, 而且没有 Navicat, 那我怎么使用课程资料包呢？`

> 这个问题之前我确实没有考虑过, 后来在东北大学软件学院刘莹老师的指导下, 想到了一种解决方案:
>
> 1.  在可以进行移植的计算机上, 进行数据移植
> 2.  使用 mysql, 将移植后的数据, 再次导出
> 3.  分享 sql 脚本
>
> 理论来说, 这样就可以使数据脱离 Navicat 依赖了。

因此, 在本节, 将给出参照上述解决方案进行解决的流程, 如果需要复现, 自然可以, 当然也可以直接使用我制作的课程资料包的补丁包 `apple` 进行操作。

## 本节目录

- [脱离 Navicat 依赖](#脱离-navicat-依赖)
  - [本节目录](#本节目录)
  - [所需环境](#所需环境)
  - [问题描述](#问题描述)
  - [解决流程与坑](#解决流程与坑)
    - [字符集有问题! ?](#字符集有问题-)
    - [DDL 就不行](#ddl-就不行)
    - [适配 DDL](#适配-ddl)
    - [适配 all.sql](#适配-allsql)
  - [使用补丁包](#使用补丁包)
  - [补丁详情](#补丁详情)
    - [DDL_my.sql](#ddl_mysql)
    - [all.sql](#allsql)
  - [后续思考](#后续思考)

## 所需环境

若想要复刻我的操作, 请准备好如下环境:

- vscode (最新版即可, 下载链接: https://code.visualstudio.com/Download)
- mysql (兼容 8.0.35 版本的皆可, 我用的是 8.0.35)

## 问题描述

在使用命令行操作 mysql 后, 用 source 命令导入资料包给的 sql 脚本, 发现大量报错, 原因经排查为"字符集"问题。

## 解决流程与坑

### 字符集有问题! ?

看日志说是字符集问题, 那么, 问题来了, 它字符集为什么有问题呢？？

使用 vscode 打开 all.sql 脚本, 发现其编码格式用的是 utf8。而且, 这个 utf8 是我特意改的。

回顾之前的操作, 原始的课程资料包里面的 kingbase 导出的内容编码是 gbk, 这导致了 mysql 导入时出错, 也使得 `基于 Navicat 执行 sql 脚本` 时出错。

为了处理这个问题, 最后我用 ksql 执行了操作后, 再用 Navicat 将数据从金仓数据库导出, 导出时, 选择导出为 utf8 编码的 sql 文件, 这样解决了数据迁移的问题。也是因为这个, 使得编码格式变为了 utf8。

既然问题的原因难以确定, 我们不妨尝试分析下到底是什么时候产生了问题...

### DDL 就不行

首先, 在自己的操作系统里安装且配置好 mysql 的环境后, 使用如下语句进行连接:

```powershell
mysql -u root -p
# 然后, 先建立数据库的结构
# 注意下面命令, 要改成你的 DDL_my.sql 的路径
source DDL_my.sql的绝对路径
# 注意不要在结尾加 `;`
```

直接报错了！输出如下:

```plaintext
Query OK, 1 row affected (0.01 sec)

Database changed
Query OK, 0 rows affected (0.04 sec)

Query OK, 0 rows affected (0.03 sec)

Query OK, 0 rows affected (0.02 sec)

Query OK, 0 rows affected (0.03 sec)

Query OK, 0 rows affected (0.02 sec)

ERROR 1271 (HY000): Illegal mix of collations for operation ' IN '
ERROR 1267 (HY000): Illegal mix of collations (utf8mb4_0900_ai_ci,IMPLICIT) and (gbk_chinese_ci,COERCIBLE) for operation '='
Query OK, 0 rows affected (0.04 sec)

Query OK, 0 rows affected (0.04 sec)
```

看其报错信息, 感觉是字符集的问题, 主要是产生了冲突, 位于 `IN` 和 `=` 这两个**操作符**上！

> 话说这为啥啊？？？我不是统一过字符集嘛？？？

### 适配 DDL

没什么好办法, 先重新统一一下字符集试试...

- 当统一字符集为 utf8mb4 后, 发现报错依旧存在...
- 当统一字符集为 gbk 后, 报错变成了 warning

那就先这样吧！都用 gbk 吧！

这一步, 代码修改如下:

```plaintext
/**@Diff:
 * 1. 创建数据库时, 追加 `CHARACTER SET GBK`
 * 2. 创建每一个表时, 都追加 ` DEFAULT CHARSET=GBK`
 */
```

具体内容可以看本节最后一部分的[补丁详情](#补丁详情)。

### 适配 all.sql

在统一字符集为 gbk 后, 在 DDL_my.sql 运行完毕后, 再运行 all.sql, 发现报错依旧存在...

看报错, 内容出现了 "\x0" 这类的字符, 有充足的理由怀疑还是字符集的问题。

然后, 用 `vscode` 打开后, 将其编码格式修改为 `gbk`, 再次运行, 发现报错没了。

> PS: vscode 里修改编码的方法, 已放置到 [QA 文档](./qa.md#如何使用-vscode-调整文件编码)。

## 使用补丁包

本部分将会介绍, 拿到补丁包后, 如何使用。

> PS: 小常识, `补丁` 的英文是 `patch`
>
> `patch` 文件一般通过 `diff` 工具生成, 我比较钟爱 `git diff`
>
> 阅读 `patch` 文件时, 左侧的 `-` 代表被删去的行, 右侧的 `+` 代表新增的行。

```powershell
mysql -u root -p
# 然后, 先建立数据库的结构
# 注意下面命令, 要改成你的 DDL_my.sql 的路径 和 all.sql 路径
source DDL_my.sql的绝对路径
source all.sql的绝对路径
```

## 补丁详情

### DDL_my.sql

`DDL_my.sql` 变化情况如下:

```diff
diff --git a/DDL_my.sql b/DDL_my.sql
index b3a89ca..3b1055f 100755
--- a/DDL_my.sql
+++ b/DDL_my.sql
@@ -12,7 +12,7 @@
  */

 -- 创建 sales 数据库（在MySQL中, schema与database是同义词）
-CREATE DATABASE IF NOT EXISTS sales;
+CREATE DATABASE IF NOT EXISTS sales CHARACTER SET GBK;

 -- 切换到 sales 数据库（在MySQL中, 使用USE来切换数据库）
 USE sales;
@@ -25,7 +25,7 @@ CREATE TABLE Adminaddrs( /*行政区划地址 administration address*/
     parentid INTEGER REFERENCES Adminaddrs(addrid), /*父地址编码*/
     /*location  GEOMETRY, */                        /*经度纬度合一起的 GIS数据, 可能需要PostGis扩展插件支持, 可以用来计算就近仓库发货  */
     currlevel INTEGER                               /*当前层级 current level*/
-);
+) DEFAULT CHARSET=GBK;

 -- 创建 Categories 表
 CREATE TABLE Categories(/*商品分类*/
@@ -34,7 +34,7 @@ CREATE TABLE Categories(/*商品分类*/
     catgname VARCHAR(128),                          /*分类名称*/
     parentid INTEGER REFERENCES Categories(catgid), /*父类编码*/
     currlevel INTEGER                               /*当前层级 current level*/
-);
+) DEFAULT CHARSET=GBK;

 -- 创建 Shopstores 表
 /* Example: https://mall.jd.com/shopLevel-952805.html */
@@ -46,7 +46,7 @@ CREATE TABLE Shopstores(/*店铺*/
     delygrading DECIMAL(5,2),          /*物流评分 delivery grading*/
     servgrading DECIMAL(5,2),          /*服务评分service grading*/
     comprgrading DECIMAL(5,2)          /*综合评分 comprehensive grading*/
-);
+) DEFAULT CHARSET=GBK;

 -- 创建 Goods 表
 CREATE TABLE Goods(/*商品*/
@@ -66,7 +66,7 @@ CREATE TABLE Goods(/*商品*/
     size POLYGON,                                   /* 商品尺寸, 长 宽 高 */
     /*image  BINARY, */                             /* 商品图片 */
     instmanual TEXT                                 /* 使用说明  instruction manual */
-);
+) DEFAULT CHARSET=GBK;

 -- 创建 Supply 表
 CREATE TABLE Supply(/*店铺商品 供应*/
@@ -79,7 +79,7 @@ CREATE TABLE Supply(/*店铺商品 供应*/
     url VARCHAR(512),                               /* 商品网址 */
     homepage LONGTEXT,                              /* 商品网页转换而来的XML数据, 方便搜索, 使用LONGTEXT代替XML, 因为MySQL 8.0不再支持XML类型 */
     PRIMARY KEY(shopid, goodid)
-);
+) DEFAULT CHARSET=GBK;

 /*CREATE TYPE GENDERENUM AS ENUM(1, 2, 9); */ /*性别, 国家编码 1 男,  2 女,  9 未知*/

@@ -96,7 +96,7 @@ CREATE TABLE Customers(/*顾客*/
     mi VARCHAR(16) NULL,               /*月收入（是一个范围值）, monthly income, 使用DECIMAL代替INT8RANGE, 因为MySQL不支持INT8RANGE类型*/
     ebg CHAR(4) CHECK (ebg IN ('小学', '初中', '高中', '中专', '大专', '本科', '硕士', '博士', '其他')), /*教育程度,  有国家编码, 为简便起见, 直接使用汉字*/
     prof CHAR(50)                      /*所在行业 profession, 有国家编码, 为简便起见, 直接使用行业名称*/
-);
+) DEFAULT CHARSET=GBK;

 -- 创建 Comments 表
 CREATE TABLE Comments(/*商品评论*/
@@ -110,7 +110,7 @@ CREATE TABLE Comments(/*商品评论*/
     CHECK ((feeling = '好评' AND stars >= 4)
         OR (feeling = '中评' AND (stars = 3 OR stars = 2))
         OR (feeling = '差评' AND stars = 1))
-);
+) DEFAULT CHARSET=GBK;

 -- 创建 Orders 表
 CREATE TABLE Orders(/*订单*/
@@ -140,7 +140,7 @@ CREATE TABLE Orders(/*订单*/
     mot CHAR(4),                                        /*运输方式 Mode of Transportation, 航空, 铁路、公路、水路等, 运输方式和发货收货距离 决定 预计到达日期 */
     doe DATE,                                           /*预计到达日期 date of estimation*/
     doa DATE                                            /*送达日期 date of arrival */
-);
+) DEFAULT CHARSET=GBK;

 -- 创建 Lineitems 表
 CREATE TABLE Lineitems(/*订单明细*/
@@ -151,4 +151,4 @@ CREATE TABLE Lineitems(/*订单明细*/
     saleamt INTEGER,                                    /*销售数量*/
     PRIMARY KEY (ordid, shopid, goodid),
     FOREIGN KEY (shopid, goodid) REFERENCES Supply(shopid, goodid)
-);
+) DEFAULT CHARSET=GBK;
```

### all.sql

将编码格式由 `utf-8` 修改至 `gbk`。

## 后续思考

既然 mysql 能够允许 gbk 编码存在, 那是不是说明, 金仓数据库的 sql 导出内容, 可以通过某些逻辑, 快速适配移植到 mysql？

由于个人精力有限, 这个就交给各位大佬研究吧。
