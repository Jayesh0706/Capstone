# Deployment Guide

This walks through standing up the whole Azure environment from nothing. It's
written so that a future you (or a teammate) can follow it start to finish
without needing to remember what we did. Almost everything is Terraform; only a
couple of one-time secret steps use the Azure CLI.

> Names might drift slightly from this guide since some resources were created
> via Copilot — check against Azure if an exact name matters.

---

## What you need first

**Tools on your machine:** `az` (Azure CLI), `terraform`, and `git`.

**Azure access:**
- **Contributor** on the resource group `GENAI-B5-Project-T3-RG`, plus the
  storage data role (we use it to log Terraform into the state storage via
  Azure AD).
- Enough to create resources inside that group.

**What you *don't* have (and how we work around it):** no admin/RBAC rights, so
we can't grant roles to identities. That's why the vault uses access policies
and some services use key-based login instead of managed identity — see the
governance doc for the full reasoning. Also, deleting resources needs manager
approval here.

The resource group is provided by the company. Terraform only *reads* it — it
never creates or deletes it.

---

## Sanity-check your access before building

```
az account show -o table
az role assignment list --assignee <your-email> --all -o table
az group show --name GENAI-B5-Project-T3-RG -o table
```

You're checking that you show up as Contributor and that the group is reachable
(note the region it reports — everything goes in the same region).

---

## Step 1 — Set up the state storage (do this once, ever)

Terraform needs somewhere shared to keep its "memory" (the state file). But it
can't store that in a storage account it hasn't created yet — chicken and egg.
So the bootstrap folder creates just that storage account, and keeps its own
tiny state locally.

```
cd infra/bootstrap
terraform init
terraform plan      # should plan to create a storage account + container
terraform apply
```

This creates `genaib5t3tfstate` with a `tfstate` container. Run it once and then
leave it alone forever — never rename or delete it.

---

## Step 2 — Build the actual environment

```
cd infra/envs/dev
terraform init      # this time it connects to the shared state from step 1
terraform plan
terraform apply
```

This builds everything inside the company resource group:
- Storage (the `charts` and `cases` containers, plus a queue)
- Key Vault (`primaryddevkv`, in access-policy mode)
- Container Registry (`primaryddevacr`)
- Document Intelligence (`primaryddev-docintel`)
- The Container Apps environment plus the backend and frontend apps (running a
  placeholder image for now, and set to scale to zero)
- The Key Vault access rules (you as admin, the backend app as reader)

If the state storage ever starts giving 403s because policy disabled key access,
add `use_azuread_auth = true` to the backend block and run
`terraform init -reconfigure`.

---

## Step 3 — Load the secrets (once, by hand)

The developer provides these privately (or you generate the backend key). We set
them directly, not through Terraform, so they stay out of the state file:

```
az keyvault secret set --vault-name primaryddevkv --name neo4j-password --value "<value>"
az keyvault secret set --vault-name primaryddevkv --name pinecone-api-key --value "<value>"
az keyvault secret set --vault-name primaryddevkv --name backend-api-key --value "<value>"
```

Check they're all there:

```
az keyvault secret list --vault-name primaryddevkv --query "[].name" -o table
```

---

## Step 4 — Wire the app's config (still to do)

Once the developer freezes his list of env var names and the images exist:
- Point the backend app's secret env vars at the three Key Vault secrets.
- Set all the plain settings (endpoints, IDs, names, toggles) as normal app
  settings — the full list is in `config-reference.md`.
- For pulling private images from ACR (which needs a login we can't do via
  identity), turn on ACR's admin login, store those credentials in Key Vault,
  and reference them in the app's `registry` block.

---

## Step 5 — Swap in the real images (still to do)

When the backend and frontend images are pushed to ACR, set the image variables
and apply:

```
backend_image  = "primaryddevacr.azurecr.io/backend:<tag>"
frontend_image = "primaryddevacr.azurecr.io/frontend:<tag>"
```

```
terraform apply
```

Open the frontend URL afterwards — you should see the real UI instead of the
placeholder page. That's the moment the environment is functionally done.

---

## Tearing it down

`terraform destroy` from `infra/envs/dev`. Remember deletion needs manager
approval here, and this won't remove the company resource group or the state
storage (the group is only referenced, not owned by Terraform).

---

## About cost

We can't see the Cost Management screens, so the cost table is built from
Azure's published prices (see the cost table). The short version of how each
thing bills:
- Storage and Key Vault: per-operation / per-GB — basically nothing when idle.
- ACR Basic: a small fixed daily charge.
- Document Intelligence: per document — **zero when nothing's being processed**.
- Container Apps: pay-for-use, and **scale-to-zero means no charge when idle**.
- An idle environment with no traffic runs to a few cents a day.

---

## Keeping this doc alive

- **When a step changes** (new module, different order) -> update it here so the
  guide always matches reality.
- **When steps 4 and 5 get done** -> move them out of "still to do" and record
  exactly what was wired and which image tags went live.
- **When you first time the full build** -> note how long a clean
  subscription-to-running takes; that number is worth having.
- **After the demo** -> capture anything that tripped you up so the next build
  is smoother.
