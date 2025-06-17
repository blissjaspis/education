# Practical MySQL Guide

A comprehensive guide to essential MySQL administration tasks, user management, and security practices.

## Table of Contents

1. [Creating MySQL Users](#creating-mysql-users)
2. [Granting Privileges](#granting-privileges)
3. [Managing User Permissions](#managing-user-permissions)
4. [Viewing Current Users and Privileges](#viewing-current-users-and-privileges)
5. [Revoking Privileges](#revoking-privileges)
6. [Database and Table Management](#database-and-table-management)
7. [Security Best Practices](#security-best-practices)
8. [Common Scenarios](#common-scenarios)
9. [Troubleshooting](#troubleshooting)

## Creating MySQL Users

### Basic User Creation

```sql
-- Create a user with password
CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';

-- Create a user that can connect from any host
CREATE USER 'username'@'%' IDENTIFIED BY 'password';

-- Create a user for specific IP address
CREATE USER 'username'@'192.168.1.100' IDENTIFIED BY 'password';

-- Create a user with no password (not recommended)
CREATE USER 'username'@'localhost';
```

### User Creation with Authentication

```sql
-- Create user with mysql_native_password (MySQL 8.0+)
CREATE USER 'username'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';

-- Create user with caching_sha2_password (default in MySQL 8.0+)
CREATE USER 'username'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'password';

-- Create user with password expiration
CREATE USER 'username'@'localhost' IDENTIFIED BY 'password' PASSWORD EXPIRE INTERVAL 90 DAY;
```

## Granting Privileges

### Database Level Privileges

```sql
-- Grant all privileges on a specific database
GRANT ALL PRIVILEGES ON database_name.* TO 'username'@'localhost';

-- Grant specific privileges on a database
GRANT SELECT, INSERT, UPDATE, DELETE ON database_name.* TO 'username'@'localhost';

-- Grant read-only access to a database
GRANT SELECT ON database_name.* TO 'username'@'localhost';
```

### Table Level Privileges

```sql
-- Grant privileges on a specific table
GRANT SELECT, INSERT, UPDATE ON database_name.table_name TO 'username'@'localhost';

-- Grant specific column privileges
GRANT SELECT (column1, column2), UPDATE (column1) ON database_name.table_name TO 'username'@'localhost';
```

### Global Privileges

```sql
-- Grant global privileges (use with caution)
GRANT ALL PRIVILEGES ON *.* TO 'username'@'localhost';

-- Grant specific global privileges
GRANT CREATE, DROP, RELOAD, PROCESS ON *.* TO 'username'@'localhost';

-- Grant administrative privileges
GRANT SUPER, RELOAD, SHUTDOWN, PROCESS ON *.* TO 'admin_user'@'localhost';
```

### Common Privilege Types

| Privilege | Description |
|-----------|-------------|
| `ALL PRIVILEGES` | All available privileges |
| `SELECT` | Read data from tables |
| `INSERT` | Insert new rows |
| `UPDATE` | Modify existing rows |
| `DELETE` | Delete rows |
| `CREATE` | Create databases and tables |
| `DROP` | Drop databases and tables |
| `ALTER` | Modify table structure |
| `INDEX` | Create and drop indexes |
| `GRANT OPTION` | Grant privileges to other users |
| `SUPER` | Administrative privileges |
| `PROCESS` | View running processes |
| `RELOAD` | Reload privileges and logs |

## Managing User Permissions

### Applying Changes

```sql
-- Apply privilege changes immediately
FLUSH PRIVILEGES;
```

### Modifying Existing Users

```sql
-- Change user password
ALTER USER 'username'@'localhost' IDENTIFIED BY 'new_password';

-- Set password expiration
ALTER USER 'username'@'localhost' PASSWORD EXPIRE;

-- Disable password expiration
ALTER USER 'username'@'localhost' PASSWORD EXPIRE NEVER;

-- Lock/unlock user account
ALTER USER 'username'@'localhost' ACCOUNT LOCK;
ALTER USER 'username'@'localhost' ACCOUNT UNLOCK;
```

## Viewing Current Users and Privileges

### List All Users

```sql
-- View all MySQL users
SELECT User, Host FROM mysql.user;

-- View users with additional information
SELECT User, Host, authentication_string, plugin FROM mysql.user;
```

### Check User Privileges

```sql
-- Show privileges for current user
SHOW GRANTS;

-- Show privileges for specific user
SHOW GRANTS FOR 'username'@'localhost';

-- View detailed privilege information
SELECT * FROM information_schema.USER_PRIVILEGES WHERE GRANTEE = "'username'@'localhost'";
SELECT * FROM information_schema.SCHEMA_PRIVILEGES WHERE GRANTEE = "'username'@'localhost'";
SELECT * FROM information_schema.TABLE_PRIVILEGES WHERE GRANTEE = "'username'@'localhost'";
```

## Revoking Privileges

### Revoke Specific Privileges

```sql
-- Revoke specific privileges
REVOKE INSERT, UPDATE ON database_name.* FROM 'username'@'localhost';

-- Revoke all privileges on a database
REVOKE ALL PRIVILEGES ON database_name.* FROM 'username'@'localhost';

-- Revoke grant option
REVOKE GRANT OPTION ON database_name.* FROM 'username'@'localhost';
```

### Delete User

```sql
-- Drop a user (removes user and all privileges)
DROP USER 'username'@'localhost';

-- Drop multiple users
DROP USER 'user1'@'localhost', 'user2'@'%';
```

## Database and Table Management

### Creating Databases

```sql
-- Create database
CREATE DATABASE database_name;

-- Create database with character set
CREATE DATABASE database_name CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create database if not exists
CREATE DATABASE IF NOT EXISTS database_name;
```

### Managing Databases

```sql
-- List all databases
SHOW DATABASES;

-- Use a database
USE database_name;

-- Drop database
DROP DATABASE database_name;

-- Drop database if exists
DROP DATABASE IF EXISTS database_name;
```

### Table Operations

```sql
-- Show tables in current database
SHOW TABLES;

-- Show table structure
DESCRIBE table_name;
SHOW COLUMNS FROM table_name;

-- Show table creation statement
SHOW CREATE TABLE table_name;
```

## Security Best Practices

### 1. Principle of Least Privilege

```sql
-- ❌ Bad: Granting unnecessary global privileges
GRANT ALL PRIVILEGES ON *.* TO 'app_user'@'%';

-- ✅ Good: Grant only required privileges
GRANT SELECT, INSERT, UPDATE, DELETE ON app_database.* TO 'app_user'@'localhost';
```

### 2. Strong Password Policies

```sql
-- Set password validation
INSTALL COMPONENT 'file://component_validate_password';

-- Configure password validation
SET GLOBAL validate_password.length = 8;
SET GLOBAL validate_password.mixed_case_count = 1;
SET GLOBAL validate_password.number_count = 1;
SET GLOBAL validate_password.special_char_count = 1;
```

### 3. Host Restrictions

```sql
-- ❌ Bad: Allow connections from anywhere
CREATE USER 'user'@'%' IDENTIFIED BY 'password';

-- ✅ Good: Restrict to specific hosts
CREATE USER 'user'@'localhost' IDENTIFIED BY 'password';
CREATE USER 'user'@'192.168.1.0/255.255.255.0' IDENTIFIED BY 'password';
```

### 4. Regular Security Audits

```sql
-- Check for users without passwords
SELECT User, Host FROM mysql.user WHERE authentication_string = '';

-- Check for users with global privileges
SELECT User, Host FROM mysql.user WHERE Super_priv = 'Y' OR Grant_priv = 'Y';

-- Check for anonymous users
SELECT User, Host FROM mysql.user WHERE User = '';
```

## Common Scenarios

### 1. Application Database User

```sql
-- Create application user with limited privileges
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'secure_password123!';
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp_db.* TO 'app_user'@'localhost';
FLUSH PRIVILEGES;
```

### 2. Read-Only Reporting User

```sql
-- Create read-only user for reporting
CREATE USER 'report_user'@'%' IDENTIFIED BY 'report_password456!';
GRANT SELECT ON reporting_db.* TO 'report_user'@'%';
FLUSH PRIVILEGES;
```

### 3. Backup User

```sql
-- Create backup user with necessary privileges
CREATE USER 'backup_user'@'localhost' IDENTIFIED BY 'backup_password789!';
GRANT SELECT, RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'backup_user'@'localhost';
FLUSH PRIVILEGES;
```

### 4. Developer User

```sql
-- Create developer user with broader privileges for development database
CREATE USER 'dev_user'@'localhost' IDENTIFIED BY 'dev_password101!';
GRANT ALL PRIVILEGES ON dev_database.* TO 'dev_user'@'localhost';
GRANT CREATE, DROP ON dev_*.* TO 'dev_user'@'localhost';
FLUSH PRIVILEGES;
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Access Denied Errors

```sql
-- Check if user exists
SELECT User, Host FROM mysql.user WHERE User = 'username';

-- Verify privileges
SHOW GRANTS FOR 'username'@'localhost';

-- Ensure privileges are flushed
FLUSH PRIVILEGES;
```

#### 2. Connection Issues

```sql
-- Check user's allowed hosts
SELECT User, Host FROM mysql.user WHERE User = 'username';

-- Verify user is not locked
SELECT User, Host, account_locked FROM mysql.user WHERE User = 'username';
```

#### 3. Password Issues

```sql
-- Reset user password
ALTER USER 'username'@'localhost' IDENTIFIED BY 'new_password';

-- Check password expiration
SELECT User, Host, password_expired FROM mysql.user WHERE User = 'username';
```

### Useful Administrative Queries

```sql
-- Show current connections
SHOW PROCESSLIST;

-- Show current database size
SELECT 
    table_schema AS 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.tables 
GROUP BY table_schema;

-- Show table sizes
SELECT 
    table_name AS 'Table',
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size (MB)'
FROM information_schema.tables 
WHERE table_schema = 'your_database_name'
ORDER BY (data_length + index_length) DESC;
```

## Quick Reference Commands

```bash
# Connect to MySQL
mysql -u username -p

# Connect to specific database
mysql -u username -p database_name

# Connect to remote MySQL server
mysql -h hostname -u username -p

# Execute SQL file
mysql -u username -p database_name < script.sql

# Export database
mysqldump -u username -p database_name > backup.sql

# Import database
mysql -u username -p database_name < backup.sql
```

---

**Remember**: Always follow the principle of least privilege, use strong passwords, and regularly audit your MySQL users and their privileges for security.
