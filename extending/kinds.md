---
title: Defining your own kind
permalink: /extending/kinds/
---

# Defining your own kind

Kevaso's own kinds (`Silo`, `Service`, `Secret`…) are fixed. If something you build on Kevaso
needs a new one (a `Database`, a `Task`), you **define** it at runtime with a `KindDefinition`. Kevaso
then serves it like a built-in kind: create, read, update, delete, watch, roles, the audit log,
`kvo get`, `kvo apply`.

A defined kind is only data until something acts on it. That something is a **controller**: a
program you write that watches the kind and makes it true. [Policies]({{ '/extending/policies/' | relative_url }})
add rules to what may be written.

---

## Define a kind

Save this as `database-kind.yaml`:

```yaml
apiVersion: kevaso/v1
kind: KindDefinition
metadata: { name: databases.acme.dev }
spec:
  group: acme.dev
  version: v1
  names: { kind: Database, plural: databases, singular: database, shortNames: [db] }
  scope: Namespaced
  subresources: { status: {} }
  schema:
    type: object
    properties:
      spec:
        type: object
        properties:
          engine:   { type: string, enum: [postgres, mysql] }
          size:     { type: string, pattern: '^[0-9]+(Gi|Ti)$' }
          replicas: { type: integer, minimum: 1, maximum: 5, default: 1 }
        required: [engine]
        x-kevaso-validations:
          - { rule: "self.engine != 'mysql' || has(self.size)", message: "mysql needs a size" }
      status: { type: object, x-kevaso-preserve-unknown: true }
  printerColumns:
    - { name: Engine, jsonPath: .spec.engine }
    - { name: Ready, jsonPath: '.status.conditions[?(@.type=="Ready")].status' }
```

```bash
kvo apply -f database-kind.yaml
```

Defining a kind needs a role that can write `kinddefinitions` (cluster admins have it). Now people
use it:

```yaml
apiVersion: acme.dev/v1
kind: Database
metadata: { name: orders, namespace: payments }
spec: { engine: postgres, size: 10Gi }
```

```bash
kvo apply -f orders.yaml
```

```bash
kvo get db -n payments
```

```text
NAME    ENGINE    READY  AGE
orders  postgres  -      4s
```

```bash
kvo get kinds
```

lists every kind the cluster serves, built-in and defined, with its group and version.

---

## The names

| | Rule |
|---|---|
| **Group** | A domain you own, in lower case, with at least one dot: `acme.dev`, `tasks.example.com`. `kevaso`, `kevaso.com` and anything under `kevaso.com` are Kevaso's and refused. |
| **Version** | `v1`, `v2`, … One version per kind, and no conversion between versions. |
| **`names.kind`** | Letters and digits, starting with a capital: `Database` |
| **`names.plural`** | Lower case: `databases` |
| **The definition's name** | **`PLURAL.GROUP`**, exactly: `databases.acme.dev`. This string is the kind's *resource*: it's what roles and policies call it. |
| **`apiVersion` of its objects** | `GROUP/VERSION`: `acme.dev/v1` |

A defined kind **can't shadow a built-in one.** Its kind, plural, singular and short names can't be
the name, plural or short name of a built-in kind (capitals don't count), so `kvo get silos` always
means Kevaso's silos. Two groups may each define a `Database`; where two kinds share a plural or
short name, use the long name: `kvo get databases.acme.dev`.

Once defined, **these can't change**: group, version, kind, plural and scope (they're in every
stored object and every role). You can change the singular and short names, the schema, the status
subresource and the printer columns.

`scope` is `Namespaced` or `Cluster`.

---

## The schema

The schema describes the object's top-level fields other than `apiVersion`, `kind` and
`metadata`. The root is `type: object`, and everything in it must be listed. A field the schema
doesn't list is refused, unless that node says `x-kevaso-preserve-unknown: true`.

It's a part of OpenAPI v3:

| You can use | |
|---|---|
| `type` | `object`, `array`, `string`, `integer`, `number`, `boolean` |
| Objects | `properties`, `required`, `additionalProperties` (a map whose values share one schema) |
| Arrays | `items`, `minItems`, `maxItems` |
| Strings | `enum`, `pattern`, `minLength`, `maxLength` |
| Numbers | `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum` |
| Any | `default`, `description` |
| `x-kevaso-preserve-unknown: true` | Keep fields the schema doesn't list |
| `x-kevaso-validations` | Rules in [CEL](https://cel.dev) on that node, each `rule` with a `message`. `self` is the node, and `oldSelf` its old value ([below](#rules-that-compare-with-the-old-value)) |

Anything else in a schema is refused when you write the definition, and so is a CEL rule that
doesn't compile.

**What happens on each write**, in order: the object is decoded (its metadata strictly), Kevaso adds
its own metadata (uid, times, generation), schema defaults fill in, [mutation policies]({{ '/extending/policies/' | relative_url }})
run, the schema and CEL rules are checked, then validation policies, and then it's stored.
`generation` goes up when anything other than `metadata` and `status` changes. (For Kevaso's own
kinds the built-in checks come between: [the order of a whole write]({{ '/extending/policies/#the-order-of-a-write' | relative_url }}).)

**Changing a definition never makes stored objects unreadable.** Reads return what's stored and
aren't checked against the schema. A new schema applies from the next write of each object.

---

## Status

With `subresources: { status: {} }`:

- People write the main object, and its `status` is ignored: it stays as stored.
- Controllers write `…/status`, which changes **only** `status`.

Without it, `status` is an ordinary field and there's no `/status` to write.

### Rules that compare with the old value

A rule can use `oldSelf`, the node's value in the stored object. It is checked on **updates only**,
and skipped on a create, and when the old object didn't have the node (so a field can be set for
the first time).

```yaml
template:
  type: string
  x-kevaso-validations:
    - { rule: "self == oldSelf", message: "a session's template can't change: start a new session" }
```

`self >= oldSelf` makes a number that only grows. A list's items have no old value to compare with,
so a rule with `oldSelf` inside `items` is refused when you write the definition: put it on the list.

---

## Scale

With `subresources: { scale: { specReplicasPath: .spec.replicas, statusReplicasPath: .status.replicas } }`,
the kind scales like a set:

```bash
kvo scale session fix-login-bug --replicas 3 -n dev
```

`--replicas` is the same flag as `--instances`. Both paths are dotted paths to integers in your
schema (the second is optional), checked when the definition is written. A scale is an ordinary
update of the object: defaults, policies, the schema's bounds and rules all apply. RBAC calls it
`PLURAL.GROUP/scale` (`get`, `update`).

---

## Roles

The built-in roles don't reach defined kinds. The exception is `cluster-admin`, which has
everything. Everyone else gets a role you write, naming the kind's resource, **`PLURAL.GROUP`**, and
`PLURAL.GROUP/status` for the status:

```yaml
apiVersion: kevaso/v1
kind: Role
metadata: { name: db-editor, namespace: payments }
rules:
  - verbs: [get, list, watch, create, update, delete]
    resources: [databases.acme.dev]
```

```yaml
apiVersion: kevaso/v1
kind: ClusterRole
metadata: { name: db-operator }
rules:
  - verbs: [get, list, watch]
    resources: [databases.acme.dev]
  - verbs: [update]
    resources: [databases.acme.dev/status]
```

The first is for people who work with databases in one namespace (bind it with a `RoleBinding`);
the second is for a controller that reads databases everywhere and reports their status (bind it
with a `ClusterRoleBinding` to the controller's service account). See [People and access]({{ '/access/' | relative_url }}).

---

## A controller

A controller is a program that **watches** your kind and **reconciles** each object: it looks at what
was asked for and makes the world match, then writes the result to the object's status. Write one
in Go with `sdk/controller`, which needs only the public `kevaso.com/core/api` and
`kevaso.com/core/sdk` modules. It:

- watches one kind through an **informer**, a cache that stays up to date;
- puts the **key** (namespace and name) of each changed object in a queue, once, however often it
  changed;
- calls your function for each key, with the *current* state, never a stale event;
- retries a failure with a growing delay (100 ms doubling to 1 minute, by default).

```go
inf := informer.New(c, "databases.acme.dev")
ctl := &controller.Controller{Name: "databases", Informer: inf, Reconcile: reconcile}
ctl.Run(ctx)
```

```go
func reconcile(ctx context.Context, k informer.Key) (controller.Result, error) {
	obj, ok := inf.Get(k.Namespace, k.Name)
	if !ok {
		return controller.Result{}, nil // it's gone: clean up what you made for it
	}
	// … read obj.(*v1.Unstructured), do the work, then report:
	// err := c.UpdateStatus(ctx, &db)
	return controller.Result{}, nil // or Result{RequeueAfter: time.Minute} to look again
}
```

`c` is a `*client.Client` signed in as the controller's own identity (a service account with the
`db-operator` role above). Your reconcile function must be safe to run twice: it's called again
after every change, and after every retry.

A complete controller for the `Database` kind above, using its own Go type, is in
`test/integration/dbcontroller_test.go` in the core repository.

---

## Deleting, and what isn't here yet

**A definition with objects left can't be deleted.** The request is refused with how many remain.
Delete the objects first, then the definition.

**Not done yet:**

- More than one version of a kind at once.
- A scale subresource with a label selector, and `oldSelf` rules on a list's items.

A plugin can ship definitions with its controller, and owns them:
[Plugins]({{ '/extending/plugins/#plugin-operators' | relative_url }}).
