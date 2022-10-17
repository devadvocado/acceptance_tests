<!-- Acceptance Testing 22.2 Release -->
#  [CREATE INDEX] Acceptance Testing - v22.2

## **Results:**
Function | Local Single Node Cluster | CC MR Cluster
:---------------- | :-------------| :-------------|
`CREATE INDEX` | ❌ |  ❌ |

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

1. Create a single-column index:
```
CREATE INDEX ON users (city);
```
2. Show jobs to validate it used the old `SCHEMA CHANGE`:
```
WITH x AS (SHOW JOBS) SELECT * FROM x WHERE job_type = 'SCHEMA CHANGE';
...
  805890464348438529 | SCHEMA CHANGE | CREATE INDEX ON movr.public.users (city)                                                                                   |           | demo      | succeeded | NULL           | 2022-10-17 12:10:52.054434 | 2022-10-17 12:10:52.331817 | 2022-10-17 12:10:55.30114  | 2022-10-17 12:10:55.2998   |                  1 |       |              1 | 1829792971658967280 | 2022-10-17 12:10:52.331818 | 2022-10-17 12:11:22.331818 |        1 | {}
...
```
3.  Turn on `set use_declarative_schema_changer = unsafe_always` to see if it would work on new declarative schema changer:
```
SET use_declarative_schema_changer = unsafe_always;
```
4. Show jobs after you create another index to see if it used the `NEW SCHEMA CHANGE`:
```
WITH x AS (SHOW JOBS) SELECT * FROM x WHERE job_type = 'NEW SCHEMA CHANGE';
...
  805891236319002625 | NEW SCHEMA CHANGE | CREATE INDEX ON movr.public.users (city)       | CREATE INDEX ON movr.public.users (city)       | demo      | failed | NULL           | 2022-10-17 12:14:47.633499 | 2022-10-17 12:14:48.377886 | 2022-10-17 12:14:51.884916 | 2022-10-17 12:14:51.605869 |                  1 | error executing PostCommitNonRevertiblePhase stage 1 of 2 with 5 MutationType ops: relation "users" (106): empty index name |              1 |   23480148625347058 | 2022-10-17 12:14:50.834283 | 2022-10-17 12:15:20.834283 |        1 | {}
  ...
```
<!-- MR Cluster Test Suite -->
## MR Cluster Test Suite

1. Create a single-column index:
```
CREATE INDEX ON users (city);
```
2. Show jobs to validate it used the old `SCHEMA CHANGE`:
```
WITH x AS (SHOW JOBS) SELECT * FROM x WHERE job_type = 'SCHEMA CHANGE';
...
  805891622248742913 | SCHEMA CHANGE | CREATE INDEX ON movr.public.users (city)                                                                                   |           | demo      | succeeded | NULL           | 2022-10-17 12:16:45.418903 | 2022-10-17 12:16:45.434142 | 2022-10-17 12:16:45.532374 | 2022-10-17 12:16:45.531614 |                  1 |       |              1 | 7017263389401716096 | 2022-10-17 12:16:45.434142 | 2022-10-17 12:17:15.434142 |        1 | {}
...
```
3.  Turn on `set use_declarative_schema_changer = unsafe_always` to see if it would work on new declarative schema changer:
```
SET use_declarative_schema_changer = unsafe_always;
```
4. Show jobs after you create another index to see if it used the `NEW SCHEMA CHANGE`:
```
WITH x AS (SHOW JOBS) SELECT * FROM x WHERE job_type = 'NEW SCHEMA CHANGE';
...
  805891842205417473 | NEW SCHEMA CHANGE | CREATE INDEX ON movr.public.users (city) | CREATE INDEX ON movr.public.users (city) | demo      | failed | NULL           | 2022-10-17 12:17:52.523261 | 2022-10-17 12:17:52.562104 | 2022-10-17 12:17:52.68218 | 2022-10-17 12:17:52.665252 |                  1 | error executing PostCommitNonRevertiblePhase stage 1 of 2 with 5 MutationType ops: relation "users" (106): empty index name |              1 | 9197349550564429527 | 2022-10-17 12:17:52.645831 | 2022-10-17 12:18:22.645831 |        1 | {}
  ...
```
<!-- Notes -->
## Notes

Error that occurs when `use_declarative_schema_changer` set to `unsafe` or `unsafe_always`
```
demo@127.0.0.1:26257/movr> CREATE INDEX ON users (city);
ERROR: internal error: error executing PostCommitNonRevertiblePhase stage 1 of 2 with 5 MutationType ops: relation "users" (106): empty index name
SQLSTATE: XX000
DETAIL: • Schema change plan for CREATE INDEX ON ‹movr›.public.‹users› (‹city›);
│
├── • PostCommitPhase
│   │
│   ├── • Stage 1 of 7 in PostCommitPhase
│   │   │
│   │   ├── • 1 element transitioning toward TRANSIENT_ABSENT
│   │   │   │
│   │   │   └── • TemporaryIndex:{DescID: 106, IndexID: 13, ConstraintID: 0, SourceIndexID: 1}
│   │   │       │ DELETE_ONLY → WRITE_ONLY
│   │   │       │
│   │   │       └── • PreviousTransactionPrecedence dependency from DELETE_ONLY TemporaryIndex:{DescID: 106, IndexID: 13, ConstraintID: 0, SourceIndexID: 1}
│   │   │             rule: "TemporaryIndex transitions to TRANSIENT_ABSENT uphold 2-version invariant: DELETE_ONLY->WRITE_ONLY"
│   │   │
│   │   └── • 3 Mutation operations
│   │       │
│   │       ├── • MakeAddedIndexDeleteAndWriteOnly
│   │       │     IndexID: 13
│   │       │     TableID: 106
│   │       │
│   │       ├── • SetJobStateOnDescriptor
│   │       │     DescriptorID: 106
│   │       │
│   │       └── • UpdateSchemaChangerJob
│   │             JobID: 805891236319002625
│   │             RunningStatus: PostCommitPhase stage 2 of 7 with 1 BackfillType op pending
│   │
│   ├── • Stage 2 of 7 in PostCommitPhase
│   │   │
│   │   ├── • 1 element transitioning toward PUBLIC
│   │   │   │
│   │   │   └── • SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
│   │   │       │ BACKFILL_ONLY → BACKFILLED
│   │   │       │
│   │   │       ├── • Precedence dependency from PUBLIC IndexColumn:{DescID: 106, ColumnID: 2, IndexID: 12}
│   │   │       │     rule: "index-column added to index before index is backfilled"
│   │   │       │
│   │   │       ├── • Precedence dependency from PUBLIC IndexColumn:{DescID: 106, ColumnID: 1, IndexID: 12}
│   │   │       │     rule: "index-column added to index before index is backfilled"
│   │   │       │
│   │   │       ├── • PreviousTransactionPrecedence dependency from BACKFILL_ONLY SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
│   │   │       │     rule: "SecondaryIndex transitions to PUBLIC uphold 2-version invariant: BACKFILL_ONLY->BACKFILLED"
│   │   │       │
│   │   │       └── • Precedence dependency from WRITE_ONLY TemporaryIndex:{DescID: 106, IndexID: 13, ConstraintID: 0, SourceIndexID: 1}
│   │   │             rule: "temp index is WRITE_ONLY before backfill"
│   │   │
│   │   └── • 1 Backfill operation
│   │       │
│   │       └── • BackfillIndex
│   │             IndexID: 12
│   │             SourceIndexID: 1
│   │             TableID: 106
│   │
│   ├── • Stage 3 of 7 in PostCommitPhase
│   │   │
│   │   ├── • 1 element transitioning toward PUBLIC
│   │   │   │
│   │   │   └── • SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
│   │   │       │ BACKFILLED → DELETE_ONLY
│   │   │       │
│   │   │       └── • PreviousTransactionPrecedence dependency from BACKFILLED SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
│   │   │             rule: "SecondaryIndex transitions to PUBLIC uphold 2-version invariant: BACKFILLED->DELETE_ONLY"
│   │   │
│   │   └── • 3 Mutation operations
│   │       │
│   │       ├── • MakeBackfillingIndexDeleteOnly
│   │       │     IndexID: 12
│   │       │     TableID: 106
│   │       │
│   │       ├── • SetJobStateOnDescriptor
│   │       │     DescriptorID: 106
│   │       │
│   │       └── • UpdateSchemaChangerJob
│   │             JobID: 805891236319002625
│   │             RunningStatus: PostCommitPhase stage 4 of 7 with 1 MutationType op pending
│   │
│   ├── • Stage 4 of 7 in PostCommitPhase
│   │   │
│   │   ├── • 1 element transitioning toward PUBLIC
│   │   │   │
│   │   │   └── • SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
│   │   │       │ DELETE_ONLY → MERGE_ONLY
│   │   │       │
│   │   │       └── • PreviousTransactionPrecedence dependency from DELETE_ONLY SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
│   │   │             rule: "SecondaryIndex transitions to PUBLIC uphold 2-version invariant: DELETE_ONLY->MERGE_ONLY"
│   │   │
│   │   └── • 3 Mutation operations
│   │       │
│   │       ├── • MakeBackfilledIndexMerging
│   │       │     IndexID: 12
│   │       │     TableID: 106
│   │       │
│   │       ├── • SetJobStateOnDescriptor
│   │       │     DescriptorID: 106
│   │       │
│   │       └── • UpdateSchemaChangerJob
│   │             JobID: 805891236319002625
│   │             RunningStatus: PostCommitPhase stage 5 of 7 with 1 BackfillType op pending
│   │
│   ├── • Stage 5 of 7 in PostCommitPhase
│   │   │
│   │   ├── • 1 element transitioning toward PUBLIC
│   │   │   │
│   │   │   └── • SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
│   │   │       │ MERGE_ONLY → MERGED
│   │   │       │
│   │   │       └── • PreviousTransactionPrecedence dependency from MERGE_ONLY SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
│   │   │             rule: "SecondaryIndex transitions to PUBLIC uphold 2-version invariant: MERGE_ONLY->MERGED"
│   │   │
│   │   └── • 1 Backfill operation
│   │       │
│   │       └── • MergeIndex
│   │             BackfilledIndexID: 12
│   │             TableID: 106
│   │             TemporaryIndexID: 13
│   │
│   ├── • Stage 6 of 7 in PostCommitPhase
│   │   │
│   │   ├── • 1 element transitioning toward PUBLIC
│   │   │   │
│   │   │   └── • SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
│   │   │       │ MERGED → WRITE_ONLY
│   │   │       │
│   │   │       └── • PreviousTransactionPrecedence dependency from MERGED SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
│   │   │             rule: "SecondaryIndex transitions to PUBLIC uphold 2-version invariant: MERGED->WRITE_ONLY"
│   │   │
│   │   └── • 3 Mutation operations
│   │       │
│   │       ├── • MakeMergedIndexWriteOnly
│   │       │     IndexID: 12
│   │       │     TableID: 106
│   │       │
│   │       ├── • SetJobStateOnDescriptor
│   │       │     DescriptorID: 106
│   │       │
│   │       └── • UpdateSchemaChangerJob
│   │             JobID: 805891236319002625
│   │             RunningStatus: PostCommitPhase stage 7 of 7 with 1 ValidationType op pending
│   │
│   └── • Stage 7 of 7 in PostCommitPhase
│       │
│       ├── • 1 element transitioning toward PUBLIC
│       │   │
│       │   └── • SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
│       │       │ WRITE_ONLY → VALIDATED
│       │       │
│       │       └── • PreviousTransactionPrecedence dependency from WRITE_ONLY SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
│       │             rule: "SecondaryIndex transitions to PUBLIC uphold 2-version invariant: WRITE_ONLY->VALIDATED"
│       │
│       └── • 1 Validation operation
│           │
│           └── • ValidateUniqueIndex
│                 IndexID: 12
│                 TableID: 106
│
└── • PostCommitNonRevertiblePhase
    │
    ├── • Stage 1 of 2 in PostCommitNonRevertiblePhase
    │   │
    │   ├── • 2 elements transitioning toward PUBLIC
    │   │   │
    │   │   ├── • SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
    │   │   │   │ VALIDATED → PUBLIC
    │   │   │   │
    │   │   │   ├── • Precedence dependency from PUBLIC IndexColumn:{DescID: 106, ColumnID: 2, IndexID: 12}
    │   │   │   │     rule: "index dependents exist before index becomes public"
    │   │   │   │
    │   │   │   ├── • Precedence dependency from PUBLIC IndexColumn:{DescID: 106, ColumnID: 1, IndexID: 12}
    │   │   │   │     rule: "index dependents exist before index becomes public"
    │   │   │   │
    │   │   │   ├── • PreviousTransactionPrecedence dependency from VALIDATED SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
    │   │   │   │     rule: "SecondaryIndex transitions to PUBLIC uphold 2-version invariant: VALIDATED->PUBLIC"
    │   │   │   │
    │   │   │   └── • SameStagePrecedence dependency from PUBLIC IndexName:{DescID: 106, Name: , IndexID: 12}
    │   │   │         rule: "index dependents exist before index becomes public"
    │   │   │         rule: "index named right before index becomes public"
    │   │   │
    │   │   └── • IndexName:{DescID: 106, Name: , IndexID: 12}
    │   │       │ ABSENT → PUBLIC
    │   │       │
    │   │       └── • Precedence dependency from BACKFILL_ONLY SecondaryIndex:{DescID: 106, IndexID: 12, ConstraintID: 0, TemporaryIndexID: 13, SourceIndexID: 1}
    │   │             rule: "index existence precedes index dependents"
    │   │
    │   ├── • 1 element transitioning toward TRANSIENT_ABSENT
    │   │   │
    │   │   └── • TemporaryIndex:{DescID: 106, IndexID: 13, ConstraintID: 0, SourceIndexID: 1}
    │   │       │ WRITE_ONLY → TRANSIENT_DELETE_ONLY
    │   │       │
    │   │       └── • PreviousTransactionPrecedence dependency from WRITE_ONLY TemporaryIndex:{DescID: 106, IndexID: 13, ConstraintID: 0, SourceIndexID: 1}
    │   │             rule: "TemporaryIndex transitions to TRANSIENT_ABSENT uphold 2-version invariant: WRITE_ONLY->TRANSIENT_DELETE_ONLY"
    │   │
    │   └── • 5 Mutation operations
    │       │
    │       ├── • SetIndexName
    │       │     IndexID: 12
    │       │     TableID: 106
    │       │
    │       ├── • MakeDroppedIndexDeleteOnly
    │       │     IndexID: 13
    │       │     TableID: 106
    │       │
    │       ├── • MakeAddedSecondaryIndexPublic
    │       │     IndexID: 12
    │       │     TableID: 106
    │       │
    │       ├── • SetJobStateOnDescriptor
    │       │     DescriptorID: 106
    │       │
    │       └── • UpdateSchemaChangerJob
    │             IsNonCancelable: true
    │             JobID: 805891236319002625
    │             RunningStatus: PostCommitNonRevertiblePhase stage 2 of 2 with 2 MutationType ops pending
    │
    └── • Stage 2 of 2 in PostCommitNonRevertiblePhase
        │
        ├── • 1 element transitioning toward TRANSIENT_ABSENT
        │   │
        │   └── • TemporaryIndex:{DescID: 106, IndexID: 13, ConstraintID: 0, SourceIndexID: 1}
        │       │ TRANSIENT_DELETE_ONLY → TRANSIENT_ABSENT
        │       │
        │       └── • PreviousTransactionPrecedence dependency from TRANSIENT_DELETE_ONLY TemporaryIndex:{DescID: 106, IndexID: 13, ConstraintID: 0, SourceIndexID: 1}
        │             rule: "TemporaryIndex transitions to TRANSIENT_ABSENT uphold 2-version invariant: TRANSIENT_DELETE_ONLY->TRANSIENT_ABSENT"
        │
        └── • 4 Mutation operations
            │
            ├── • CreateGcJobForIndex
            │     IndexID: 13
            │     TableID: 106
            │
            ├── • MakeIndexAbsent
            │     IndexID: 13
            │     TableID: 106
            │
            ├── • RemoveJobStateFromDescriptor
            │     DescriptorID: 106
            │     JobID: 805891236319002625
            │
            └── • UpdateSchemaChangerJob
                  IsNonCancelable: true
                  JobID: 805891236319002625
                  RunningStatus: all stages completed

--
stages graphviz: https://cockroachdb.github.io/scplan/viz.html#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
--
dependencies graphviz: https://cockroachdb.github.io/scplan/viz.html#H4sIAAAAAAAA/9RZ727aSBD/DE+x8qdeFVJsCGnSYIkEUnFNSRVI/yiqosU7JVbM2re79Jo7VcprNY+TJzl5/d841NS26vvIzux49/ebmZ0ZiLlg2LlB6N9mg6/m3g/DWnEB7Jqr7nLGekeuNyw8B6uvcIEFLIEKrrxyl2n36otNBTf/gb5yoOx4akdHAs8tQIaFOe8r9koAU/QjwfQjQYLVG8DEXeaGM9+dBnaPXgiiH72QqlJdT9qSP0Jb+gUQbAggqf2C6CcXo8FshMaT4egjOp+gx/sfS/sre7x/2HVWc8s0dh/vf6w4MP54/4CePd7/MExx93j/8Mf6EX5uXRpCz1wTm/bP8CLTRGzLC3lDPVppBktNfYffYAf6CrUpKJ8lAc1G43smb1qCN4HZAkLS9gqT9lyyNqYEvp3Y1mpJt+TNkJvGJERDWzdgutZjKmqGjjQb12n3fh1L2qsfLuqv4XJr0kjhzejT9fTy9HT8sWoA90sCcAqGTQlmdxLJLTHMgw+3V8yAcVpzW3wCJVg6NguOG9fuFADzZZneOMFLqADHct3noKQbzxJ8FL12JyME+SU36SJ00xE1bGLSKLkLtsqAuySv2xZWtV2/vJYn32cAXzIwav2AyZPwM4D5PQnfrz78IoOA41UYVNWu/LX24bvL47PxibeBqp1AoKYE3UCgHR4PTt6cjs/ORkNfqKnrwuvzydmnQN6O5MPR2Wg2ikvV/Uj6dnTxOjSrHqQEiV0vI2HypL1I8H5wNh4OZpHFvUj24WKcPIcW3r1zODiejiazQKBFgsSntNBcN+NeWi+Szi4Gk+l4NJldJy3vZ6lkmOpGeuvnDpHYSx0vxK+Xgkhr6S6Ahm3ZrK8wIEFsKdKXEQEHKHHbCATfTC7QHL7YDJAnnYNhL4Ejr05P2Ow+abPlRRLChABBwg5txQybHM2xcfvFtCwgoUdWcNZORWfttnTX2YsmravP3LCd3XNny2TVDrPGc2ngLb6FY/+QJl3ILDgECwScU+tOKjd/atR7Docby4yZTFrDvElrYxpLQplBUbL4RIJhyk1h2pS7VHmOjlbOjW0RpLW+AuOmTZFJv2JmYioOUZS/WnpGrKmec9SFxIDAqFYqh7NprNgZbnreyidXQqxpT8e0DGSgBiCHgQEEOEoHe4KtTnmmSjxVZoIpzXulz7b09ce47X67eKleRRICIm/9FtgiqMbrmoACGKugMJZ0Wvp6ZbPvfrn4DKhMAl3GfPI+MPP/8H74IFZBn1ektvS1Skw9cD9afMhSGnMua9u8G6k4LcRaoJScvQxzzl7yUOyjXRnF8QANO4ieV47WhmJ8CwO3TE1e+Z0sd+sdob0nC/vC9IV9X0tPNj577jeLj5HLYu89tkyCBVxS86/VVpH6mzjz8auCsyibtvR03651vIKxLrxNQSSntnUlTPM7fcyY/beLQl8hJl7alCg7GSRebd9Sv/JXKV4CQcxc3GxU/xyOTlxG69NlhXk01iQPKClc7GSMAitgWYKZWasKWDpo6/ZF2stMzQXsZbZDyfIgnTbSY7P8VfX6lMwtCLX6pP4TBljAa+NPe35qs0KpvwQf87epGVEhPzeY8+Dv9doGgU9wtU6WOaVt6U/Nd+UUq0YNAb6FIbMdJ5Hn6p/fuk+OU2L5yOSxGiJ4g4IZbYqQap0kXspsGOs3G9+bjWaD7nlzp+BGQbWk7HBxZ0FfIZjf+DNm2vPGXbl0ZUCo+XRfuro57bp9V/TXRwY4mbvUtveo5PqEqnovWh7l5vfmfwEAAP//zDVdbTslAAA=stack trace:
github.com/cockroachdb/cockroach/pkg/sql/catalog/errors.go:24: ValidateName()
github.com/cockroachdb/cockroach/pkg/sql/catalog/tabledesc/validate.go:1138: validateTableIndexes()
github.com/cockroachdb/cockroach/pkg/sql/catalog/tabledesc/validate.go:641: ValidateSelf()
github.com/cockroachdb/cockroach/pkg/sql/catalog/internal/validate/validate.go:71: func1()
github.com/cockroachdb/cockroach/pkg/sql/catalog/internal/validate/validate.go:178: validateDescriptorsAtLevel()
github.com/cockroachdb/cockroach/pkg/sql/catalog/internal/validate/validate.go:67: Validate()
github.com/cockroachdb/cockroach/pkg/sql/catalog/internal/validate/validate.go:41: Self()
github.com/cockroachdb/cockroach/pkg/sql/catalog/descs/collection.go:284: WriteDescToBatch()
github.com/cockroachdb/cockroach/pkg/sql/schemachanger/scdeps/exec_deps.go:229: CreateOrUpdateDescriptor()
github.com/cockroachdb/cockroach/pkg/sql/schemachanger/scexec/exec_mutation.go:106: func2()
github.com/cockroachdb/cockroach/pkg/sql/catalog/nstree/by_id_map.go:44: func1()
github.com/cockroachdb/cockroach/pkg/sql/catalog/nstree/tree.go:71: func1()
github.com/google/btree/external/com_github_google_btree/btree.go:524: iterate()
github.com/google/btree/external/com_github_google_btree/btree.go:777: Ascend()
github.com/cockroachdb/cockroach/pkg/sql/catalog/nstree/tree.go:70: ascend()
github.com/cockroachdb/cockroach/pkg/sql/catalog/nstree/by_id_map.go:43: ascend()
github.com/cockroachdb/cockroach/pkg/sql/catalog/nstree/id_map.go:71: Iterate()
github.com/cockroachdb/cockroach/pkg/sql/schemachanger/scexec/exec_mutation.go:105: performBatchedCatalogWrites()
github.com/cockroachdb/cockroach/pkg/sql/schemachanger/scexec/exec_mutation.go:60: executeDescriptorMutationOps()
github.com/cockroachdb/cockroach/pkg/sql/schemachanger/scexec/executor.go:32: ExecuteStage()
github.com/cockroachdb/cockroach/pkg/sql/schemachanger/scrun/scrun.go:175: executeStage()
github.com/cockroachdb/cockroach/pkg/sql/schemachanger/scrun/scrun.go:134: func3()
github.com/cockroachdb/cockroach/pkg/sql/schemachanger/scdeps/run_deps.go:141: func1()
github.com/cockroachdb/cockroach/pkg/sql/catalog/descs/txn.go:52: func1()
github.com/cockroachdb/cockroach/pkg/sql/catalog/descs/txn.go:163: func3()
github.com/cockroachdb/cockroach/pkg/kv/db.go:963: func1()
github.com/cockroachdb/cockroach/pkg/kv/txn.go:960: exec()
github.com/cockroachdb/cockroach/pkg/kv/db.go:962: runTxn()
github.com/cockroachdb/cockroach/pkg/kv/db.go:925: TxnWithAdmissionControl()
github.com/cockroachdb/cockroach/pkg/kv/db.go:904: Txn()
github.com/cockroachdb/cockroach/pkg/sql/catalog/descs/txn.go:155: TxnWithExecutor()
github.com/cockroachdb/cockroach/pkg/sql/catalog/descs/txn.go:49: Txn()

HINT: You have encountered an unexpected error.

Please check the public issue tracker to check whether this problem is
already tracked. If you cannot find it there, please report the error
with details by creating a new issue.

If you would rather not post publicly, please contact us directly
using the support form.

We appreciate your feedback.
```