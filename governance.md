# Governance & Security Notes

This is the "why we set things up the way we did" doc — how we handle secrets,
who can access what, and how patient data is kept safe. The short version is
that we were handed a fairly locked-down Azure account, and instead of fighting
it, we designed around it. Most of what follows is us turning a constraint into
a deliberate, defensible choice.

---

## 1. The permission situation we're working within

The person deploying this has **Contributor** rights (can create and manage
resources) but *not* the higher admin rights — specifically, no ability to hand
out permissions to other identities (no RBAC role assignments), and deleting
things needs manager sign-off. We also can't see the billing/cost screens.

That one missing power — granting roles — is the thing that shaped several
decisions below. Wherever the "textbook best practice" needed a role grant we
can't make, we picked a solid alternative that works with what we actually have,
and wrote down why.

---

## 2. How we handle secrets

- Every real secret lives in **Azure Key Vault** (`primaryddevkv`).
- The vault runs in **access-policy mode**, not RBAC mode. This was a
  deliberate choice: access policies are something our Contributor rights *can*
  manage, whereas RBAC-based vault access would need role grants we can't do.
  This one setting is what lets the app read the vault later without us ever
  needing admin rights.
- We put the secret *values* in by hand (`az keyvault secret set`) rather than
  through Terraform. That's on purpose — it means the actual passwords never get
  written into the Terraform state file, which would otherwise store them in
  plain text.
- Right now the vault holds exactly three secrets — the Neo4j password, the
  Pinecone key, and our backend API key. Only things that genuinely grant access
  go in here. Everything else (URLs, IDs, settings) is not secret and is handled
  as plain env vars.

We tested the whole thing with a throwaway secret during setup — write and read
both worked.

---

## 3. Who can touch the vault (kept deliberately small)

We grant vault access one identity at a time, each as its own explicit rule in
Terraform (never as inline blocks on the vault itself — mixing the two styles
makes Terraform fight itself, which we hit and fixed).

- **The engineer (admin):** full secret rights (read, write, delete, purge).
- **The backend app:** read-only (Get/List). It reads its secrets at runtime
  using its own built-in identity — which works precisely because the vault is
  in access-policy mode.
- **The frontend app:** no vault access at all, on purpose. The UI doesn't need
  secrets, so it doesn't get any.
- **The developer (only if needed for local testing):** read-only, and only if
  granted deliberately — noted with a date and removed once it's not needed.

The principle throughout: give each thing the least it needs, nothing more.

---

## 4. How each service logs in (given we can't do RBAC)

- **Key Vault** — the app logs in with its built-in identity. Works, because
  access policies don't need a role grant. This is our preferred path.
- **Blob Storage** — only supports the role-grant style of identity login, which
  we can't set up. So we fall back to a key: the storage key goes in Key Vault
  and the app reads it. (Waiting on the developer to switch his code to this.)
- **Doc Intelligence / Foundry** — same story; we use a key or connection string
  where the identity path would need a role grant.

The trade-off is "a few keys in one locked vault" instead of "zero secrets
anywhere." We're comfortable with that — it's fully within our permissions, and
every key stays centralized, out of the code, and out of the logs.

---

## 5. The Terraform state file

- The state file lives in its own private, versioned storage container, kept
  separate from any application data.
- Because state can contain sensitive values, that container is locked down (no
  public access, encryption in transit) and access is restricted.
- If company policy switches off key-based access to that storage account, we
  switch Terraform to log in with Azure AD instead (using the storage data role
  the engineer already has) — no shared key needed.

---

## 6. Patient data boundary (this is the important one)

- Patient chart data only ever lives in Blob Storage and inside our own compute
  (the container apps).
- It is **never** written to Neo4j, Pinecone, logs, evaluation reports, or any
  outside service. The knowledge stores only ever hold ICD-10 reference
  material — never anything about a real patient.
- All the test/golden cases are made-up (synthetic) data.

---

## 7. Keeping it auditable

- Everything is built through Terraform — nothing clicked together by hand in
  the portal — so the whole environment can be rebuilt from code, and anyone can
  read the code to see exactly who has access to what.

---

## How to keep this doc alive

Update this whenever a security-relevant decision changes:

- **An auth method gets decided** (blob, Foundry) -> update section 4 and say
  which key we ended up storing.
- **Someone new gets vault access** -> add them to section 3 with the date and
  the reason; remove them here when access is revoked.
- **A new external service joins** -> note in section 6 that no patient data
  flows to it.
- **Before the demo** -> tighten `CORS_ORIGINS` (it's `*` right now, which is
  fine for building but too open for a demo) and note it here.
- **At the end** -> decide whether any temporary dev access to the vault should
  be removed, and record that it was.

---

## Still open (clear these as they resolve)

- Lock in the blob/Foundry auth method (key-based expected) and record exactly
  which secrets that added.
- Tighten `CORS_ORIGINS` from `*` before the demo.
- Decide whether the developer's dev-time vault access (if we grant it) gets
  removed before the final.
