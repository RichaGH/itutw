# вопрос
You are a database engineer with 20 years of experience. Your task is to create a group to access the metadata of all tables in the <mydatabase> postgresql version 11 database. Create a role to assign to the specified group. Create a user with this role. So that the user can eventually query the metadata of the tables, but cannot view the contents of the tables themselves. The user should not have the right to change anything in the database.

# Рассуждения
Okay, let's tackle this problem step by step. The user wants to create a group role in PostgreSQL 11 that can access metadata of all tables in a database called <mydatabase>, but without allowing access to the actual data. They also need a user assigned to this group who can't modify anything.

First, I remember that PostgreSQL uses roles for managing permissions. Groups are essentially roles that other roles can inherit. So, I'll start by creating a group role, maybe name it 'metadata_viewers'.

Next, the group needs access to metadata. Metadata in PostgreSQL is stored in the information_schema and pg_catalog schemas. But by default, users have some access to these, but maybe not all. The user shouldn't be able to see the table contents, so SELECT on tables should be denied. But how to allow metadata access?

I need to grant USAGE on the schema(s) where the metadata resides. For information_schema, granting USAGE should let them see the metadata. But wait, public schema access might be a problem. Wait, no, the information_schema and pg_catalog are separate. Also, by default, PUBLIC has access to information_schema views, but maybe not all. Need to check.

Wait, the user shouldn't be able to view table contents. So, even if they can see the table names in information_schema.tables, they shouldn't have SELECT on the actual tables. So the main thing is to ensure that the group role doesn't have any privileges on the tables themselves. But how to make sure that when new tables are created, they also don't get access? Maybe the default privileges are set correctly.

So the steps would be:

1. Create the group role: CREATE ROLE metadata_viewers;

2. Grant CONNECT on the database to the group so they can connect: GRANT CONNECT ON DATABASE mydatabase TO metadata_viewers;

3. Grant USAGE on the schemas (information_schema and pg_catalog?) to allow viewing metadata. But pg_catalog might have more sensitive info. Wait, maybe information_schema is safer. Let's think: information_schema is SQL standard and provides metadata views. So maybe granting USAGE on SCHEMA information_schema and then SELECT on the relevant views.

Wait, but in PostgreSQL, the information_schema is automatically available. However, to access its views, you need to have permissions on the underlying tables. Wait, no, the information_schema views are designed to show only the objects that the user has access to. So if the user has no access to any tables, the information_schema will show them only the tables they can access. But the user should see all tables' metadata, regardless of their permissions. Wait, that's conflicting. Hmm.

Wait, the information_schema.tables view lists all tables that the current user has access to (by way of being the owner or having some privilege). So if the user has no privileges on the tables, the information_schema won't show them. So that's a problem. The user needs to see all tables in the database but not access their data.

So how to allow them to see all tables in information_schema? They need to have any privilege on the tables, even if it's just a dummy privilege. Because PostgreSQL's information_schema will only list tables where the user has some privilege. So for example, if we grant SELECT on all tables to the group, but then revoke it, but that's messy. Alternatively, maybe grant some other privilege like USAGE or something else. Wait, maybe the GRANT USAGE ON SCHEMA public? No, that's for schemas.

Wait, perhaps the pg_catalog has tables that list all tables regardless of permissions. Because pg_catalog is the system catalog, and those tables are owned by PostgreSQL. So if the user has permission to read pg_catalog, they can see all tables. So maybe using pg_catalog instead of information_schema.

So the metadata can be accessed via pg_catalog.pg_tables, which lists all tables. But to access pg_catalog, the user needs to have permissions. By default, PUBLIC has access to pg_catalog, but maybe not all views. Let me check: in PostgreSQL, the pg_catalog tables are accessible by default, but certain columns might be hidden unless the user is a superuser. For example, pg_class.relacl (access privileges) is only visible if the user has permission.

But for just seeing table names, schemas, etc., pg_tables might be accessible. So maybe the user can query pg_catalog.pg_tables and see all tables. But I need to confirm.

Alternatively, maybe creating a role that can access pg_catalog and information_schema, but without granting any privileges on the actual tables. Then, the user can use those system views to see the metadata.

Wait, but by default, a new user can connect to a database and query pg_catalog.pg_tables, but they would see all tables in the database. Let's test this scenario.

Suppose I create a user with no privileges. Then, when they connect to the database, can they run SELECT * FROM pg_catalog.pg_tables; and see all tables? Yes, because pg_catalog is accessible to all users. However, they won't see the contents of the tables, just the metadata.

Wait, but the problem is that in PostgreSQL, the pg_catalog tables are world-readable. So a user can see all tables in pg_catalog, but when they try to select from a table they don't have permissions on, they get permission denied. But for metadata, like the list of tables, that's stored in pg_catalog, and those tables are readable by all users. So maybe the solution is to just create a user, grant them connect to the database, and they can query pg_catalog to get the metadata, but not access the actual tables.

But the user's requirement is to "access the metadata of all tables". If the user can already do that via pg_catalog, then maybe no additional grants are needed. But maybe the user is referring to using information_schema, which is more restrictive.

Wait, let's check. For example, if a user has no privileges on a table, then information_schema.tables will not list that table. But pg_catalog.pg_tables will list all tables, regardless of permissions. So if the user can query pg_catalog, they can see all tables, but not the data.

In that case, maybe the solution is to allow the user to access pg_catalog, which they can by default, and then ensure they have no privileges on the actual tables. Then they can see the metadata via pg_catalog, but cannot select from the tables.

But the original question mentions creating a group to access the metadata. So maybe the user expects to have explicit permissions on the information_schema. But according to the above, maybe that's not necessary. However, perhaps the user wants to use information_schema specifically, which would require granting some permissions.

Alternatively, the answer might be to grant USAGE on the schema(s) and then grant SELECT on specific information_schema views. But that might not be necessary if pg_catalog is sufficient.

Wait, but the user might not know about pg_catalog and might be using information_schema, so perhaps we need to grant the necessary privileges to see all tables in information_schema.

But how to make information_schema.tables list all tables for the user? That requires that the user has some privilege on each table. So, for example, if we grant USAGE on the schema where the tables are, then the user can see the tables in information_schema. But if the user doesn't have any privileges on the tables, they won't be listed.

Wait, here's the catch: information_schema.tables includes tables that the current user has any privilege on. So if we want the user to see all tables in the database via information_schema, the user must have at least one privilege on each table. But granting even a minimal privilege like SELECT would allow them to view the data, which we don't want.

So that's a problem. Therefore, using information_schema may not be feasible because to list the tables, the user needs to have some privilege on them. Therefore, the solution would be to use pg_catalog instead, which lists all tables regardless of privileges.

So, in that case, the user can just query pg_catalog.pg_tables, and they can see all tables without needing any special permissions. Therefore, the steps would be:

1. Create a group role: CREATE ROLE metadata_viewers;

2. Grant CONNECT on the database to the group: GRANT CONNECT ON DATABASE mydatabase TO metadata_viewers;

3. Create a user and assign the role: CREATE USER metadata_user WITH PASSWORD 'password'; GRANT metadata_viewers TO metadata_user;

4. Ensure that the user does not have any other privileges, so they cannot SELECT or modify any tables.

But wait, by default, when a user connects to a PostgreSQL database, they can access the pg_catalog schema and see all tables. However, they can't access the data in the tables unless granted permission.

So perhaps the initial approach is correct. The user can already view metadata via pg_catalog without any additional grants, but cannot view the table contents. Therefore, the group role doesn't need any additional privileges except CONNECT to the database.

But let me verify this with an example. Suppose there's a table 'mytable' in schema 'public'. A user with no privileges can connect to the database and run:

SELECT * FROM pg_catalog.pg_tables WHERE schemaname = 'public';

This would list 'mytable'. However, if they try to SELECT from mytable, they get a permission denied error. So this meets the requirement: the user can see the metadata (table exists) but can't view the contents.

But the original question mentions creating a group to access the metadata. So perhaps the steps are:

- Create a group role that has CONNECT privilege on the database.

- Assign this role to a user.

- No other privileges are needed, as pg_catalog is accessible by default.

But perhaps the database is set up with some default privileges that prevent access. For example, if the database has a schema that's not public, but in this case, the user wants all tables in the database. But pg_catalog.pg_tables lists all tables in all schemas.

Alternatively, if the user is supposed to access metadata via information_schema, then the approach would be different, but as discussed, that requires granting some privileges on the tables, which isn't desired.

Therefore, the answer is that the group role only needs CONNECT on the database. The user can then use pg_catalog to view table metadata.

But the user might have other requirements, like wanting to use information_schema. So perhaps the answer should cover both possibilities.

But according to the problem statement, the user wants to "access the metadata of all tables", which can be done via pg_catalog. So the steps would be as follows:

1. Create the group role:

CREATE ROLE metadata_viewers;

2. Grant CONNECT on the database to the group:

GRANT CONNECT ON DATABASE mydatabase TO metadata_viewers;

3. Revoke all privileges from the group on all schemas and tables. But by default, a new role has no privileges.

4. Create the user:

CREATE USER metadata_user WITH PASSWORD 'secure_password';

GRANT metadata_viewers TO metadata_user;

But wait, in PostgreSQL, the PUBLIC role has some default privileges. For example, in PostgreSQL, the PUBLIC role has USAGE on the pg_catalog schema. So users can access pg_catalog by default. Therefore, the user can already query pg_catalog.pg_tables to see all tables.

But in some cases, administrators might revoke default privileges. So to ensure access, maybe explicitly grant USAGE on pg_catalog:

GRANT USAGE ON SCHEMA pg_catalog TO metadata_viewers;

But in standard setups, this is already granted to PUBLIC. So perhaps it's redundant, but including it ensures that even if the defaults were changed, the group still has access.

Additionally, to prevent the user from accessing the tables, we need to make sure they don't have any privileges on the tables. Since the group is newly created and hasn't been granted any privileges, this should be the case.

Moreover, to prevent the user from altering anything, we should ensure they don't have CREATE, USAGE on other schemas, or any write privileges.

So, putting it all together:

- Create the group role.

- Grant CONNECT on the database.

- Grant USAGE on pg_catalog (though maybe redundant).

- Create the user and assign the group role.

Therefore, the SQL commands would be:

-- Create the group role

CREATE ROLE metadata_viewers;

-- Allow the group to connect to the database

GRANT CONNECT ON DATABASE mydatabase TO metadata_viewers;

-- Grant USAGE on pg_catalog to access system views (might be redundant)

GRANT USAGE ON SCHEMA pg_catalog TO metadata_viewers;

-- Create the user and assign the role

CREATE USER metadata_user WITH PASSWORD 'secure_pass';

GRANT metadata_viewers TO metadata_user;

Additionally, to ensure the user cannot modify anything, they should not have any write privileges. Since the group has no such privileges, this is already handled.

But perhaps the user should be able to query information_schema as well. In that case, we need to grant USAGE on information_schema:

GRANT USAGE ON SCHEMA information_schema TO metadata_viewers;

But again, by default, information_schema is accessible to PUBLIC. However, if the user wants to ensure that the group can access it, explicitly granting it may help.

But the information_schema might not show all tables unless the user has some privilege. So, to make information_schema.tables show all tables, the user needs to have any privilege on those tables. But that's not desired because that would require granting a privilege like SELECT, which allows data access.

Thus, the correct approach is to use pg_catalog for metadata, which doesn't require any privileges on the tables.

So the final answer would be the steps above. Let me write the SQL commands accordingly.

# Ответ

To achieve the desired setup in PostgreSQL 11, follow these steps:

### 1. **Create a Group Role for Metadata Access**
This role will allow members to view metadata without accessing table contents or modifying data.

```sql
-- Create the group role
CREATE ROLE metadata_viewers;
```

### 2. **Grant Database Connection Privilege**
Allow the group to connect to the target database (`mydatabase`):

```sql
GRANT CONNECT ON DATABASE mydatabase TO metadata_viewers;
```

### 3. **Grant Access to System Catalogs (Metadata)**
Explicitly grant access to `pg_catalog` (where PostgreSQL stores metadata). While this is often available by default, explicitly defining it ensures robustness:

```sql
GRANT USAGE ON SCHEMA pg_catalog TO metadata_viewers;
```

### 4. **Revoke Public Schema Privileges (Optional)**
If your tables are in the `public` schema, revoke default privileges to prevent accidental access. **Skip this if tables are in other schemas**:

```sql
REVOKE ALL PRIVILEGES ON SCHEMA public FROM metadata_viewers;
```

### 5. **Create a User and Assign the Group Role**
Create a user and grant them membership in the `metadata_viewers` group:

```sql
-- Create the user
CREATE USER metadata_user WITH PASSWORD 'secure_password';

-- Assign the user to the group
GRANT metadata_viewers TO metadata_user;
```

---

### **Key Points**
- **Metadata Access**: The user can query metadata via `pg_catalog` (e.g., `pg_tables`, `pg_columns`) to see all tables, columns, etc., but **cannot access actual table data**.
- **No Write Privileges**: The group role has no privileges to `INSERT`, `UPDATE`, `DELETE`, or modify the database structure.
- **Example Metadata Query**:
  ```sql
  -- List all tables in the database
  SELECT * FROM pg_catalog.pg_tables WHERE schemaname != 'pg_catalog';
  ```

---

### **Verification**
1. **Test Metadata Access**:
   ```sql
   -- Connect to the database as metadata_user
   \c mydatabase metadata_user

   -- Query metadata (should succeed)
   SELECT tablename FROM pg_catalog.pg_tables WHERE schemaname = 'public';
   ```

2. **Test Data Access**:
   ```sql
   -- Attempt to read table data (should fail)
   SELECT * FROM public.some_table;
   -- ERROR: permission denied for table some_table
   ```

This setup ensures the user can view metadata but cannot access or modify data. Adjust schema names if your tables reside in non-default schemas.
