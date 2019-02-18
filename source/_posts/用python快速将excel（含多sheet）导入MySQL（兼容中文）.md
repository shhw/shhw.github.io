---
title: 用python快速将excel（含多sheet）导入MySQL（兼容中文）
date: 2019-02-17 21:43:15
tags:
  - python 
  - MySQL

categories: 数据库
copyright : true
---

需要使用xlrd和MySQLdb库，可自行百度下载。
<!--more-->

```
#coding:utf-8

import xlrd
import MySQLdb

data=xlrd.open_workbook(r'F:\test\baseParam.xls') #读取表格
db="测试" #需要操作的数据库

conn= MySQLdb.connect(
        host='localhost',   
        port = 3306,
        user='root',
        passwd='123456',
		charset='utf8'
        )   #连接mysql
cur=conn.cursor()
cur.execute("drop database if exists "+db)  
cur.execute("create database "+db)  
conn.select_db(db)  #初始化数据库

sheet_names=data.sheet_names()
for sheet_name in sheet_names:
	sheet=data.sheet_by_name(sheet_name)
	row_data=sheet.row_values(0)
	row_data=' varchar(256) DEFAULT NULL, '.join(row_data)
	row_data=row_data+' varchar(256) DEFAULT NULL'
	cur.execute('create table '+sheet_name+'('+row_data+')') #数据库中创建表格
	ss=''
	for index in range(sheet.ncols):
		ss=ss+'%s, '
	ss=ss.rstrip(', ')
	sql="insert "+ sheet_name+ " values(" +ss +")"
	param=[]
	for index in range(1,sheet.nrows):
		row_values=sheet.row_values(index)
		param.append(row_values)
	cur.executemany(sql,param) #插入数据
	conn.commit()
	
cur.close()  
conn.close()  #释放数据连接



```