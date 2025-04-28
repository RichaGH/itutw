# Вопрос
How can i see all rights and privileges granted or inherited by the role wich name=<myrole>?

# Объяснение
Okay, the user wants to check all the rights and privileges granted to or inherited by a specific role named <myrole> in PostgreSQL. Let me think about how to approach this.

First, I know that in PostgreSQL, roles can have privileges directly assigned or inherited through group membership. So the user needs to see both direct privileges and those from other roles that <myrole> is a member of.

I remember there are system catalog tables that store this information. The key ones are pg_roles for role information, pg_auth_members for role memberships, and various privilege-related views like information_schema.table_privileges for table privileges.

Maybe start by checking the basic role attributes using \du in psql. That shows roles and their attributes, but might not show all privileges. Then, to get detailed info on memberships, query pg_auth_members where the roleid is the member. But to get role names, I need to join with pg_roles using the oid.

For database privileges, the pg_database.datacl column holds the access privileges, but parsing that might be tricky. Similarly, schema privileges are in pg_namespace.nspacl, and table privileges in information_schema.table_privileges or pg_class.relacl.

Wait, using \dp or \z commands in psql can show table-level privileges, but the user might want a SQL query. Also, default privileges might be a factor, but maybe that's more complex than needed.

I should also mention the has_table_privilege function to check specific privileges on objects. Maybe include examples of checking for SELECT on a particular table.

Putting it all together: the user needs a combination of queries to check role memberships, database, schema, and table privileges. Also, note that system catalog access might be restricted, so the user needs privileges to view those tables.

I should structure the answer with steps: check role attributes, memberships, database, schema, and table privileges, then object-specific checks. Maybe include the psql commands and SQL queries, explaining each part.

# Ответ

To inspect all privileges and inherited permissions for a role (`<myrole>`) in PostgreSQL, use the following methods:

---

### **1. Check Role Attributes & Membership**
#### **Via `psql` Command:**
```sql
\du <myrole>  -- Lists role attributes, privileges, and group memberships
```

#### **Via SQL Queries:**
```sql
-- Check direct privileges and role membership
SELECT 
  rolname, 
  rolsuper, 
  rolcreaterole, 
  rolcreatedb, 
  rolinherit, 
  rolcanlogin 
FROM pg_roles 
WHERE rolname = '<myrole>';

-- List all roles that <myrole> is a member of (inherited privileges)
SELECT 
  pg_get_userbyid(roleid) AS role,
  pg_get_userbyid(member) AS member,
  admin_option
FROM pg_auth_members 
WHERE member = (SELECT oid FROM pg_roles WHERE rolname = '<myrole>');
```

---

### **2. Check Database-Level Privileges**
```sql
-- Show privileges on the database
SELECT 
  datname, 
  pg_get_userbyid(datdba) AS owner,
  datacl 
FROM pg_database 
WHERE datname = current_database();  -- Replace with your DB name if needed
```

---

### **3. Check Schema-Level Privileges**
```sql
-- Show privileges on schemas
SELECT 
  nspname AS schema,
  pg_get_userbyid(nspowner) AS owner,
  nspacl 
FROM pg_namespace;
```

---

### **4. Check Table-Level Privileges**
#### **Via `psql` Command:**
```sql
\dp *.*  -- Lists all table/column privileges in the current database
```

#### **Via SQL Query:**
```sql
-- List table privileges for <myrole>
SELECT 
  grantee, 
  table_schema, 
  table_name, 
  privilege_type 
FROM information_schema.table_privileges 
WHERE grantee = '<myrole>';
```

---

### **5. Check Default Privileges**
```sql
-- Show default privileges granted by <myrole> or to <myrole>
SELECT 
  defaclrole::regrole AS granting_role,
  defaclobjtype AS object_type,
  defaclacl AS privileges
FROM pg_default_acl;
```

---

### **6. Check Object-Specific Privileges**
Use PostgreSQL’s `has_table_privilege` function to test specific access:
```sql
-- Check if <myrole> has SELECT on a specific table
SELECT has_table_privilege('<myrole>', 'schema.table', 'SELECT');
```

---

### **Key Notes**
- **`pg_catalog` Access**: If `<myrole>` has access to `pg_catalog`, it can query metadata (e.g., `pg_tables`, `pg_views`).
- **Inherited Privileges**: Privileges from group roles are automatically inherited unless `NOINHERIT` is set.
- **`datacl`/`nspacl`/`relacl`**: These columns store access control lists (ACLs) in raw format. Use helper functions like `aclexplode` to parse them.

---

### **Example Output Interpretation**
If you see entries like:
```sql
rolname | rolsuper | rolcreaterole | rolcreatedb | rolinherit | rolcanlogin
--------+----------+---------------+-------------+------------+-------------
myrole  | f        | f             | f           | t          | f

datacl: {=Tc/postgres,postgres=CTc/postgres,myrole=c/postgres}
```
- `=Tc/postgres`: `myrole` has `CONNECT` (`c`) and `TEMPORARY` (`T`) on the database.
- No `INSERT`, `UPDATE`, or `SELECT` privileges imply no data access.

This approach ensures you see exactly what `<myrole>` can or cannot do. Adjust schema/table names as needed!
