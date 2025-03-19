---
title: 数据库实验--有无索引的性能比较
tags:
  - MySQL
createTime: 2025/03/17 19:16:25
permalink: /article/f0hspqnu/
---



在数据库学习中，为了直观展示索引带来的性能差异，可以通过以下实验，生成10G大小的数据库表，对比有无索引查找其中数据的时间差异。由于技术的进步，硬盘的读写性能飞涨，而《数据库系统概念》（**[Avi Silberschatz](http://www.cs.yale.edu/homes/avi)**）第七版给出的参考答案：100万倍，已经不再确切。

我们可以通过[此处](#_3-完整代码)的代码，测试在自己的硬件上的性能表现。

### 1. Python连接数据库

pymysql是一个用于Python编程的第三方模块，用于连接和操作MySQL数据库。它提供了一个简单而强大的接口，使开发者能够轻松地在Python程序中执行各种数据库操作，如查询、插入、更新和删除数据等。

可以通过如下语句安装pymysql：

```shell
pip install pymysql
```

然后就可以测试与数据库的连接：

```python
# 导入pymysql模块
import pymysql

# 建立数据库连接
conn = pymysql.connect(
    host='localhost',		# 主机名（或IP地址）
    port=3306,				# 端口号，默认为3306
    user='root',			# 用户名
    password='password',	# 密码
    charset='utf8mb4'  		# 设置字符编码
)

# 获取mysql服务信息（测试连接，会输出MySQL版本号）
print(conn.get_server_info())

# 创建游标对象
cursor = conn.cursor()

# 选择数据库
conn.select_db("mytable")

# 执行查询操作
cursor.execute('SELECT * FROM mytable')

# 获取查询结果，返回元组
result : tuple = cursor.fetchall()

# 关闭游标和连接
cursor.close()
conn.close()
```

### 2. 生成10G大小的数据表

首先，我们来预估一下，10GB需要多少条数据。

如果数据库表项的类型都是INT这样仅需几个字节的数据，假设每条记录4字节，那么需要：
$$
\frac{10\mathrm{GB}}{4\mathrm{B}}=\frac{10\times1024\times1024\times1024\mathrm{B}}{4\mathrm{B}}=\frac{10\times1024\times1024\times1024}{4}
$$
大约25亿条数据，这显然不行。

于是，我们考虑让每条记录，达到1MB的量级。

怎么办呢？可以考虑`LONGBLOB`这个类型的数据，它是Mysql一种用于存储二进制数据的类型，适合存储大型的二进制文件，例如图片、音频、视频等。

因此，我们可以建立以下数据表test1:

```sql
        CREATE TABLE test1 (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(255),
            large_blob LONGBLOB
        );
```

在向数据库中插入数据时，可以使用`os.urandom()`来生成随机二进制数据，插入到数据库中。

### 3. 完整代码

```python
import pymysql
import time
import os

def connect_to_mysql():
    return pymysql.connect(
        host='localhost',
        user='root',
        password='',#你的数据库密码
        charset='utf8mb4'
    )

def create_database(cursor):
    cursor.execute("DROP DATABASE IF EXISTS test2;")
    cursor.execute("CREATE DATABASE test2;")
    cursor.execute("USE test2;")

def create_table(cursor):
    cursor.execute("""
        CREATE TABLE test1 (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(255),
            large_blob LONGBLOB
        );
    """)

def insert_large_data(cursor, num_records=1000):
    print(f"Inserting {num_records} records...")
    for i in range(1, num_records + 1):
        large_content = os.urandom(1024 * 1024)  # Generate 1MB of random data
        cursor.execute("INSERT INTO test1 (name, large_blob) VALUES (%s, %s);", (f"Record {i}", large_content))
    print("Data insertion completed.")

def measure_query_time(cursor, with_index=False):
    if with_index:
        print("Creating index...")
        cursor.execute("CREATE INDEX idx_name ON test1 (name);")
        print("Index created.")
    
    start_time = time.time()
    cursor.execute("SELECT * FROM test1 WHERE name='Record 9999';")
    result = cursor.fetchall()
    end_time = time.time()
    
    print(f"Query time ({'with' if with_index else 'without'} index): {end_time - start_time:.4f} seconds")
    return result

def main():
    connection = connect_to_mysql()
    try:
        with connection.cursor() as cursor:
            # Step 1: Create database and table
            create_database(cursor)
            create_table(cursor)

            # Step 2: Insert large data
            insert_large_data(cursor, num_records=10000)
            connection.commit()
            # Step 3: Measure query time without index
            print("Measuring query time without index...")
            measure_query_time(cursor, with_index=False)
            
            # Step 4: Measure query time with index
            print("Measuring query time with index...")
            measure_query_time(cursor, with_index=True)
            connection.commit()
    finally:
        connection.close()

if __name__ == "__main__":
    main()

```

