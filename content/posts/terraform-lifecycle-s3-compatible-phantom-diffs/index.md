---
title: "Terraform Lifecycle Rules Against S3-Compatible Storage: Hunting Two Phantom Diffs"
date: 2026-06-12T10:00:00+01:00
slug: "terraform-lifecycle-s3-compatible-phantom-diffs"
tags: ["terraform", "opentofu", "hetzner", "s3", "object-storage"]
series: []
description: "Hetzner Object Storage accepts your S3 lifecycle configuration just fine — but the AWS Terraform provider fights you twice: an apply timeout from an attribute Ceph never returns, and a perpetual diff from rule ordering. A full post-mortem."
draft: true
---

> TL;DR: Hetzner Object Storage (Ceph/RGW under the hood) accepts your S3 lifecycle configuration just fine — but the AWS Terraform provider will fight you twice afterwards: once with an apply timeout caused by an attribute Hetzner never returns, and once with a perpetual diff caused by rule ordering. Both are fixable in the module. Here's the full post-mortem.

## Context

I recently split my infrastructure monorepo into a private deployment repo and a public, versioned Terraform module collection. As part of the rewire, my backups bucket on **Hetzner Object Storage** moved from an inline `aws_s3_bucket` resource to a generic `object-storage` module pinned to a release tag, gaining two things it never had: an explicit private ACL and lifecycle retention rules:

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "backups" {
  bucket = aws_s3_bucket.backups.id

  rule {
    id     = "expire-daily"
    status = "Enabled"
    filter { prefix = "daily/" }
    expiration { days = 7 }
  }

  rule {
    id     = "expire-weekly"
    status = "Enabled"
    filter { prefix = "weekly/" }
    expiration { days = 28 }
  }

  rule {
    id     = "expire-monthly"
    status = "Enabled"
    filter { prefix = "monthly/" }
    expiration { days = 90 }
  }
}
```

Nothing exotic. Prefix-scoped expiration, the textbook example from the provider docs. The plan was clean: one `moved`, two resources to add, zero to destroy.

Then I applied it.

## Phantom #1: the apply that times out on a resource that exists

The apply created the ACL, executed the state move… and then hung on the lifecycle configuration until the provider gave up:

```text
Error: waiting for S3 Bucket Lifecycle Configuration ... timeout while waiting for state
```

Here's the fun part: **the lifecycle configuration was created successfully.** Querying the bucket directly showed all three rules, enabled, with the right prefixes and expiration days. The bucket was in exactly the desired state. Terraform just didn't know it.

The cause lives in how the AWS provider confirms eventual consistency. After a `PutBucketLifecycleConfiguration`, it polls `GetBucketLifecycleConfiguration` until the response matches what it wrote — including `transition_default_minimum_object_size`, an attribute that **only AWS S3 returns** (it defaults to `all_storage_classes_128K` on real S3). Ceph's RADOS Gateway, which is what Hetzner runs, doesn't know this attribute exists and never includes it in the response. The provider waits for a convergence that can never happen, then times out.

The damage: a real resource in the cloud with no entry in your state. The next plan shows a spurious `1 to add`, and re-applying would just re-trigger the same timeout.

### The fix, in two parts

First, adopt the orphan instead of recreating it:

```bash
tofu import module.backups.aws_s3_bucket_lifecycle_configuration.backups <bucket-name>
```

Import only reads from the API and writes to local state — no mutation, no timeout.

Second, stop the provider from ever caring about the phantom attribute. In the module:

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "this" {
  # ...

  lifecycle {
    ignore_changes = [transition_default_minimum_object_size]
  }
}
```

Since the module defines no storage-class transitions, the attribute is inert anyway. That shipped as `v1.0.1`.

## Phantom #2: the diff that swaps two rules forever

With the import done and `v1.0.1` consumed, I expected `No changes`. Instead:

```text
~ resource "aws_s3_bucket_lifecycle_configuration" "backups" {
    # rule positions 2 and 3 swapped: weekly <-> monthly
  }
Plan: 0 to add, 1 to change, 0 to destroy.
```

Same three rules. Same prefixes, same expiration days, same everything — just reordered.

This one is a collision between two reasonable behaviors:

1. **Ceph/RGW returns lifecycle rules sorted alphabetically by rule ID.** So the API response is always `expire-daily`, `expire-monthly`, `expire-weekly`.
2. **The AWS provider models `rule` as an ordered list**, not a set. Declaration order is configuration; response order is state. If they differ, that's a diff.

My module declared the rules in the human-intuitive order — daily, weekly, monthly — so positions 2 and 3 could never match. And applying the "fix" would be pointless: the provider would write the rules, Ceph would return them alphabetically on the next read, and the diff would resurrect. A perpetual phantom.

Worth noting: this one is invisible at plan-gate time. A plan from a branch shows the rules as a clean `add`; the ordering mismatch only materializes **after** the resource exists in state and gets refreshed. If your change process relies on reviewing plans before apply — mine does — this class of diff will always ambush you post-apply.

### The fix

Declare the rules in the order the backend will return them — alphabetically by ID:

```hcl
# daily, monthly, weekly — matches Ceph's sort order on read
rule { id = "expire-daily"   ... }
rule { id = "expire-monthly" ... }
rule { id = "expire-weekly"  ... }
```

It feels wrong to let a storage backend dictate your declaration order, but it's the only idempotent option short of forking the provider. That shipped as `v1.0.2`, and the stack finally reached the only acceptable end state:

```text
No changes. Your infrastructure matches the configuration.
```

## Takeaways

**S3-compatible is a wire protocol claim, not a behavioral one.** Hetzner, Cloudflare R2, MinIO, Backblaze — they all speak the API, but attribute defaults, response ordering, and consistency semantics are where the provider's AWS-shaped assumptions leak. Lifecycle configuration seems to be a particularly rich seam.

**Pin modules to tags and gate every bump with a plan.** Both phantoms were caught and fixed across two patch releases without production ever applying a wrong change. The pin-bump-plan-apply loop is boring, and boring is the point.

**`import` is the right tool for timeout orphans.** When an apply times out but the resource exists, the instinct to re-apply is wrong — adopt the resource into state and reconcile from there.

**Know your post-apply blind spot.** Some diffs (like response-ordering mismatches) cannot appear in a pre-apply plan. Budget for one verification plan _after_ every apply, not just before.

**Document the quirks in the module itself.** Both workarounds live in the module's CHANGELOG and the stack's README now. The next person hitting a three-minute hang on a Hetzner bucket — possibly me, in a year — gets the answer in `git blame` instead of an afternoon of debugging.

---

_The module in question is open source: generic Hetzner server, object storage, and Cloudflare configuration modules, hardened defaults, Molecule and Terratest coverage. Both phantom fixes are in `v1.0.2`._
