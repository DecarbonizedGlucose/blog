+++
date = '2025-08-02T15:46:23+08:00'
draft = false
title = 'SQL和数据库管理系统（MySQL）'

+++

## 为什么需要数据库？
数据太多，文件搞不定。
- 查询复杂
- 数据结构混乱难以维护
- 文件难以并发处理

数据库，
- 结构化数据管理
- 高效查询
- 并发控制
- 数据安全
## 数据库和SQL

数据库 ：

- 关系型数据库
- 非关系型数据库

SQL是一种管理关系型数据库的语言。

- SQL (Structured Query Language)：结构化查询语言
- 用于操作关系型数据库：定义数据表，插入/查询/修改/删除数据
## MySQL？SQLite？PostgreSQL？

Database Management System. 是一种软件，不是数据库。

- MySQL         ->          主流服务器端
- SQLite          ->          嵌入式
- PostgreSQL ->          支持复杂查询

## MySQL安装

Ubuntu

~~~shell
sudo apt install mysql-common mysql-client mysql-server
sudo systemctl start mysql-server
sudo systemctl enable mysql-server # 开机自启
~~~

Arch

~~~shell
sudo pacman -S mariadb
sudo mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
sudo systemctl start mariadb
sudo systemctl enable mariadb
~~~
## 开始使用（Ubuntu）

~~~shell
sudo mysql
~~~
~~~mysql
CREATE USER 'new_user'@'host_name' IDENTIFIED BY 'password115414';
~~~

`host_name`=`"localhost"` 只能从本地访问

`host_name`=`"%"` 任何地方

~~~mysql
GRANT ALL PRIVILEGES ON database.table TO 'new_user'@'host_name' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
~~~
- ALL PRIVILEGES
- SELECT  读/查
- INSERT  插入/写
- UPDATE  更新数据
- DELETE  删除数据
- CREATE  创建表
- DROP  删除表
- ALTER  修改表结构
- INDEX  创建/删除索引
- GRANT OPTION  授予其他用户权限
~~~mysql
REVOKE (权限) ON database.table FROM 'user_name'@'host_name';
~~~
最小权原则
~~~shell
mysql -u new_user -p
~~~
## SQL语言要点
细分,
- **DDL（数据定义语言）**
  - `CREATE TABLE` / `DROP TABLE`
  - `ALTER TABLE` 增加/修改列
- **DML（数据操作语言）**
  - `INSERT` / `UPDATE` / `DELETE` (增删改)
- **DQL（数据查询语言）**
  - `SELECT ... FROM ... WHERE ...` (查)
- **DCL（数据控制语言）**
  - `GRANT` / `REVOKE` (权限控制)
## MySQL数据库

~~~mysql
CREATE DATABASE database_name
SHOW DATABASES;
USE database_name;
DROP DATABASE [IF EXISTS] database_name;
~~~
## MySQL的数据类型

### 整数类

| 类型               | 占用空间 | 范围（有符号）   | 说明           |
| ------------------ | -------- | ---------------- | -------------- |
| `TINYINT`          | 1字节    | -128 ~ 127       | 最小的整数     |
| `SMALLINT`         | 2字节    | -32,768 ~ 32,767 | 小整数         |
| `MEDIUMINT`        | 3字节    | -8百万 ~ 8百万   | 不常用         |
| `INT` 或 `INTEGER` | 4字节    | -21亿 ~ 21亿     | 常用           |
| `BIGINT`           | 8字节    | 极大整数         | 适合ID、金额等 |
### 浮点类
| 类型           | 用途     | 精度           | 说明                             |
| -------------- | -------- | -------------- | -------------------------------- |
| `FLOAT(M,D)`   | 小数     | 约7位有效数字  | 近似存储，速度快                 |
| `DOUBLE(M,D)`  | 精确小数 | 约15位有效数字 | 用于科学计算                     |
| `DECIMAL(M,D)` | 精确小数 | 任意精度       | **金融计算首选**，不会有精度丢失 |

M是总位数，D是小数位。

比如，DECIMAL(10, 2) max=?
### 字符串类型

| 类型         | 最大长度                    | 特点               |
| ------------ | --------------------------- | ------------------ |
| `CHAR(n)`    | 最多255                     | 固定长度           |
| `VARCHAR(n)` | 最多65535（受限于表总行宽） | 变长字符串，最常用 |

这里n指的是最大长度，不是一定要存的长度
### 文本类型

| 类型         | 最大长度 | 用途                       |
| ------------ | -------- | -------------------------- |
| `TINYTEXT`   | 255字节  | 短文本                     |
| `TEXT`       | 64KB     | 常用文本                   |
| `MEDIUMTEXT` | 16MB     | 较大文本                   |
| `LONGTEXT`   | 4GB      | 超大文本，如文章内容、日志 |

不能设置默认值，不能索引全文。
### 二进制类型
- **`BINARY(n)`**
- **`VARBINARY(n)`**
    用于存储短数据（密码等）
- **`BLOB`**
    用于图片/二进制文件
### 日期时间

| 类型        | 格式                  | 说明              |
| ----------- | --------------------- | ----------------- |
| `DATE`      | `YYYY-MM-DD`          | 日期              |
| `DATETIME`  | `YYYY-MM-DD HH:MM:SS` | 最常用            |
| `TIMESTAMP` | 同上                  | 会随时区变动      |
| `TIME`      | `HH:MM:SS`            | 时间段            |
| `YEAR`      | `YYYY`                | 年份（1901-2155） |
### 枚举和集合
- **`ENUM(a, b, c)`**  多选一
- **`SET(a, b, c)`**  多值，相当于子集
## 关键字
- **`PRIMARY KEY`**: 唯一标识一条数据（一定NOT NULL，UNIQUE）
~~~mysql
CREATE TABLE users (
    user_id VARCHAR(30) PRIMARY KEY,
    ...
);
~~~
- **`INDEX`**: 索引
~~~mysql
CREATE TABLE users (
	...
	username VARCHAR(50),
    INDEX(username),
	...
);
~~~
~~~mysql
CREATE INDEX idx_username ON users(username);
DROP INDEX idx_username ON users;
~~~
~~~mysql
SHOW INDEX FROM users;
~~~
- **`NULL`**
~~~mysql
CREATE TABLE users (
    ...
    user_email VARCHAR(255) NOT NULL UNIQUE,
    ...
);
~~~
可能是空，查询时判断必须用`IS NULL`/`IS NOT NULL`
~~~mysql
SELECT * FROM users WHERE email IS NULL;
~~~
- **`FOREIGN KEY, REFERENCES`**
~~~mysql
CREATE TABLE friends (
    user_id VARCHAR(30) NOT NULL,
    ...
    FOREIGN KEY(user_id) REFERENCES users(user_id),
    ...
);
~~~
- **`ON DELETE CASCADE, ON UPDATE CASCADE`**
~~~mysql
user_id VARCHAR(30) NOT NULL,
...
FOREIGN KEY(user_id) REFERENCES users(user_id)
    ON UPDATE CASCADE
    ON DELETE CASCADE
    ...
~~~
- **`CHECK`** (MySQL >= 8.0)
~~~mysql
age INT CHECK (age >= 0)
~~~
- **`DEFAULT`**
~~~mysql
is_active BOOLEAN DEFAULT TRUE
                / DEFAULT 1
~~~
- **`AUTO_INCREMENT`**
~~~mysql
message_id BIGINT PRIMARY KEY AUTO_INCREMENT
~~~
- **组合主键**
~~~mysql
PRIMARY KEY(group_id, user_id)
~~~
## 查
语法：
~~~mysql
SELECT 列[, 列2[, 列3[...]]] FROM 表 [WHERE 条件]
~~~
| 操作       | 示例                                                         | 说明            |
| ---------- | ------------------------------------------------------------ | --------------- |
| 查询所有列 | `SELECT * FROM users;`                                       | 选出整张表      |
| 查询指定列 | `SELECT nickname FROM users;`                                | 只查 `nickname` |
| 条件查询   | `SELECT * FROM users WHERE age > 18;`                        | 加筛选条件      |
| 模糊匹配   | `SELECT * FROM users WHERE username LIKE 'a%';`              | 查以 a 开头的   |
| 去重       | `SELECT DISTINCT gender FROM users;`                         | 去重取性别种类  |
| 排序       | `SELECT * FROM users ORDER BY created_at DESC;`              | 最近的排最前    |
| 限制       | `SELECT * FROM users LIMIT 10 OFFSET 20;`                    | 分页：第21-30条 |
| 统计       | `SELECT COUNT(*) FROM users;`                                | 总人数          |
| 分组       | `SELECT gender, COUNT(*) FROM users GROUP BY gender;`        | 按性别计数      |
| 联表       | `SELECT u.username, m.content FROM users u JOIN messages m ON u.user_id = m.sender_id;` | 查询聊天记录    |
### 模糊匹配
`%`任意数量的字符
`-`恰好一个字符
~~~mysql
SELECT * FROM users WHERE username LIKE "A%"; # 以A开头
SELECT * FROM users WHERE username LIKE "%114%"; # 包含这串数字
SELECT * FROM users WHERE username LIKE "_c%"; # 第二个字符是'c'
~~~
此操作默认区分大小写。
有以下工具：
`LOWER()` `UPPER()` `TRIM()`(去除空格)
若要模糊大小写：
~~~mysql
SELECT * FROM users WHERE LOWER(username) LIKE LOWER('A%');
~~~
## 增删改
插入`INSERT`语法：
~~~mysql
INSERT INTO 表 主键
VALUES (...), (...), (...); # 可以一次插入多个
~~~
更新`UPDATE`语法：
~~~mysql
UPDATE 表 SET 字段（列）= 值 WHERE 条件
~~~
条件可以选取一行(WHERE id='u001')
或者批量选择(WHERE time > '2025-05-14')
删除`DELETE`语法：
~~~mysql
DELETE FROM 表 WHERE 条件;
~~~
如果没用上条件
~~~mysql
DELETE FROM 表; # 会删除表内所有数据！
~~~
