<!-- Acceptance Testing 22.2 Release -->
#  [CREATE INDEX] Acceptance Testing - v22.2

## **Results:**
Function | Local Single Node Cluster | CC MR Cluster
:---------------- | :-------------| :-------------|
`DROP INDEX` | ❌ |  ❌ |

## Setup
<!-- Local Single Node Environment -->
### Local Single Node Environment
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

<!-- CC Staging MR Enviornment -->
### CC Staging MR Enviornment
Cluster name: [quiet-kc-schema-test](https://management-staging.crdb.io/cluster/5fab2764-7f93-4198-a76b-0ad25406ade3)
```
CockroachDB version: latest-v22.2-build(sha256:13207e8fb2b4de120171fd23d11f6c98d643a9fdba52cf40903b005b8dc86baf)

3 regions / 9 machines:
Mumbai - (ap-south-1) - 3 nodes
London - (eu-west-2) - 3 nodes
Oregon - (us-west-2) - 3 nodes
Compute - 2 vCPU, 8 GiB RAM ($0.51/hr)
Storage - 15 GiB ($0.008/hr)
```

<!-- Single Node Cluster Test Suite -->
## Single Node Cluster Test Suite

1. Create a new index:
```
CREATE INDEX ON users (city, id);
```
2. Drop the index:
```
DROP INDEX users@users_city_idx;
```
1. Show jobs after you drop index to see if it used the `NEW SCHEMA CHANGE`:
```
WITH x AS (SHOW JOBS) SELECT * FROM x WHERE job_type = 'NEW SCHEMA CHANGE';
...

...
```
4. Use `SCHEMA CHANGE` in jobs search to find it:
```
...
  807099454268375041 | SCHEMA CHANGE | DROP INDEX movr.public.users@users_city_idx                               |           | demo      | succeeded | NULL           | 2022-10-21 18:40:06.482996 | 2022-10-21 18:40:06.529877 | 2022-10-21 18:40:06.593993 | 2022-10-21 18:40:06.592837 |                  1 |       |              1 | 5101843398805637073 | 2022-10-21 18:40:06.529877 | 2022-10-21 18:40:36.529877 |        1 | {}
...
```
<!-- MR Cluster Test Suite -->
## MR Cluster Test Suite
1. Create a new index:
```
CREATE INDEX ON users (city, id);
```
2. Drop the index:
```
DROP INDEX users@users_city_idx;
```
1. Show jobs after you drop index to see if it used the `NEW SCHEMA CHANGE`:
```
WITH x AS (SHOW JOBS) SELECT * FROM x WHERE job_type = 'NEW SCHEMA CHANGE';
...

...
```
4. Use `SCHEMA CHANGE` in jobs search to find it:
```
...
  807099454268375041 | SCHEMA CHANGE | DROP INDEX movr.public.users@users_city_idx                               |           | demo      | succeeded | NULL           | 2022-10-21 18:40:06.482996 | 2022-10-21 18:40:06.529877 | 2022-10-21 18:40:06.593993 | 2022-10-21 18:40:06.592837 |                  1 |       |              1 | 5101843398805637073 | 2022-10-21 18:40:06.529877 | 2022-10-21 18:40:36.529877 |        1 | {}
...
```

<!-- Notes -->
## Notes
