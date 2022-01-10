---
title: "Concourse CI on Kubernetes (GKE), Part 6: Concourse & Vault: Backup & Restore"
date: 2022-01-08T04:55:18-08:00
draft: false
images:
- https://upload.wikimedia.org/wikipedia/commons/2/29/Postgresql_elephant.svg
---

{{< figure src="https://upload.wikimedia.org/wikipedia/commons/2/29/Postgresql_elephant.svg" alt="PostgreSQL logo" >}}

### Recreating the Cluster

We want to recreate our cluster while **preserving our Vault and Concourse
data** (we want to recreate our GKE regional cluster as a zonal cluster to take
advantage of the [GKE free
tier](https://cloud.google.com/kubernetes-engine/pricing#cluster_management_fee_and_free_tier)
which saves us $74.40 per month).

Note: when we say, "recreate the cluster", we really mean, "recreate the
cluster". We destroy the old cluster, including our worker nodes and persistent
volumes.

### Backup Vault

In the following example, our storage path is `/vault/data`, but there's a
chance that yours is different. If it is, replace occurrences of `/vault/data`
with your storage path:

```zsh
 # Find the storage path
kubectl exec -it vault-0 -n vault -- cat /tmp/storageconfig.hcl # look for storage.path, e.g. "/vault/data"
 # We need to do this in two steps because Vault's tar is BusyBox's, not GNU's
kubectl exec -it -n vault vault-0 -- tar czf /tmp/vault_bkup.tgz /vault/data
 # We encode it in base64 to avoid "tar: Damaged tar archive"
kubectl exec -it -n vault vault-0 -- base64 /tmp/vault_bkup.tgz > ~/Downloads/vault_bkup.tgz.base64
```

Note: this backup is very specific to our configuration; if your configuration
is different (e.g. Consul storage backend, Disaster Recovery Replication
enabled), then refer to the [Vault
documentation](https://learn.hashicorp.com/tutorials/vault/sop-backup).

Check that the backup is valid (that the .tar file isn't corrupted):

```zsh
base64 -d < ~/Downloads/vault_bkup.tgz.base64 | tar tvf -
```

### Backup Concourse CI's Database

_Note: the name of our Helm Concourse CI release is "ci-nono-io" (its URL is
<https://ci.nono.io>). Remember: when you see it, substitute the name of your
Helm Concourse CI release. You'll see the release name in pod names
("ci-nono-io-postgresql-0"), secrets ("ci-nono-io-postgresql"), etc. Similarly,
our Helm Vault release's name is "vault". Yours is probably the same._

Now we move onto backup up Concourse. In the command below, our Concourse CI's
PostgreSQL's pod's name is `ci-nono-io-postgresql-0`. Substitute your pod's
name.

```zsh
 # the postgres user "concourse"'s password is "concourse"
echo concourse | \
  kubectl exec -it ci-nono-io-postgresql-0 -- \
  pg_dump -Fc -U concourse concourse \
  > ~/Downloads/concourse.dump
```

Check that the backup is valid. The following command should complete without
errors:

```zsh
pg_restore -l ~/Downloads/concourse.dump
```

### Recreate the Cluster

We burn our cluster to the ground & recreate it from scratch:

```zsh
terraform destroy
...
```

### Restore Vault

We've deployed Vault (`helm install vault hashicorp/vault ....`), now let's
restore our old vault's data:

```zsh
 # Confirm the storage path
kubectl exec -it vault-0 -n vault -- cat /tmp/storageconfig.hcl # look for storage.path, e.g. "/vault/data"
kubectl exec -it vault-0 -n vault -- sh -c "rm -r /vault/data/*"
kubectl exec -it -n vault vault-0 -- \
  sh -c "base64 -d | tar xzvf -" < ~/Downloads/vault_bkup.tgz.base64
```

At this point we can browse to our vault (ours is <https://vault.nono.io>) and
unseal it. Our original unsealing keys should work. We log in using our root
token and browse to make sure our secrets are there.

### Restore Concourse

We've deployed Concourse (`helm install ci-nono-io concourse/concourse ...`),
but haven't logged in or configured any pipelines. The install is pristine.
Let's restore the database. First, let's get the postgres database's postgres
user's password. Our secret is `ci-nono-io-postgresql`; substitute yours
appropriately in the following command:

```zsh
kubectl get secret ci-nono-io-postgresql -o json \
  | jq -r '.data."postgresql-postgres-password"' \
  | base64 -d
```

In our case, the postgres user's password is "uywhz4bUkJ".

Our PostgreSQL pod's name is `ci-nono-io-postgresql-0`; substitute yours in the
command below:

```zsh
kubectl cp ~/Downloads/concourse.dump ci-nono-io-postgresql-0:/tmp/concourse.dump
kubectl exec -it ci-nono-io-postgresql-0 -- bash
psql --username=postgres # enter the password from above
```

Now let's drop the old database. Note that we must first disallow additional
database connections and terminate existing database connections before dropping
the database:

```sql
UPDATE pg_database SET datallowconn = false WHERE datname = 'concourse';
SELECT pg_terminate_backend (pid)
    FROM pg_stat_activity
    WHERE pg_stat_activity.datname = 'concourse';
DROP DATABASE concourse;
CREATE DATABASE concourse;
EXIT;
```

Use the postgres password again when prompted below:

```zsh
pg_restore --dbname=concourse --username=postgres /tmp/concourse.dump
```

Now browse to your Concourse CI. Try logging in. Try kicking off a build. If
your resources are having trouble, yielding messages such as "run check: find or
create container on worker ... disappeared from worker", wait a few minutes. It
should clear up on its own.

### Troubleshooting

If you uninstall Concourse CI, `helm uninstall ...`, remember to delete any
lingering Persistent Volume Claims (pvc), `kubectl delete pvc ...`, before
reinstalling, lest your postgresql-postgres-password secret becomes out-of-sync
with the actual password.

If, when triggering a Concourse job that depends on a Vault secret, the job
aborts with the error `failed to interpolate task config: undefined vars:`, then
you probably forgot to `--set secrets.vaultAuthParam=...` when `helm install
ci-nono-io concourse/concourse ...`. Fix by running `helm upgrade ... --set
secrets.vaultAuthParam=...`

### References

- [How to Backup PostgreSQL Database for Concourse
  CI](https://medium.com/hoonio/how-to-backup-postgresql-database-for-concourse-ci-15c3370af059).
  We have mixed feelings about this post: on one hand, they do a nice
  description of backing up the Concourse CI database; on the other hand, they
  never bother restoring the database to make sure their procedure is correct
  (spoiler: it isn't).
