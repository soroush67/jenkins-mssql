# ms-stack

Microsoft SQL Server 2022, deployable to any destination via `Jenkins +
Ansible + Docker Compose` - same shape and same discipline as this
user's `jenkins-postgresql` project (Ansible-templated compose, hardened
containers, `DEPLOY_PATH` + `TARGET_HOST` Jenkins parameters, volumes
under `/data/<name>`, never `become`/sudo). Started from a hand-written
`docker-compose.yml` that had never actually been run hardened - see
"Design notes" below for every real bug/requirement found while getting
it there.

## Quickstart

```
ansible-playbook playbooks/deploy.yml -e mssql_sa_password='...'   # idempotent - safe to rerun
ansible-playbook playbooks/status.yml -e mssql_sa_password='...'   # read-only health/connectivity check
```

`mssql_sa_password` is required, never committed, and must meet SQL
Server's own password policy (at least 8 characters, 3 of {uppercase,
lowercase, digit, symbol}) - `roles/preflight` checks both and fails
cleanly before ever touching Docker.

`-e target_host=<ip-or-hostname>` (Jenkins: `TARGET_HOST`) deploys to
any destination without needing to pre-add it to `inventory/hosts.ini`
first. `-e mssql_deploy_path=/opt/servers/mssql-prod-1` (Jenkins:
`DEPLOY_PATH`) pins exactly where `docker-compose.yml` gets created -
the actual data **volume** lives separately, under `/data/<the last
folder of that path>` (`/data/mssql-prod-1` for the example above).
Both default to sensible local-testing values when left unset.

## Design notes

Real issues found and requirements confirmed while wrapping the original
hand-written stack in Ansible - same "verify by actually running it"
discipline as this user's other infra projects; nothing here was assumed
from Microsoft's docs alone.

**The original file used an external named Docker volume (`ms-data`),
not a bind mount.** Changed to a bind mount under `/data/<the last
folder of DEPLOY_PATH>`, matching `jenkins-postgresql`'s established
convention for this user's infrastructure (their own explicit
requirement there: all volumes live under `/data/`, namespaced by
deployment). Flag if the external-named-volume approach was actually
intentional for this stack specifically - it's a real behavior change
from the original file, not just a wrapper around it.

**`cap_drop: ALL` alone made `sqlservr` fail to even launch** -
`/opt/mssql/bin/launch_sqlservr.sh: line 20: /opt/mssql/bin/sqlservr:
Operation not permitted`, confirmed directly, with no other symptom (the
container's data directory was already correctly owned; this wasn't a
permissions-on-disk problem). Root cause, also confirmed directly via
`getcap`: the `sqlservr` binary has `cap_net_bind_service=ep` set as a
file capability - the kernel only honors a binary's own file
capabilities at exec time if that capability is also present in the
container's capability *bounding set*, which `cap_drop: ALL` empties
out entirely. Fixed with `cap_add: [NET_BIND_SERVICE]` - confirmed as
the complete, minimal fix by testing broader candidate sets first
(`CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID`, `SYS_PTRACE`,
`SYS_RESOURCE`, `IPC_LOCK` - none of these fixed it on their own) before
finding the actual one via `getcap` rather than guessing further.

**The mssql image already bakes in `USER mssql` (uid 10001) in its own
Dockerfile** - unlike the postgres image `jenkins-postgresql` uses
(which defaults to root unless told otherwise, letting its entrypoint's
own privilege-dropping dance run). Confirmed directly: a throwaway
container used to `chown` a host directory for this image needs
`--user root` *explicitly* - simply omitting `--user` here still runs as
the image's own non-root default, not root, and the chown fails
("Operation not permitted"). Every privileged-container step in
`roles/mssql` passes `--user root` explicitly for this reason.

**`mssql-tools18`'s `sqlcmd` requires `-C` (trust server certificate)** -
confirmed directly: v18 encrypts connections by default and refuses an
unverified self-signed certificate without it. Used in both the
container healthcheck and every SA-connectivity check this role/the
status playbook run.

**SQL Server's own password policy needed checking up front, not left to
fail inside the container.** A password that doesn't meet the policy
(8+ characters, 3 of {uppercase, lowercase, digit, symbol}) makes the
container fail in a much less obvious way than a clean, up-front Ansible
assertion - `roles/preflight` checks this directly (confirmed: a
deliberately weak test password was correctly rejected before Docker was
ever touched).

**The rest of the permission/idempotency/`DEPLOY_PATH`/`target_host`
design is identical to `jenkins-postgresql`**, including two real bugs
already found and fixed there and inherited correctly here rather than
re-discovered: `realpath` (not a bare `basename`) is required to derive
`/data/<name>` from `mssql_deploy_path`'s own default value (which ends
in a literal `/..`); and `ignore_errors: true` (not `failed_when: false`)
is required for the "try directly, fall back to a privileged container"
pattern to actually detect a real permission failure - `failed_when:
false` silently rewrites the registered result's own `failed` key,
breaking the downstream `is failed` check that gates the fallback. See
that project's own README for the full story on both.

**Verified end-to-end, not just "should work":** a real `sqlcmd -Q
"SELECT 1"` against the SA login succeeded after every deploy, not just
a "container running" check; two consecutive `deploy.yml` runs left
`docker inspect --format '{{.State.StartedAt}}'` completely unchanged;
tested the permission-fallback path for real by resetting `/data` to
root-owned and confirming the deploy still completed without stopping;
tested a custom `DEPLOY_PATH` and confirmed `docker-compose.yml` and the
data volume landed in exactly the two separate places expected, nothing
else created anywhere.
