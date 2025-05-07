---
title: sql和sqlx使用
author: cotes
date: 2024-06-13 11:33:00 +0800
categories: [学习]
tags: [学习]
pin: true
math: true
mermaid: true
---

### 初始化 MySql

```
var Db *sqlx.DB
func InitMySQL() (err error) {
	dsn := "root:root@tcp(127.0.0.1:3306)/community?charset=utf8mb4&parseTime=true"
	Db, err = sqlx.Connect("mysql", dsn)
	if err != nil {
		panic(err)
	}
	err = Db.Ping()
	if err != nil {
		return
	}
	fmt.Println("connect to database successfully!!!")
	Db.SetMaxOpenConns(200)                 //最大连接数
	Db.SetConnMaxLifetime(time.Second * 10) //连接池里面的连接最大空闲时长
	Db.SetMaxIdleConns(10)                  //最大空闲连接数
	return
}
```

## sql

```
type user struct {
	id   int
	name string
	age  int
}
```

- 单条数据查询
  - ```QueryRowCommunity
    func QueryRowCommunity() {
      	sqlStr := "select id, name, age from user where id = ?"
      	var u user
      	err := Db.QueryRow(sqlStr, 4).Scan(&u.id, &u.name, &u.age)
      	if err != nil {
      		fmt.Printf("scan failed, err%v\n", err)
      		return
      	}
      	fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
      }
    ```
- 多条数据查询
  - ```QueryMultiRowCommunity
    func QueryMultiRowCommunity() {
      	sqlStr := "select id, name, age from user where id > ?"
      	row, err := Db.Query(sqlStr, 0)
      	if err != nil {
      		fmt.Printf("query failed, err:%v\n", err)
      		return
      	}
      	defer row.Close()
      	for row.Next() {
      		var u user
      		err := row.Scan(&u.id, &u.name, &u.age)
      		if err != nil {
      			fmt.Printf("scan failed, err:%v\n", err)
      			return
      		}
      		fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
      	}
      }
    ```
- 预处理查询
  - ```PrepareQuery
    func PrepareQuery() {
      	sqlStr := "select id, name, age from user where id>?"
      	stmt, err := Db.Prepare(sqlStr)
      	if err != nil {
      		fmt.Printf("prepare failed, err: %v\n", err)
      		return
      	}
      	defer stmt.Close()
      	rows, err := stmt.Query(0)
      	if err != nil {
      		fmt.Printf("Query failed, err:%v\n", err)
      		return
      	}
      	defer rows.Close()
      	//循环读取
      	for rows.Next() {
      		var u user
      		err := rows.Scan(&u.id, &u.name, &u.age)
      		if err != nil {
      			fmt.Printf("Scan failed, err %v\n", err)
      			return
      		}
      		fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
      	}
      }
    ```
- 插入数据
  - ```InsertRowCommunity
    func InsertRowCommunity() {
      	sqlStr := "insert into user(name, age) values (?, ?)"
      	ret, err := Db.Exec(sqlStr, "爱丽", 13)
      	if err != nil {
      		fmt.Printf("insert failed, err: %v", err)
      		return
      	}
      	var theID int64
      	theID, err = ret.LastInsertId()
      	if err != nil {
      		fmt.Printf("get LastInsertId failed, err%v\n", err)
      		return
      	}
      	fmt.Printf("insert success, the id is %d\n", theID)
      }
    ```
- 预插入数据
  - ```PrepareInsert
    func PrepareInsert() {
      	sqlStr := "insert into user(name, age) values (?, ?)"
      	ist, err := Db.Prepare(sqlStr)
      	if err != nil {
      		fmt.Printf("prepare failed, err: %v\n", err)
      		return
      	}
      	defer ist.Close()
      	_, err = ist.Exec("mary", 14)
      	if err != nil {
      		fmt.Printf("insert failed err%v\n", err)
      		return
      	}
      }
    ```
- 更新数据
  - ```UpdateRowCommunity
    func UpdateRowCommunity() {
      	sqlStr := "update user set age=?,name=? where id=?"
      	ret, err := Db.Exec(sqlStr, 11, "mafula", 2)
      	if err != nil {
      		fmt.Printf("update failed, err:%v\n", err)
      		return
      	}
      	var num int64
      	num, err = ret.RowsAffected()
      	if err != nil {
      		fmt.Printf("get RowsAffected failed, err:%v\n", err)
      		return
      	}
      	fmt.Printf("update success, affected rows:%d\n", num)
      }
    ```
- 删除数据
  - ```DeleteRowCommunity
    func DeleteRowCommunity(id int) {
      	sqlStr := "delete from user where id=?"
      	ret, err := Db.Exec(sqlStr, id)
      	if err != nil {
      		fmt.Printf("delete failed, err:%v\n", err)
      		return
      	}
      	var num int64
      	num, err = ret.RowsAffected()
      	if err != nil {
      		fmt.Printf("get RowsAffected failed, err:%v\n", err)
      		return
      	}
      	fmt.Printf("delete success, affected rows:%d\n", num)
      }
    ```

## sqlx

```
type User struct {
	ID   int    `db:"id"`
	Age  int    `db:"age"`
	Name string `db:"name"`
}
```

- 单条数据查询
  - ```QueryRow
    func QueryRow() {
      	sqlStr := "select id, name, age from user where id = ?"
      	var u User
      	err := db.Get(&u, sqlStr, 2)
      	if err != nil {
      		fmt.Printf("get failed: err:%v\n", err)
      		return
      	}
      	fmt.Printf("users: %v\n", u)
      }
    ```
- 多条数据查询
  - ```QueryMultiRow
    func QueryMultiRow() {
      	sqlStr := "select id, name, age from user where id > ?"
      	var u []User
      	err := db.Select(&u, sqlStr, 2)
      	if err != nil {
      		fmt.Printf("get failed: err:%v\n", err)
      		return
      	}
      	fmt.Printf("users: %v\n", u)
      }
    ```
- 其他查询基本同 sql 一样
