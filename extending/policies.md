---
title: Policies
permalink: /extending/policies/
---

# Policies

A policy is a rule about what may be written to the cluster. Two kinds, both written in
[CEL](https://cel.dev), an expression language, and both run inside the control plane: no
webhook, no outside service to be down.

| Kind | Does |
|---|---|
| `MutationPolicy` | **Sets** fields on an object before it's checked: defaults, labels, who made it |
| `ValidationPolicy` | **Refuses** a write whose expressions aren't true |

Both are cluster-scoped, so applying one needs a role on `validationpolicies` or `mutationpolicies`
(cluster admins have it).

They cover Kevaso's own kinds (`Silo`, `Service`…) and [kinds you define]({{ '/extending/kinds/' | relative_url }}).

---

## A validation policy

```yaml
apiVersion: kevaso/v1
kind: ValidationPolicy
metadata: { name: prod-says-what-happens-on-host-loss }
spec:
  match:
    kinds: [Silo]
    namespaces: [prod]
    operations: [CREATE, UPDATE]
  validations:
    - expression: "has(object.spec.availability) && has(object.spec.availability.onHostLoss)"
      message: "silos in prod must set spec.availability.onHostLoss"
```

```bash
kvo apply -f policy.yaml
```

From now on, a silo in `prod` without it is refused, with the `message`.

### `match`

| Field | Meaning |
|---|---|
| `kinds` | Required. A built-in kind by name or plural (`Silo`, `silos`), a defined kind by its resource (`databases.acme.dev`), or `*` for all. |
| `namespaces` | Empty: every namespace, and cluster-scoped objects too. Otherwise only objects in these namespaces. A cluster-scoped object matches only a policy with no namespaces. |
| `operations` | `CREATE`, `UPDATE`. Empty: both. |

### What an expression can see

| Variable | Is |
|---|---|
| `object` | The object being written, as it would be stored |
| `oldObject` | The object as it is now, or `null` on a create |
| `request` | Who and what: `user`, `groups`, `verb`, `operation`, `namespace`, `name`, `kind`, `resource` |

Use `has(object.spec.x)` before reading a field that might not be there.

### How they run

**Every** matching validation policy runs, in **name order**, and a refusal lists **every**
failure, each starting with its policy's name. So you see all the problems at once.

---

## A mutation policy

```yaml
apiVersion: kevaso/v1
kind: MutationPolicy
metadata: { name: record-who-made-it }
spec:
  match:
    kinds: [Silo]
    operations: [CREATE]
  mutations:
    - field: metadata.annotations.created-by
      value: request.user
    - field: metadata.labels.runtime
      value: object.spec.runtime
```

| Field | Meaning |
|---|---|
| `field` | A dotted path: `spec.size`, `spec.limits.cpu`, `metadata.labels.KEY`, `metadata.annotations.KEY`. Never `apiVersion`, `kind`, the rest of `metadata`, or `status`. |
| `value` | A CEL expression for the new value. `null` **removes** the field. |
| `when` | Optional CEL. The mutation applies only when it's true. |

A string value is a CEL string: write `"'docker'"`, with the inner quotes, for the text `docker`.
Labels and annotations must come out as strings.

Matching policies run in **name order**, each one's mutations in **list order**, each seeing the
object as the ones before left it. After the last one, the object must still be a valid one for its
kind: a mutation that makes a silo undecodable fails the write.

**Order of a whole write:** mutation policies, then the kind's own checks (a defined kind's schema
and rules), then validation policies, then it's stored. So validation policies judge the object
*after* mutation.

---

## Fail closed

A policy that **can't be evaluated refuses every write it matches**, and says which policy and
why. That covers a runtime error (reading a field that isn't there without `has`), an answer that
isn't `true` or `false`, a mutation that sets something it can't, and a `when` that isn't a boolean.
A policy whose CEL doesn't compile is refused when you apply it, so it never gets that far.

Fail-closed is deliberate: a broken rule can't quietly let things through. The way back from a bad
policy is to fix or delete it, and that always works (see the next section).

---

## What they never apply to

| Not covered | Why |
|---|---|
| `ValidationPolicy` and `MutationPolicy` themselves | A bad policy can always be fixed or deleted |
| `Role`, `ClusterRole`, `RoleBinding`, `ClusterRoleBinding` | A policy can never lock people out |
| `/status` writes | Controllers reporting aren't blocked |
| Deletes | Only creates and updates are matched |

---

## Not done yet

- **Parameters.** A policy can't take a parameter object: the values are in the expression.
- **Webhooks.** There are none by design.
