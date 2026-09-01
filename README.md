# NCSC Cyber Drill — demo sandbox definition

A starting point for authoring your own scenario, based on
[cyberrangecz/library-demo-training](https://github.com/cyberrangecz/library-demo-training)
with our AWS corrections already applied.

**This will not build until you replace the AMI placeholders.** That is deliberate — a
placeholder fails loudly, whereas a wrong-but-plausible image ID fails confusingly.

---

## What is in here

| File | What it does | Required? |
|---|---|---|
| `topology.yml` | The network: hosts, routers, networks, addresses | Yes |
| `provisioning/playbook.yml` | What gets configured on each machine | Yes |
| `provisioning/requirements.yml` | External Ansible roles to download | Yes (may be empty) |
| `training.json` | The exercise: levels, questions, answers, scoring | Yes, to run a training |
| `variables.yml` | Per-participant randomised values | Only for adaptive trainings |

---

## Before your first build

1. **Replace both AMI placeholders in `topology.yml`.** `REPLACE-WITH-UBUNTU-AMI-ID` and
   `REPLACE-WITH-DEBIAN-AMI-ID` must become AMI IDs that exist in the account. Confirm each
   one appears in the platform's image list first — a missing AMI fails the build outright.

2. **Change the flag.** It appears in two places and they must match:
   `provisioning/playbook.yml` (the file placed on the server) and `training.json` (the
   expected `answer`). If they differ, the exercise is unsolvable.

3. **Push to a Git repository the platform can reach.** GitLab is not reachable from the
   orchestration account, so this goes to GitHub. One branch per definition.

4. **Use an authenticated token.** Anonymous GitHub allows 60 requests an hour and twelve
   worker pods will exhaust that immediately — a 10-sandbox pool took over two hours before
   this was fixed. See Issue 10 in the deployment runbook.

---

## The push-to-running loop

**Every change follows this order. Skipping step 3 is the most common wasted hour.**

```
1. Commit and push
2. Verify the remote actually has your commit
3. CLEAR THE REDIS TOPOLOGY CACHE      ← deployment runbook §8.3
4. Allocate the pool
```

The platform caches topologies against a revision string. For GitHub that string is the
*branch name*, not the commit ID — so it never changes when you commit, and the platform
keeps serving your old topology. Your edit is fine; the cache is stale. See Issue 7 in the
deployment runbook.

---

## What this topology deliberately tests

The vendor demo uses **one** router for both networks. This uses **two**, one per network.

That is not decoration. Our early testing suggested chained multi-router layouts might not
be possible on AWS, which matters because the security team's reference architecture needs
exactly that. The platform documentation says otherwise — *a network connects to one router,
but a topology may have many* — so this build settles the question at the cheapest possible
scale.

**What to check once it builds:**

- Both routers appear as instances, and neither fails with `AttachmentLimitExceeded`.
- The client (`192.168.30.5`) can reach the server (`192.168.20.5`) — traffic crossing both
  routers is the whole point.
- The topology view renders both routers.

If all three hold, chained topologies are viable and the constraint recorded in
`CyberDrill-Platform-Business-Case.md` section 6 should be revised. If routing between
segments fails, the routers' own routing tables are the next thing to look at — that is
provisioning work, not a platform limitation.

---

## Growing this

**Add a host:** an entry under `hosts`, a `net_mappings` entry giving it an address, and a
play in `playbook.yml`.

**Add a network:** an entry under `networks` with a non-overlapping CIDR, and a
`router_mappings` entry attaching it to exactly one router. **Watch the router's size** —
each attached network adds an interface, and instance types cap how many a machine can have.
That is why the routers here are `t3.xlarge` and not `t3.small`.

**Naming rules:** lowercase letter first, then `a-z A-Z 0-9 -`. Host, network and router
names must be unique within the topology, and network ranges must not overlap.

**Addresses:** the first addresses in each subnet are reserved for the gateway and DHCP.
Hosts here use `.5` and routers `.254` for that reason.

---

## Extending `training.json`

This file has three levels — info, access, and one scored task. It is intentionally small.

Level types available: `INFO_LEVEL`, `ACCESS_LEVEL`, `TRAINING_LEVEL`, `ASSESSMENT_LEVEL`.

**Before adding hints, assessments or MITRE ATT&CK mappings, copy the exact structure from the
vendor's own `training.json`** rather than inventing it — that file exercises every level type
and is the authoritative example:

```
https://github.com/cyberrangecz/library-demo-training/blob/master/training.json
```

The `hints` array here is left empty for that reason: its inner field names have not been
verified against a live import, and a malformed level fails the upload.

---

## Reminders

- `training.json` must be **uploaded before** the training definition can be created. The
  definition cannot exist without it.
- Pool size can be increased but **never reduced** — to shrink one, delete and recreate.
- `ChangeMe123` in the playbook is a throwaway lab credential, not a secret. Change it before
  any real drill and never reuse the pattern outside a disposable sandbox.

---

## Reference

- [Sandbox Definition docs](https://docs.platform.cyberrange.cz/user-guide-advanced/sandboxes/sandbox-definition/)
- [Topology Definition docs](https://docs.platform.cyberrange.cz/user-guide-advanced/sandboxes/topology-definition/)
- [Vendor reference labs](https://github.com/cyberrangecz) — the `library-*` repositories
- `CyberRange-AWS-Deployment` — our deployment runbook (issues, fixes, commands)
