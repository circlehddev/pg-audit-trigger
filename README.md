A simple, customisable table audit system for PostgreSQL implemented using triggers.

See: http://wiki.postgresql.org/wiki/Audit_trigger_91plus

# Audit trigger 10plus Library Snippets

## Generic audit trigger function (enhanced)

Works with PostgreSQL: 10+
Written in: PL/pgSQL

Here is an example of a generic trigger function used for recording changes to tables into an audit log table. It records quite a bit more detail than the older Audit trigger and does so in a more structured manner.

Row values are recorded as hstore fields rather than as flat text. This allows much more sophisticated querying against the audit history and allows the audit system to record only changed fields for updates.

Auditing can be done coarsely at a statement level or finely at a row level. Control is per-audited-table.

The information recorded is:

- Change type - Insert, Update, Delete or Truncate.
- client IP/port if not a UNIX socket
- Session user name ("real" user name, not effective username from SET ROLE or SECURITY DEFINER)
transaction, statement, and wall clock timestamps
- Top-level statement that caused the change
- The row value before the change (or after in the case of INSERT)
- In the case of UPDATE, the new values of any changed columns. The new value can be reconstructed using row_value || changed_fields
- The transaction ID of the tx that made the change
- The application_name
- Target schema and table, by OID and name.
- This trigger can not track:
    - SELECTs
    - DDL like ALTER TABLE
    - Changes to system catalogs
    - Changes by the table owner and superusers are tracked, but can be trivially tampered with.

If you want this audit log to be trustworthy, your app should run with a role that has at most USAGE to the audit schema and SELECT rights to audit.logged_actions. Most importantly, your app must not connect with a superuser role and must not own the tables it uses. Create your app's schema with a different user to the one your app runs as, and GRANT your app the minimum rights it needs.

The trigger
You can obtain the latest version of the audit trigger 2ndQuadrant/audit-trigger from GitHub.

Basic usage
`SELECT audit.audit_table('target_table_name');`
The table will now have audit events recorded at a row level for every insert/update/delete, and at a statement level for truncate. Query text will always be logged.

To later cancel the auditing:

`DROP TRIGGER audit_trigger_row ON target_table_name;
DROP TRIGGER audit_trigger_stm ON target_table_name;`
If you want finer control use `audit.audit_table(target_table regclass, audit_rows boolean, audit_query_text boolean, excluded_cols text[])` or CREATE TRIGGER the audit trigger yourself. For example:

`SELECT audit.audit_table('target_table_name', 'true', 'false', '{version_col, changed_by, changed_timestamp}'::text[]);`
... would create audit triggers on target_table_name that record each row change but omit the query text (the 'false' argument) and omit the version_col, changed_by and changed_timestamp columns from logged rows. An UPDATE that only changes ignored columns won't result in an audit record being created at all.

Ignoring some changes
In PostgreSQL 9.0 and above, you can add an optional WHEN clause to invocations of the trigger. This is great for auditing, as it lets you exclude some changes without even paying the cost of invoking the trigger.

A common case is to ignore a frequently updated field that isn't worth auditing. Unfortunately, you have to test for a change in all the fields you ARE interested in, there isn't an easy way to say "if anything except fields <x> and <y> have changed between OLD and NEW":

`CREATE TRIGGER tablename_audit_insert_delete
AFTER INSERT OR DELETE ON sometable FOR EACH ROW
EXECUTE PROCEDURE audit.if_modified_func();`
 
`CREATE TRIGGER tablename_audit_update_selective
AFTER UPDATE ON sometable FOR EACH ROW
WHEN ( (OLD.col1, OLD.col2, OLD.col3) IS DISTINCT FROM (NEW.col1, NEW.col2, NEW.col3) )
EXECUTE PROCEDURE audit.if_modified_func();`
... but of course, you can do a lot more. You'll often need to separate your INSERT, DELETE and UPDATE trigger definitions rather than doing them all in one trigger definition, though.

Note the use of IS DISTINCT FROM rather than =. Think about the effect of NULL.

Time zone concerns
TIMESTAMP WITH TIME ZONE fields are always stored in UTC and are displayed in local time by default. You can control display using AT TIME ZONE in queries, or with SET timezone = 'UTC' as a per-session GUC. See the Pg docs.

Loosely based on Audit trigger by bricklen, completely rewritten by ringerc for more audit detail, hstore logging, and more.

