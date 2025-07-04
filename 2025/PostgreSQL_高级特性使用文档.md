# PostgreSQL 高级特性使用文档

本文档详细介绍 PostgreSQL 中视图、函数、过程和触发器的使用方法，包含语法说明和实际示例。

## 目录
1. [视图 (Views)](#视图-views)
2. [函数 (Functions)](#函数-functions)
3. [存储过程 (Procedures)](#存储过程-procedures)
4. [触发器 (Triggers)](#触发器-triggers)
5. [最佳实践](#最佳实践)

---

## 视图 (Views)

### 什么是视图
视图是基于SQL语句结果集的虚拟表。它包含行和列，就像真正的表一样，但视图中的字段来自一个或多个真实的表中的字段。

### 创建视图

#### 基本语法
```sql
CREATE VIEW view_name AS
SELECT column1, column2, ...
FROM table_name
WHERE condition;
```

#### 示例1：简单视图
```sql
-- 创建员工信息视图
CREATE VIEW employee_info AS
SELECT 
    emp_id,
    first_name,
    last_name,
    department,
    salary
FROM employees
WHERE status = 'active';
```

#### 示例2：复杂视图（多表连接）
```sql
-- 创建员工部门详情视图
CREATE VIEW employee_department_view AS
SELECT 
    e.emp_id,
    e.first_name || ' ' || e.last_name AS full_name,
    d.department_name,
    d.location,
    e.salary,
    e.hire_date
FROM employees e
INNER JOIN departments d ON e.department_id = d.dept_id
WHERE e.status = 'active';
```

#### 示例3：可更新视图
```sql
-- 创建可更新的视图
CREATE VIEW active_employees AS
SELECT emp_id, first_name, last_name, email, salary
FROM employees
WHERE status = 'active'
WITH CHECK OPTION;
```

### 视图管理

#### 查看视图定义
```sql
-- 查看视图定义
\d+ view_name

-- 或者使用系统表
SELECT definition FROM pg_views WHERE viewname = 'employee_info';
```

#### 修改视图
```sql
-- 替换视图
CREATE OR REPLACE VIEW employee_info AS
SELECT 
    emp_id,
    first_name,
    last_name,
    department,
    salary,
    email  -- 新增字段
FROM employees
WHERE status = 'active';
```

#### 删除视图
```sql
DROP VIEW IF EXISTS view_name;
```

---

## 函数 (Functions)

### 什么是函数
PostgreSQL 函数是可重用的代码块，接受参数并返回值。函数可以用多种编程语言编写。

### PL/pgSQL 函数

#### 基本语法
```sql
CREATE OR REPLACE FUNCTION function_name(parameter_list)
RETURNS return_type
LANGUAGE plpgsql
AS $$
DECLARE
    -- 变量声明
BEGIN
    -- 函数体
    RETURN value;
END;
$$;
```

#### 示例1：简单计算函数
```sql
-- 计算两个数的和
CREATE OR REPLACE FUNCTION add_numbers(a INTEGER, b INTEGER)
RETURNS INTEGER
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN a + b;
END;
$$;

-- 使用函数
SELECT add_numbers(10, 20);
```

#### 示例2：字符串处理函数
```sql
-- 格式化员工姓名
CREATE OR REPLACE FUNCTION format_employee_name(first_name TEXT, last_name TEXT)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
    IF first_name IS NULL OR last_name IS NULL THEN
        RETURN 'Unknown';
    END IF;
    
    RETURN UPPER(last_name) || ', ' || INITCAP(first_name);
END;
$$;

-- 使用示例
SELECT format_employee_name('john', 'doe');
```

#### 示例3：返回表的函数
```sql
-- 获取部门员工信息的函数
CREATE OR REPLACE FUNCTION get_department_employees(dept_name TEXT)
RETURNS TABLE(
    employee_id INTEGER,
    full_name TEXT,
    position TEXT,
    salary NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        e.emp_id,
        e.first_name || ' ' || e.last_name,
        e.position,
        e.salary
    FROM employees e
    INNER JOIN departments d ON e.department_id = d.dept_id
    WHERE d.department_name = dept_name
    AND e.status = 'active';
END;
$$;

-- 使用函数
SELECT * FROM get_department_employees('Engineering');
```

#### 示例4：带异常处理的函数
```sql
-- 安全的除法函数
CREATE OR REPLACE FUNCTION safe_divide(dividend NUMERIC, divisor NUMERIC)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
BEGIN
    IF divisor = 0 THEN
        RAISE EXCEPTION 'Division by zero is not allowed';
    END IF;
    
    RETURN dividend / divisor;
EXCEPTION
    WHEN OTHERS THEN
        RAISE NOTICE 'Error occurred: %', SQLERRM;
        RETURN NULL;
END;
$$;
```

### SQL 函数

#### 示例：SQL函数
```sql
-- 获取员工总数的SQL函数
CREATE OR REPLACE FUNCTION get_employee_count()
RETURNS BIGINT
LANGUAGE sql
AS $$
    SELECT COUNT(*) FROM employees WHERE status = 'active';
$$;
```

---

## 存储过程 (Procedures)

### 什么是存储过程
存储过程是预编译的SQL语句集合，可以执行复杂的业务逻辑，但不返回值（PostgreSQL 11+ 支持）。

### 基本语法
```sql
CREATE OR REPLACE PROCEDURE procedure_name(parameter_list)
LANGUAGE plpgsql
AS $$
DECLARE
    -- 变量声明
BEGIN
    -- 过程体
END;
$$;
```

### 示例1：数据维护过程
```sql
-- 清理过期数据的存储过程
CREATE OR REPLACE PROCEDURE cleanup_expired_sessions()
LANGUAGE plpgsql
AS $$
DECLARE
    deleted_count INTEGER;
BEGIN
    -- 删除过期的会话
    DELETE FROM user_sessions 
    WHERE expires_at < NOW();
    
    GET DIAGNOSTICS deleted_count = ROW_COUNT;
    
    -- 记录日志
    INSERT INTO system_logs (action, details, created_at)
    VALUES ('cleanup_sessions', 
            'Deleted ' || deleted_count || ' expired sessions',
            NOW());
    
    RAISE NOTICE 'Cleaned up % expired sessions', deleted_count;
END;
$$;

-- 调用存储过程
CALL cleanup_expired_sessions();
```

### 示例2：数据迁移过程
```sql
-- 数据迁移存储过程
CREATE OR REPLACE PROCEDURE migrate_employee_data(
    source_table TEXT,
    target_table TEXT
)
LANGUAGE plpgsql
AS $$
DECLARE
    rec RECORD;
    processed_count INTEGER := 0;
BEGIN
    -- 开始事务
    BEGIN
        -- 动态迁移数据
        FOR rec IN EXECUTE format('SELECT * FROM %I', source_table)
        LOOP
            EXECUTE format('INSERT INTO %I VALUES ($1.*)', target_table) USING rec;
            processed_count := processed_count + 1;
        END LOOP;
        
        RAISE NOTICE 'Successfully migrated % records', processed_count;
        
    EXCEPTION
        WHEN OTHERS THEN
            RAISE EXCEPTION 'Migration failed: %', SQLERRM;
    END;
END;
$$;
```

---

## 触发器 (Triggers)

### 什么是触发器
触发器是特殊的存储过程，在特定数据库事件发生时自动执行。

### 触发器类型
- **BEFORE**: 在事件发生前执行
- **AFTER**: 在事件发生后执行
- **INSTEAD OF**: 替代事件执行（主要用于视图）

### 创建触发器函数

#### 基本语法
```sql
-- 1. 创建触发器函数
CREATE OR REPLACE FUNCTION trigger_function_name()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- 触发器逻辑
    RETURN NEW; -- 或 OLD 或 NULL
END;
$$;

-- 2. 创建触发器
CREATE TRIGGER trigger_name
    BEFORE/AFTER INSERT/UPDATE/DELETE
    ON table_name
    FOR EACH ROW
    EXECUTE FUNCTION trigger_function_name();
```

### 示例1：审计触发器
```sql
-- 创建审计表
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name TEXT,
    operation TEXT,
    old_values JSONB,
    new_values JSONB,
    user_name TEXT,
    timestamp TIMESTAMP DEFAULT NOW()
);

-- 创建审计触发器函数
CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, old_values, user_name)
        VALUES (TG_TABLE_NAME, TG_OP, row_to_json(OLD), current_user);
        RETURN OLD;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, old_values, new_values, user_name)
        VALUES (TG_TABLE_NAME, TG_OP, row_to_json(OLD), row_to_json(NEW), current_user);
        RETURN NEW;
    ELSIF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, new_values, user_name)
        VALUES (TG_TABLE_NAME, TG_OP, row_to_json(NEW), current_user);
        RETURN NEW;
    END IF;
    RETURN NULL;
END;
$$;

-- 为employees表创建审计触发器
CREATE TRIGGER employees_audit_trigger
    AFTER INSERT OR UPDATE OR DELETE
    ON employees
    FOR EACH ROW
    EXECUTE FUNCTION audit_trigger();
```

### 示例2：自动更新时间戳触发器
```sql
-- 创建更新时间戳的触发器函数
CREATE OR REPLACE FUNCTION update_modified_column()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$;

-- 为表添加自动更新时间戳触发器
CREATE TRIGGER update_employees_modtime
    BEFORE UPDATE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION update_modified_column();
```

### 示例3：数据验证触发器
```sql
-- 创建数据验证触发器函数
CREATE OR REPLACE FUNCTION validate_employee_data()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- 验证邮箱格式
    IF NEW.email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN
        RAISE EXCEPTION 'Invalid email format: %', NEW.email;
    END IF;
    
    -- 验证薪资范围
    IF NEW.salary < 0 OR NEW.salary > 1000000 THEN
        RAISE EXCEPTION 'Salary must be between 0 and 1,000,000';
    END IF;
    
    -- 自动设置创建时间
    IF TG_OP = 'INSERT' THEN
        NEW.created_at = NOW();
    END IF;
    
    RETURN NEW;
END;
$$;

-- 创建验证触发器
CREATE TRIGGER validate_employee_trigger
    BEFORE INSERT OR UPDATE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION validate_employee_data();
```

### 触发器管理

#### 查看触发器
```sql
-- 查看表的所有触发器
SELECT 
    tgname AS trigger_name,
    tgtype AS trigger_type,
    proname AS function_name
FROM pg_trigger t
JOIN pg_proc p ON t.tgfoid = p.oid
JOIN pg_class c ON t.tgrelid = c.oid
WHERE c.relname = 'employees';
```

#### 禁用/启用触发器
```sql
-- 禁用触发器
ALTER TABLE employees DISABLE TRIGGER trigger_name;

-- 启用触发器
ALTER TABLE employees ENABLE TRIGGER trigger_name;

-- 禁用所有触发器
ALTER TABLE employees DISABLE TRIGGER ALL;
```

#### 删除触发器
```sql
DROP TRIGGER IF EXISTS trigger_name ON table_name;
```

---

## 最佳实践

### 视图最佳实践
1. **命名规范**: 使用 `v_` 前缀或 `_view` 后缀
2. **性能考虑**: 避免在视图中使用复杂的子查询
3. **安全性**: 使用视图控制数据访问权限
4. **文档化**: 为复杂视图添加注释

```sql
-- 好的视图命名和注释示例
CREATE VIEW v_employee_summary AS
SELECT 
    emp_id,
    full_name,
    department,
    salary_grade
FROM employees_detailed_info
WHERE status = 'active';

COMMENT ON VIEW v_employee_summary IS '活跃员工摘要信息，用于报表展示';
```

### 函数最佳实践
1. **单一职责**: 每个函数只做一件事
2. **参数验证**: 始终验证输入参数
3. **异常处理**: 妥善处理可能的异常
4. **性能优化**: 使用适当的索引和查询计划

```sql
-- 良好的函数示例
CREATE OR REPLACE FUNCTION get_employee_by_id(emp_id INTEGER)
RETURNS TABLE(
    employee_id INTEGER,
    full_name TEXT,
    department TEXT,
    salary NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
    -- 参数验证
    IF emp_id IS NULL OR emp_id <= 0 THEN
        RAISE EXCEPTION 'Invalid employee ID: %', emp_id;
    END IF;
    
    -- 返回查询结果
    RETURN QUERY
    SELECT e.emp_id, e.first_name || ' ' || e.last_name, d.name, e.salary
    FROM employees e
    JOIN departments d ON e.dept_id = d.id
    WHERE e.emp_id = get_employee_by_id.emp_id;
    
    -- 检查是否找到记录
    IF NOT FOUND THEN
        RAISE NOTICE 'No employee found with ID: %', emp_id;
    END IF;
END;
$$;
```

### 存储过程最佳实践
1. **事务管理**: 合理使用事务
2. **错误处理**: 实现完整的错误处理机制
3. **日志记录**: 记录重要操作
4. **原子性**: 确保操作的原子性

### 触发器最佳实践
1. **最小化逻辑**: 保持触发器逻辑简单
2. **避免递归**: 防止触发器相互调用
3. **性能影响**: 考虑触发器对DML操作的性能影响
4. **测试充分**: 全面测试触发器的各种场景

```sql
-- 性能友好的触发器示例
CREATE OR REPLACE FUNCTION efficient_audit_trigger()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- 只在必要时记录审计
    IF TG_OP = 'UPDATE' THEN
        -- 只在重要字段变化时记录
        IF OLD.salary != NEW.salary OR OLD.department_id != NEW.department_id THEN
            INSERT INTO audit_log (table_name, operation, record_id, changes)
            VALUES (TG_TABLE_NAME, TG_OP, NEW.emp_id, 
                   jsonb_build_object('old', row_to_json(OLD), 'new', row_to_json(NEW)));
        END IF;
    END IF;
    
    RETURN COALESCE(NEW, OLD);
END;
$$;
```

### 总体最佳实践
1. **版本控制**: 将数据库对象的创建脚本纳入版本控制
2. **测试**: 为所有数据库对象编写测试用例
3. **文档**: 维护详细的文档和注释
4. **监控**: 监控性能和使用情况
5. **备份**: 定期备份数据库结构和数据

---

## 结语

PostgreSQL 的视图、函数、过程和触发器是强大的数据库特性，正确使用它们可以：
- 提高代码重用性
- 增强数据安全性
- 简化复杂查询
- 实现业务逻辑自动化
- 维护数据完整性

建议在实际项目中根据具体需求选择合适的特性，并遵循最佳实践以确保系统的可维护性和性能。