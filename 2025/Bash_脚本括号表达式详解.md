# Bash 脚本括号表达式详解

## 概述

本文档详细介绍了bash脚本中各种括号表达式的用法和语法规则。这些表达式是bash脚本编程的基础工具，每种都有其特定的用途。

## 表达式类型

### 1. **$() - 命令替换**

- **功能**: 执行命令并捕获其输出，允许将命令结果存储在变量中
- **语法**: `$(command)`
- **示例**:
  ```bash
  log_file="/var/log/syslog"
  keyword="error"
  output=$(grep "$keyword" "$log_file")
  ```

### 2. **{} - 命令分组**

- **功能**: 在同一个shell进程中执行一组命令
- **语法**: `{ command1; command2; command3; }`
- **示例**:
  ```bash
  {
      sudo apt install exa
      echo exa
      echo "Listed files using exa"
  }
  ```

### 3. **() - 数组创建**

- **功能**: 创建一个值数组，用括号定义数组
- **语法**: `array_name=(value1 value2 value3)`
- **示例**:
  ```bash
  files=(log.txt log2.txt log3.txt)
  for file in "${files[@]}"; do
      echo "Processing $file"
  done
  ```

### 4. **( ) - 子shell执行**

- **功能**: 在独立的子shell中执行命令列表
- **语法**: `( command_list )`
- **示例**:
  ```bash
  (
      cd /home/user
      ls
      whoami
  )
  ```

### 5. **{范围} - 大括号展开**

- **功能**: 展开为多个字符串，用于生成序列或多个字符串
- **语法**: `{start..end}` 或 `{option1,option2,option3}`
- **示例**:
  ```bash
  for file in backup_{1..4}.tar.gz; do
      mv $file /var/oldbackups
  done
  ```

### 6. **${表达式} - 参数展开**

- **功能**: 修改变量内容，允许改变变量的值
- **语法**: `${variable%pattern}`, `${variable#pattern}`, `${variable/pattern/replacement}`
- **示例**:
  ```bash
  filename="report.txt"
  backup_file="${filename%.txt}.bak"
  echo "Backup file: $backup_file"
  ```

### 7. **${变量} - 变量访问**

- **功能**: 访问变量的值，在需要附加字符时使用
- **语法**: `${variable_name}`
- **示例**:
  ```bash
  username="John"
  greeting="Hello, ${username}!"
  echo "$greeting"
  ```

### 8. **$(()) - 算术运算**

- **功能**: 执行算术计算，双括号用于数学运算
- **语法**: `$((expression))`
- **示例**:
  ```bash
  num1=5
  num2=3
  result=$((num1 * num2 + 1))
  ```

### 9. **[ ] - 测试命令（单括号）**

- **功能**: 使用单括号测试条件，检查文件是否存在等
- **语法**: `[ condition ]`
- **示例**:
  ```bash
  file="/etc/passwd"
  if [ -f "$file" ]; then
      echo "File exists"
  fi
  ```

### 10. **[[ ]] - 测试命令（双括号）**

- **功能**: 使用双括号测试条件，功能更强大
- **语法**: `[[ condition ]]`
- **特点**: 支持高级模式匹配和逻辑运算符
- **示例**:
  ```bash
  user=$USER
  if [[ $user == "root" ]]; then
      echo "You are the root user"
  fi
  ```

## 使用建议

1. **命令替换**: 优先使用 `$()` 而不是反引号 `` ` ` ``
2. **条件测试**: 在bash中优先使用 `[[ ]]` 而不是 `[ ]`
3. **算术运算**: 使用 `$(())` 进行数学计算
4. **数组操作**: 使用 `()` 创建数组，`"${array[@]}"` 展开所有元素
5. **变量展开**: 使用 `${}` 进行复杂的变量操作

## 总结

这些括号表达式是bash脚本编程的核心工具，掌握它们的用法能够大大提高脚本编写的效率和功能性。每种表达式都有其特定的应用场景，正确使用能够让脚本更加健壮和可读。

---

*文档创建时间: 2025年*
*参考资料: sysxplore.com bash brackets guide*