---
title: c++连接mysql的中文编码问题
date: 2019-02-17 22:01:03
tags:
 - C++  
 - MySQL

categories: 数据库
copyright : true
---

c++连接mysql时，比如查询语句中含有中文，或者得到结果中含有中文，经常出现编译出错或乱码问题。
VS编译器默认使用gbk编码。
<!--more-->
如果将mysql设置为utf-8编码，则需要先将c++中的各种中文字符串转为utf-8编码输入mysql，得到的结果为utf-8编码，需要转为gbk才能正常显示。转来转去很麻烦。
换个角度，将mysql设置为gbk编码，这不就大功告成了吗?附代码如下：

```
#include <iostream>
#include <winsock2.h>
#include "mysql.h"

using namespace std;
int main(int argc, char* argv[])
{
	mysql_library_init(NULL, 0, 0);
	MYSQL mysql;
	mysql_init(&mysql);
	
	if (0 == mysql_options(&mysql, MYSQL_SET_CHARSET_NAME, "gbk"))//设置字符集
	{
		cout << "设置字符集成功\n\n" << endl;
	}

	if (!mysql_real_connect(&mysql, "localhost", "root", "123456", "测试", 3306, NULL, CLIENT_MULTI_STATEMENTS))//连接数据库
	{
		cout << "not connect mysql" << endl;
	}
	else
	{
		cout << "welcome to mysql\n\n\n";
	}
	
	mysql_query(&mysql, "select * from 学生信息");          //执行SQL语句
	MYSQL_RES *result = mysql_use_result(&mysql);       //获取资源
	int rowcount = mysql_num_rows(result);                //获取记录数
	unsigned int fieldcount = mysql_num_fields(result);   //获取字段数

	MYSQL_FIELD *field = NULL;                            //字段
	MYSQL_ROW row = NULL;                         //记录
	while (row = mysql_fetch_row(result))
	{
		for (unsigned int i = 0; i<fieldcount; i++)
		{
			field = mysql_fetch_field_direct(result, i);
			cout << field->name << ":" << row[i] << "\n";
		}
	}

	mysql_free_result(result);
	mysql_close(&mysql);
	mysql_server_end();
	mysql_library_end();

	return 0;
}
```
