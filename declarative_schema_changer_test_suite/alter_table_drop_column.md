<!-- Acceptance Testing 22.2 Release -->
#  **`ALTER TABLE...DROP COLUMN` Acceptance Testing - v22.2**

## **Results:**
Function | Local Single Node Cluster | CC MR Cluster
:---------------- | :-------------| :-------------|
`ALTER TABLE...DROP COLUMN` | ✅ |  ✅ |

## **Environments**
<!-- Local Single Node Environment -->
### **Local Single Node Environment**
```
Build Tag:        v22.2.0-alpha.1-303-g4dcb32c034
Build Time:       2022/10/11 17:18:44
Distribution:     CCL
Platform:         darwin amd64 (x86_64-apple-darwin21.6.0)
Go Version:       go1.18.4
C Compiler:       Apple LLVM 13.0.0 (clang-1300.0.29.3)
Build Commit ID:  4dcb32c0346e20a95847763f89b9b0796d9ed4dc
Build Type:       development
```

### **Setup**
Use `cockroach demo` to open a SQL shell to a temporary, in-memory cluster with the [MovR database](https://www.cockroachlabs.com/docs/stable/movr.html?) preloaded:
```
cockroach demo movr
```

<!-- CC Staging MR Enviornment -->
### **CC Staging MR Enviornment**
Cluster name: [quiet-kc-schema-test](https://management-staging.crdb.io/cluster/5fab2764-7f93-4198-a76b-0ad25406ade3)
```
CockroachDB version: latest-v22.2-build(sha256:13207e8fb2b4de120171fd23d11f6c98d643a9fdba52cf40903b005b8dc86baf)
3 regions / 9 machines:
Mumbai - (ap-south-1) - 3 nodes
London - (eu-west-2) - 3 nodes
Oregon - (us-west-2) - 3 nodes
Compute - 2 vCPU, 8 GiB RAM ($0.51/hr)
Storage - 15 GiB ($0.008/hr)
Cluster Support ID: 5fab2764-7f93-4198-a76b-0ad25406ade3
```

### **Setup**
1. Load [MovR database and dataset](https://www.cockroachlabs.com/docs/stable/movr.html?):
```
cockroach workload init movr "postgresql://kevin:<PASSWORD>@quiet-kc-schema-test-6qbm.aws-ap-south-1.crdb.io:26257/movr?sslmode=verify-full&sslrootcert=$HOME/Library/CockroachCloud/certs/quiet-kc-schema-test-ca.crt"
```
2. Connect to CC cluster:
```
cockroach sql --url "postgresql://kevin:<PASSWORD>@quiet-kc-schema-test-6qbm.aws-ap-south-1.crdb.io:26257/movr?sslmode=verify-full&sslrootcert=$HOME/Library/CockroachCloud/certs/quiet-kc-schema-test-ca.crt"
```
3. Set MovR as the current database:
```sql
USE movr;
```
## **Test Suite**

### **Initial State**

```
SHOW CREATE TABLE users;
  table_name |                     create_statement
-------------+-----------------------------------------------------------
  users      | CREATE TABLE public.users (
             |     id UUID NOT NULL,
             |     city VARCHAR NOT NULL,
             |     name VARCHAR NULL,
             |     address VARCHAR NULL,
             |     credit_card VARCHAR NULL,
             |     is_awesome BOOL NULL DEFAULT true,
             |     CONSTRAINT users_pkey PRIMARY KEY (city ASC, id ASC)
             | )
(1 row)
```
### **Expected Behavior**
If the schema change is using CRDB's new declarative schema changer, we expect the job of the schema change to be `NEW SCHEMA CHANGE`. You can show just schema change jobs by using SHOW JOBS as the data source for a SELECT statement, and then filtering the job_type value with the WHERE clause:
```
WITH x AS (SHOW JOBS) SELECT * FROM x WHERE job_type = 'NEW SCHEMA CHANGE';
```
## **Single Node Cluster Test Suite**

1. Add column:
```
ALTER TABLE users ADD COLUMN pronouns STRING, ADD COLUMN is_awesome BOOL DEFAULT true;
```
2. Drop column:
```
ALTER TABLE users DROP COLUMN is_awesome;
```
3. Show jobs to validate it used the `NEW SCHEMA CHANGE`:
```
WITH x AS (SHOW JOBS) SELECT * FROM x WHERE job_type = 'NEW SCHEMA CHANGE';
...
  804825538344648705 | NEW SCHEMA CHANGE | ALTER TABLE movr.public.users DROP COLUMN is_awesome                                              | ALTER TABLE movr.public.users DROP COLUMN is_awesome                                              | demo      | succeeded | NULL           | 2022-10-13 17:54:22.413539 | 2022-10-13 17:54:22.45782  | 2022-10-13 17:54:22.616832 | 2022-10-13 17:54:22.615848 |                  1 |       |              1 | 5560824096851323628 | 2022-10-13 17:54:22.45782  | 2022-10-13 17:54:52.45782  |        1 | {}
...
```
## **MR Cluster Test Suite**
1. Add column:
```
ALTER TABLE users ADD COLUMN pronouns STRING, ADD COLUMN is_awesome BOOL DEFAULT true;
```
2. Drop column:
```
ALTER TABLE users DROP COLUMN is_awesome;
```
3. Show jobs to validate it used the `NEW SCHEMA CHANGE`:
```
WITH x AS (SHOW JOBS) SELECT * FROM x WHERE job_type = 'NEW SCHEMA CHANGE';
...
  804827472115531783 | NEW SCHEMA CHANGE | ALTER TABLE movr.public.users DROP COLUMN is_awesome                                              | ALTER TABLE movr.public.users DROP COLUMN is_awesome                                              | kevin     | succeeded | NULL           | 2022-10-13 18:04:11.670546 | 2022-10-13 18:04:13.813986 | 2022-10-13 18:04:35.634554 | 2022-10-13 18:04:35.633204 |                  1 |       |              7 | 7569205178275470005 | 2022-10-13 18:04:13.813986 | 2022-10-13 18:04:43.813986 |        1 | {}
...
```
## **Notes**
