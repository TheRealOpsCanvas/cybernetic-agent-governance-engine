# OpsCanvas (`provider_09`): Round 1 Response

**Responds to:** [`DECISION_POINTS.md`](DECISION_POINTS.md) (Round 1, published 2026-10-04)
**Fixtures:** [`tests/fixtures/provider_09/DP-01.json`](../../../tests/fixtures/provider_09/DP-01.json) through `DP-10.json`
**Schema:** [`estate_decision_response.schema.json`](../../../schemas/provider_09/estate_decision_response.schema.json) v1.0, unchanged

This response reports what OpsCanvas supplies to the estate seam today and what we plan to supply.
We treat the ten decision points as our roadmap for the seam.

## 1. How to read the fixtures

Each fixture holds the values the adapter would return. The adapter is not built yet.

We describe each decision point and each field with two values.

**Fit** uses the v1.0 `fit` values exactly, and describes OpsCanvas **today**. For a decision point
it is the fixture's `fit`; for a field it is given in the fixture's `gaps` and in section 3.

| Fit | For a decision point | For a field |
|---|---|---|
| `NATIVE` | Its key predicates are answered completely, read directly. | Answered completely, read directly. |
| `DERIVED` | Its key predicates are answered completely, computed from other data. | Answered completely, computed from other data. |
| `PARTIAL` | Some of its key predicates are answered. | Answered in some cases, such as for one provider only, or incompletely, such as a lower bound. |
| `NOT_AVAILABLE` | None of its key predicates are answered. | Not answered. |

**Roadmap** says what happens to whatever is not available today:

| Roadmap | Meaning |
|---|---|
| `PLANNED` | On our roadmap. |
| `NOT_PLANNED` | Not on our roadmap. |

`NATIVE` and `DERIVED` mean complete, with no further work expected, so they carry no roadmap.
`PARTIAL` and `NOT_AVAILABLE` always carry one. A derived answer that is incomplete, such as a
lower bound, is `PARTIAL` and `PLANNED`. Every roadmap in this response is `PLANNED`, except
`estimated_affected_users`, which is `NOT_PLANNED` (section 3).

The v1.0 schema has no place for a roadmap, so each entry in a fixture's `gaps` starts with the
field, its fit and its roadmap, for example
`"resource_urn: PARTIAL, PLANNED. Resolving by provider ID is available for AWS today and planned
for GCP and Azure."`. We propose a `roadmap` field for v1.1 (section 6).

### Values for fields we cannot answer

In v1.0, only three predicate fields may be `null`: `last_iac_commit_hash`,
`compensating_rollback_action` and `error`. Every other field must carry a value of its type, even
when we have no answer. So a value in a v1.0 predicate can mean one of three things:

- **an answer**, for a field that applies to the resource and that we answer;
- **not applicable**, for a field that does not apply to the resource;
- **a placeholder for unknown**, for a field that applies but that we do not answer yet
  (fit `NOT_AVAILABLE`). Each such field is listed in the fixture's `gaps`.

Policy cannot tell these apart from the value. In our fixtures we choose each placeholder so that
CAGE defers or denies rather than allowing on a value nobody observed. Some types have no strict
value, and their placeholder points the permissive way:

| Type | Placeholder in our v1.0 fixtures | Strict? |
|---|---|---|
| Enum | The strictest member: `TIER_0`, `PRODUCTION`, `IRREVERSIBLE_TERMINAL`, `DRIFT_DETECTED` (meaning unverified) | Yes |
| Boolean | The strictest value for that field: `is_load_bearing: true`, `has_failover_redundancy: false` | Yes |
| Number | The strictest value for that field: `estimated_blast_radius_score: 1.0` | Yes |
| Integer | `0` | No. `0` reads as none: no dependents, no flows. |
| String | A sentinel: `owner_team_urn: urn:opscanvas:team:unassigned` | No. It reads as a real value. |
| String or null | `null` | Yes |

The per-class fields we propose (section 5) make this worse, because a value of the type's default
(`0`, `false`, `""`) is often the permissive one: `bucket_public_access: false` reads as private,
and `volume_attached_to: ""` reads as unattached, the case that is safe to delete.

We propose two rules for v1.1 (section 6):

- **Unknown is `null`.** Every predicate field becomes nullable, with `null` meaning a field that
  applies but is unknown. Every field stays required, and policy fails closed on `null`.
- **Defaults only where a field does not apply.** If one sparse document carries every class's
  fields (section 5, option B), a field that does not apply to the resource's class carries its
  type's default: `0`, `false`, `""`, `[]`, or an enum's `UNKNOWN` member. Rules over those fields
  must be scoped by `resource_class`, so a default is never read. A default never stands in for
  unknown.

Fixture URNs are GCP full resource names. Where a field is available for AWS today and planned for
GCP and Azure, the fixture's `gaps` say so.

## 2. Answers to Q1 to Q7

### Q1. Resource URN

> What is your canonical resource URN scheme, and does it stay stable when resources are renamed or re-parented?

`resource_urn` is the provider's own full resource ID:

| Provider | Form |
|---|---|
| GCP | Full resource name, `//sqladmin.googleapis.com/projects/{project}/instances/{instance}` |
| Azure | Resource ID, `/subscriptions/{id}/resourceGroups/{rg}/providers/{namespace}/{type}/{name}` |
| AWS | ARN |

This is what an agent's tool call names, and it also names resources created in a console, which
have no IaC declaration. The adapter resolves the URN to the OpsCanvas model (IaC declaration,
environment, dependents) when it answers. A resource declared in a scanned repository resolves for
any provider, for the resource types we record today; resolving by provider ID alone is available
for AWS today and planned for GCP and Azure.

This is the Deployed Resource key in [Operational Estate](https://github.com/TheRealOpsCanvas/operational-estate), an open ontology of a
software estate: "its provider and the provider's id for it" (section 6, item 10).

Stability: a rename that keeps the resource keeps the URN. A recreated resource gets a new URN,
which is correct. Some moves change the provider ID, such as a GCP resource moved to another project
or an Azure resource moved to another resource group, so CAGE sees a new URN. OpsCanvas could later
carry history across such a move.

### Q2. Resource not in the graph

> How should "resource not in graph" be signalled? Should the seam add `NOT_FOUND`, or reuse `INVALID`?

Add two statuses rather than reuse `INVALID`:

- `NOT_FOUND`: the resource does not exist in an account the estate covers.
- `NOT_COVERED`: the resource is in an account or of a type the estate does not cover.

Both fail closed, but they mean different things. `NOT_FOUND` on a mutation is suspicious;
`NOT_COVERED` is a coverage gap to fix. Until v1.1, DP-09 returns `INVALID` with a `not_found:`
prefix on `error`.

The same distinction is drawn in [Operational Estate](https://github.com/TheRealOpsCanvas/operational-estate) (section 8 of its
specification): an answer about something the estate does not hold says whether the source was read
and the thing is not there, or nothing that would show it was read.

### Q3. Freshness and "confirmed"

> What freshness guarantee does a snapshot carry (`ttl_seconds`), and how is "confirmed" defined per edge?

"Confirmed" means a practitioner reviewed and committed that node or edge. Each one carries
`confirmed_at` and a provenance of `extracted` (read from a cited source), `inferred` (proposed by a
model), or `operator` (stated by a person). These map naturally onto the per-field provenance
proposal already queued for v1.1. We do not record deploy time.

We suggest TTL be a policy choice per field rather than one window per snapshot. An environment tier
rarely changes, while a dependents count can change with any deploy.

### Q4. Snapshot hash and signing

> Is `snapshot_hash` computed over a canonical serialisation (e.g. RFC 8785 JCS)? Can snapshots be **signed** with a key resolvable by `kid` from a manifest CAGE fetches independently?

Not today. Each fact carries a sha256 content hash over its RFC 8785 JCS serialization. There is no
whole-graph snapshot hash yet: that needs a stable graph position, which we plan to provide through
an append-only change history. Signing with a `kid` that CAGE resolves from a manifest it fetches
independently is feasible but not built.

In the fixtures, `snapshot_hash` is sha256 over the JCS array of the content hashes of the facts an
answer read. It shows which facts were used, not which graph state, and it is not signed.

### Q5. Transport and latency

> Transport: MCP over stdio, SSE or streamable HTTP? What P99 latency should we expect for a predicate query?

MCP over stdio, local to the practitioner, today. There is no hosted endpoint yet, so a gateway
cannot reach OpsCanvas over the network. We have not measured latency. We propose latency budgets per
domain rather than one budget across the seam.

### Q6. Sandbox estate

> Can we get a sandbox estate for over-the-wire conformance tests?

Yes, once a hosted endpoint exists. It needs an estate seeded in an account OpsCanvas owns, reachable
by CAGE over the wire.

### Q7. Blast radius score

> How is `estimated_blast_radius_score` composed?

We do not compute one, and we propose that CAGE does. OpsCanvas supplies the facts of the impact;
CAGE does the math.

We propose that `query_blast_radius` return the resources an action affects, not only counts. Each
entry would carry:

| Field | Meaning |
|---|---|
| `resource_urn` | The affected resource. |
| `resource_class` | Its class, such as `DATABASE` or `STORAGE_VOLUME` (section 5). |
| `relation` | How it is affected, as an Operational Estate relation such as `connectsTo` or `runsOn` (section 6, item 10). |
| `depth` | Hops from the target of the action. |
| `environment_tier`, `criticality_tier` | As in the predicate. |
| `exposure` | `PUBLIC` if the resource is reachable from the internet, `INTERNAL` otherwise. |
| `provenance` | Whether the entry is observed, derived, or a placeholder. |

Further details per entry, such as cost, are to be agreed. We do not propose a count of affected
users: it cannot be known from the estate, and `exposure` gives policy the part that can be.

CAGE computes the score from this list, and a human reviewer sees the list behind it rather than a
number they cannot explain. We would not send a score of our own as well: two scores for one action
invite a disagreement nobody can settle in review. If you want a provider score as a cross-check, we
can add one later.

The affected list depends on the action's parameters, not only its verb: scaling to 0 affects more
than scaling to 1. This is why we also propose per-verb parameters on `query_blast_radius`
(section 6).

In v1.0, `estimated_affected_services` is the length of that list, and
`estimated_blast_radius_score` is the strict placeholder `1.0`.

## 3. Field coverage

### Shared fields (`CloudOpsResourcePredicate`, v1.0)

| Field | Fit | Roadmap | What the value means |
|---|---|---|---|
| `status` | NATIVE |  | `STALE` is derived: computed from when each fact was last confirmed (Q3). |
| `resource_urn` | PARTIAL | PLANNED | Provider-native ID (Q1). Resolved to a declared resource for any provider, for the resource types we record today; resolved by provider ID for AWS today, GCP and Azure planned. |
| `resource_type` | NATIVE |  | The provider's resource type. |
| `is_load_bearing` | NOT_AVAILABLE | PLANNED | From inbound dependents and environment tier. |
| `direct_dependents_count` | NOT_AVAILABLE | PLANNED | Today, only workloads that run on a resource are known (see `BlastRadiusEstimate`). |
| `transitive_dependency_depth` | NOT_AVAILABLE | PLANNED | Follows from direct dependents. |
| `has_failover_redundancy` | NOT_AVAILABLE | PLANNED | From the per-class fields below, starting with databases. |
| `active_network_flows_last_1h` | NOT_AVAILABLE | PLANNED | Needs network flow data from the customer's account. |
| `environment_tier` | NOT_AVAILABLE | PLANNED | Proposed by OpsCanvas and confirmed by a practitioner. Never inferred from a name. |
| `criticality_tier` | NOT_AVAILABLE | PLANNED | From a cited service catalog where one exists, otherwise from a versioned rule, reported with its inputs. |
| `owner_team_urn` | NOT_AVAILABLE | PLANNED | The team that owns the resource. |
| `managed_by_terraform` | PARTIAL | PLANNED | `true` for any provider when the resource is declared in IaC, in any IaC tool, not only Terraform. `false`, meaning it exists in the account without an IaC record, is available for AWS today. We propose `managed_by_iac` (section 6). |
| `iac_drift_status` | PARTIAL | PLANNED | `UNMANAGED` is available for AWS today. `SYNCHRONIZED` and `DRIFT_DETECTED` are planned. |
| `last_iac_commit_hash` | PARTIAL | PLANNED | The commit of the IaC that declares the resource. Today for resources that run workloads; planned for the rest. |
| `reversibility_tier` | NOT_AVAILABLE | PLANNED | Computed from the per-class fields below, such as versioning, backups and deletion protection. |
| `estimated_blast_radius_score` | NOT_AVAILABLE | PLANNED | We propose CAGE computes it from the affected resources we return (Q7). |
| `compensating_rollback_action` | NOT_AVAILABLE | PLANNED | Per class, such as restoring from the latest snapshot where one exists. |
| `snapshot_id`, `snapshot_hash` | PARTIAL | PLANNED | Identifies the facts an answer read, not a whole-graph state (Q4). |
| `evaluated_at_utc` | NATIVE |  | |
| `query_latency_ms` | NOT_AVAILABLE | PLANNED | Measured once the adapter exists (Q5). Omitted from the fixtures. |

### Per-class fields (proposed)

Each class adds these fields to the shared fields above (section 5). We intend to support
`STORAGE_VOLUME` first. All are new; one is partly answered today.

**`DATABASE`**: a managed relational database instance.

| Field | Type | Fit | Roadmap | What the value means |
|---|---|---|---|---|
| `database_role` | `PRIMARY` / `READ_REPLICA` / `STANDBY` | NOT_AVAILABLE | PLANNED | The instance's role. |
| `database_replica_of` | URN | NOT_AVAILABLE | PLANNED | The primary, for a replica. |
| `database_replica_count` | int | NOT_AVAILABLE | PLANNED | Replicas of this instance. |
| `database_high_availability` | bool | NOT_AVAILABLE | PLANNED | A standby in another zone is configured. |
| `database_deletion_protection` | bool | NOT_AVAILABLE | PLANNED | The provider blocks deletion. |
| `database_backup_retention_days` | int | NOT_AVAILABLE | PLANNED | Automated backup retention. |
| `database_point_in_time_recovery` | bool | NOT_AVAILABLE | PLANNED | Point-in-time recovery is enabled. |
| `database_final_snapshot_on_delete` | bool | NOT_AVAILABLE | PLANNED | A final backup is taken on deletion, where the provider supports it. |
| `database_publicly_accessible` | bool | NOT_AVAILABLE | PLANNED | Reachable from the internet. |
| `database_connected_services` | int | NOT_AVAILABLE | PLANNED | Services that connect to it. |

**`STORAGE_VOLUME`**: a block storage volume.

| Field | Type | Fit | Roadmap | What the value means |
|---|---|---|---|---|
| `volume_attached_to` | URN | NOT_AVAILABLE | PLANNED | The instance it is attached to. |
| `volume_snapshot_schedule` | bool | NOT_AVAILABLE | PLANNED | Scheduled snapshots are configured. |
| `volume_latest_snapshot_age_hours` | int | NOT_AVAILABLE | PLANNED | Age of the newest known snapshot. |
| `volume_source_snapshot` | URN | NOT_AVAILABLE | PLANNED | The snapshot it was created from. |
| `volume_encrypted` | bool | NOT_AVAILABLE | PLANNED | Encrypted at rest with a managed key. |
| `volume_size_gb` | int | NOT_AVAILABLE | PLANNED | Provisioned size. |
| `volume_monthly_cost` | number | PARTIAL | PLANNED | Recent monthly cost. Available for AWS today; GCP and Azure planned. |

**`OBJECT_BUCKET`**: an object storage bucket.

| Field | Type | Fit | Roadmap | What the value means |
|---|---|---|---|---|
| `bucket_versioning_enabled` | bool | NOT_AVAILABLE | PLANNED | Deleted or overwritten objects are kept as versions. |
| `bucket_retention_locked` | bool | NOT_AVAILABLE | PLANNED | A locked retention policy or object lock applies. |
| `bucket_retention_days` | int | NOT_AVAILABLE | PLANNED | Minimum retention. |
| `bucket_force_destroy` | bool | NOT_AVAILABLE | PLANNED | IaC may delete it while it still holds objects. |
| `bucket_public_access` | bool | NOT_AVAILABLE | PLANNED | Any object is readable by anyone. |
| `bucket_replicated_to` | URN | NOT_AVAILABLE | PLANNED | Where its objects are replicated. |
| `bucket_lifecycle_expiration_days` | int | NOT_AVAILABLE | PLANNED | Objects expire after this many days. |
| `bucket_stored_gb` | number | NOT_AVAILABLE | PLANNED | Data stored. |
| `bucket_customer_managed_key` | bool | NOT_AVAILABLE | PLANNED | Encrypted with a customer-managed key. |

Under the v1.1 proposal, any of these fields may be `null` when it applies but is unknown (section 1).

### `BlastRadiusEstimate`

| Field | Fit | Roadmap | What the value means |
|---|---|---|---|
| `estimated_affected_services` | PARTIAL | PLANNED | The length of the affected list (Q7). Today a lower bound: the workloads that run on the resource. Their callers are planned. |
| `estimated_affected_users` | NOT_AVAILABLE | NOT_PLANNED | A count of end users cannot be known from the estate. We propose `exposure` on each affected resource instead (Q7). |
| `estimated_recovery_time_minutes` | NOT_AVAILABLE | PLANNED | An estimate per class, such as restore time from size, labeled as an estimate. |
| `requires_approval`, `approval_tier` | NOT_AVAILABLE | PLANNED | A versioned rule over environment tier and criticality, reported with its inputs. CAGE may instead set these itself from the affected list (Q7). |

### `TopologySnapshot`

| Field | Fit | Roadmap | What the value means |
|---|---|---|---|
| `nodes`, `edges` | NATIVE |  | Confirmed facts, each with provenance, `confirmed_at` and a content hash. Kinds and relations follow Operational Estate (section 6, item 10). |
| `captured_at_utc` | DERIVED |  | The newest `confirmed_at` among the facts returned. |
| `ttl_seconds` | NOT_AVAILABLE | PLANNED | As a per-field policy value (Q3). |
| `snapshot_id`, `snapshot_hash` | PARTIAL | PLANNED | As above. |

## 4. Decision points

| DP | Fit | Roadmap | Answered today | Planned |
|---|---|---|---|---|
| DP-01 | `NOT_AVAILABLE` | PLANNED | Resource type, IaC-managed | Environment tier, criticality, dependents, load-bearing, database fields |
| DP-02 | `NOT_AVAILABLE` | PLANNED | IaC-managed, IaC commit | Drift, failover, affected list for CAGE's score |
| DP-03 | `PARTIAL` | PLANNED | IaC commit | Drift |
| DP-04 | `PARTIAL` | PLANNED | `UNMANAGED` (AWS) | Owner team, GCP and Azure |
| DP-05 | `DERIVED` |  | `STALE` from confirmation time | None |
| DP-06 | `NATIVE` |  | `UNAVAILABLE` | None |
| DP-07 | `NOT_AVAILABLE` | PLANNED | Project known | Environment tier, reversibility |
| DP-08 | `PARTIAL` | PLANNED | Workloads on the shared cluster | Their callers, approval tier |
| DP-09 | `PARTIAL` | PLANNED | Resource not found | `NOT_FOUND` status |
| DP-10 | `PARTIAL` | PLANNED | IaC-managed | Environment tier, volume fields, reversibility |

## 5. Per-class predicates

The v1.0 predicate is the same for every resource. The facts that make an action risky are not:
deleting a database depends on whether it is a primary or a replica, deleting a volume on whether it
is attached and backed up, deleting a bucket on whether its objects are versioned or retention
locked. For the domain to be meaningful, the predicate needs fields for each resource class.

We propose starting with three classes, named provider-neutrally: `DATABASE`, `STORAGE_VOLUME` and
`OBJECT_BUCKET`. We intend to support `STORAGE_VOLUME` first. Their fields are listed in section 3.
They give policy concrete inputs where a single `reversibility_tier` or `has_failover_redundancy`
cannot. For example, deleting a bucket is
irreversible unless versioning is on or the bucket is replicated, and `bucket_force_destroy` makes
it worse. DP-01 and DP-10 turn on the `DATABASE` and `STORAGE_VOLUME` fields. No decision point
covers buckets yet; we propose one in section 6.

Each class extends the v1.0 predicate. A new field, `resource_class`, names the class. Every field
stays flat, with a class prefix (`database_role`, `volume_attached_to`) so names cannot collide
across classes. This keeps the current grammar, which reads `input.<param>` and has no dotted
paths. It also fits CAGE's STPA layout, which merges hazards from several source files into one
generated policy: hazards could be written in one STPA file per class, scoped with
`applies_when: resource_class == DATABASE`, and still compile to one policy.

**Shape.** The open question is how many document types `get_resource_predicate` returns:

| | A. One document per class | B. One sparse document |
|---|---|---|
| Fields | The v1.0 fields plus that class's fields | The v1.0 fields plus every class's fields, populated for the class acted on |
| Schema | One definition per class, selected by `resource_class`, each requiring its own fields | One definition; a class's fields are required through `if` / `then` on `resource_class` |
| Fields that do not apply | Absent | Present, carrying their type's default (section 1) |
| Seam | Each class subclasses `CloudOpsResourcePredicate`, so the method's return type is unchanged | One dataclass that grows with every class |
| Adding a class | A new definition; other classes are untouched | Changes the shared document |
| Grammar | Flat, no change | Flat, no change |

Both examples show a read replica, with the shared v1.0 fields left out for brevity.

Option A carries only the database fields:

```json
{
  "resource_class": "DATABASE",
  "database_role": "READ_REPLICA",
  "database_replica_of": "//sqladmin.googleapis.com/projects/p/instances/orders",
  "database_replica_count": 0,
  "database_high_availability": false,
  "database_deletion_protection": true,
  "database_backup_retention_days": 7,
  "database_point_in_time_recovery": false,
  "database_final_snapshot_on_delete": false,
  "database_publicly_accessible": false,
  "database_connected_services": 3
}
```

Option B carries every class's fields. Those that do not apply to a database carry their type's
default (section 1):

```json
{
  "resource_class": "DATABASE",
  "database_role": "READ_REPLICA",
  "database_replica_of": "//sqladmin.googleapis.com/projects/p/instances/orders",
  "database_replica_count": 0,
  "database_high_availability": false,
  "database_deletion_protection": true,
  "database_backup_retention_days": 7,
  "database_point_in_time_recovery": false,
  "database_final_snapshot_on_delete": false,
  "database_publicly_accessible": false,
  "database_connected_services": 3,
  "volume_attached_to": "",
  "volume_snapshot_schedule": false,
  "volume_latest_snapshot_age_hours": 0,
  "volume_source_snapshot": "",
  "volume_encrypted": false,
  "volume_size_gb": 0,
  "volume_monthly_cost": 0,
  "bucket_versioning_enabled": false,
  "bucket_retention_locked": false,
  "bucket_retention_days": 0,
  "bucket_force_destroy": false,
  "bucket_public_access": false,
  "bucket_replicated_to": "",
  "bucket_lifecycle_expiration_days": 0,
  "bucket_stored_gb": 0,
  "bucket_customer_managed_key": false
}
```

In either shape, a field the adapter leaves out would make a Rego rule over it not fire, which
fails open. So every field a class defines is required, and a field that applies but is unknown is
`null` (section 1). We would like your view on which shape you prefer.

## 6. Proposals and questions for Lars

None of these change v1.0. Each one is listed in the `gaps` of the fixtures it affects.

1. **A `roadmap` field alongside `fit`** (`PLANNED` or `NOT_PLANNED`), so a response can say whether
   a gap will close. It is required when `fit` is `PARTIAL` or `NOT_AVAILABLE`, and absent when
   `fit` is `NATIVE` or `DERIVED`, which mean complete. The same applies to each field, through
   structured `gaps` entries (`field`, `fit`, `roadmap`, `reason`) in place of the text prefix we
   use in v1.0.

   *Question:* Will you add a `roadmap` field alongside `fit`, and structured `gaps` entries, in
   v1.1?

2. **`NOT_FOUND` and `NOT_COVERED` statuses**, distinct from `INVALID` (Q2).

   *Question:* Do you agree with separate `NOT_FOUND` and `NOT_COVERED` statuses?

3. **Per-class predicates**: a `resource_class` field and the per-class fields in section 3,
   carried in one of the two shapes in section 5.

   *Question:* Should `get_resource_predicate` return one document per class, or one sparse
   document?

4. **Nullable predicate fields**, with `null` meaning a field that applies but is unknown, and type
   defaults only for fields that do not apply (section 1). Every field stays required, so a rule
   over it cannot silently not fire, and policy fails closed on `null`. This replaces the
   placeholders our v1.0 fixtures use.

   *Question:* Will you make predicate fields nullable in v1.1, with `null` meaning unknown and type
   defaults only for fields that do not apply?

5. **Affected resources from `query_blast_radius`**, each with its `exposure` (`PUBLIC` or
   `INTERNAL`), with CAGE computing the score (Q7). This replaces `estimated_affected_users`.

   *Question:* Should `query_blast_radius` return the affected resources, with CAGE computing the
   score?

6. **Per-verb parameters on `query_blast_radius`.** Scaling from 3 to 0 is a different risk from 3
   to 1, though the verb is the same. For example, `scale_down` with a target replica count, or
   `set_iam_policy` with the bindings added and removed.

   *Question:* A typed argument per verb, or a free parameters object?

7. **TTL set per field by policy**, rather than one window per snapshot (Q3).

   *Question:* Should TTL be set per field by policy?

8. **A bucket decision point (DP-11)**: an agent deletes an object storage bucket, or makes one
   public. ALLOW when `bucket_versioning_enabled` is true and the bucket is outside production;
   DEFER otherwise.

   *Question:* Would you add a bucket decision point?

9. **`managed_by_iac` in place of `managed_by_terraform`**: an enum naming the IaC tool
   (`TERRAFORM`, `OPENTOFU`, `PULUMI`, `CLOUDFORMATION`, `CDK`, `KUBERNETES`, and others), or `NONE`
   for a resource that exists with no IaC record. `NONE` is a value, not `null`, because `null` means
   unknown under item 4. `NONE` corresponds to `iac_drift_status: UNMANAGED`. Until then, our
   fixtures report `managed_by_terraform: true` for a resource declared in any IaC tool.

   *Question:* Will you replace `managed_by_terraform` with a `managed_by_iac` enum?

10. **Snapshot nodes and edges follow [Operational Estate](https://github.com/TheRealOpsCanvas/operational-estate)**, an open,
    vendor-neutral ontology of a software estate. In v1.0, `nodes` and `edges` are untyped objects
    and `scope` has no grammar. We propose that a node's `kind` is an ontology class (such as
    Deployed Resource, Cloud Account or Service Instance), its key follows the ontology's key rules,
    and an edge's `relation` is an ontology relation (such as `within`, `runsOn` or `connectsTo`).
    Our fixtures use the ontology's class and relation names; they identify nodes by provider-native
    URN rather than by the ontology's keys. The ontology is a draft, so the seam would pin a version.
    OpsCanvas started it and maintains a reference implementation; the specification names no
    product.

    *Questions:* Should snapshot nodes and edges follow Operational Estate, or do you want a
    CAGE-defined shape? And with the affected-resources list in item 5, is the snapshot still needed
    for blast radius, or only for context after a deferral (DP-03) and for evidence?

11. **Per-field provenance** (`field_provenance`), so policy can tell an observed value from a
    derived value or a strict placeholder. Already queued for v1.1; the provenance values in Q3
    map onto it.
