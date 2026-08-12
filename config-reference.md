# Configuration Reference

This is our "which value goes where" cheat sheet. Any time someone asks *"what's
the Cosmos connection called again?"* or *"does the app read this from the vault
or just as a setting?"*, the answer is here. Keeping this accurate is what stops
the classic failure where the infra sets a value under one name and the code
looks for it under another.

The simple rule we follow: **if leaking a value would let someone into
something, it's a secret and it lives in Key Vault. Everything else is just a
setting and gets passed to the app as a plain environment variable.**

> Heads-up on names: a lot of these resources were created through Copilot from
> short prompts, so the odd name might read slightly differently in Azure than
> here. If a name matters, check it in Azure first.

---

## The environment at a glance (dev)

| Thing | Name / value |
|---|---|
| Resource group | `GENAI-B5-Project-T3-RG` (company-provided, East US) |
| Key Vault | `primaryddevkv` |
| Storage account | `genaib5t3cases` |
| Blob container we use | `charts` |
| Old container, unused | `cases` — leftover; check with dev before touching |
| Queue | `cases-queue` |
| Container Registry | `primaryddevacr` (`primaryddevacr.azurecr.io`) |
| Document Intelligence | `primaryddev-docintel` |
| Cosmos DB account | `primaryddev-cosmos` |
| Cosmos database | `Icd10_kb` |
| Cosmos container | `guideline_chunks` (vector, 3072-dim, cosine) |
| Backend app | `primaryddev-backend` |
| Frontend app | `primaryddev-frontend` |
| Terraform state storage | `genaib5t3tfstate` / container `tfstate` |

---

## The secrets (they live in Key Vault: `primaryddevkv`)

Set by hand with `az keyvault secret set` (never through Terraform, so values
never land in the state file). All loaded with real values.

| Secret name | Who reads it | What it's for |
|---|---|---|
| `neo4j-password` | backend | Neo4j Aura database password |
| `pinecone-api-key` | backend | Pinecone key (being phased out — Cosmos replacing it) |
| `backend-api-key` | backend + frontend | Password guarding our API; must match in backend, frontend, and local `.env` |
| `cosmos-connection` | backend | Cosmos DB connection string (AccountEndpoint + AccountKey) |

Possible future secrets (only if that auth decision lands on key-based):

| Secret name | Added if... |
|---|---|
| `storage-key` | blob uses key auth instead of managed identity |
| `foundry-connection` | Foundry code uses a key/connection string |

---

## The plain settings (environment variables on the app)

Not sensitive — addresses, IDs, names, toggles. Set directly on the backend
container. Blanks are the developer's external services.

| Setting | Value | Note |
|---|---|---|
| `AZURE_KEY_VAULT_URI` | `https://primaryddevkv.vault.azure.net/` | |
| `AZURE_STORAGE_ACCOUNT_URL` | `https://genaib5t3cases.blob.core.windows.net` | |
| `AZURE_BLOB_CONTAINER` | `charts` | |
| `AZURE_JOBS_PREFIX` | `jobs` | prefix inside container, not a resource |
| `AZURE_CHARTS_PREFIX` | `charts` | prefix, not a resource |
| `AZURE_DOC_INTELLIGENCE_ENDPOINT` | `https://primaryddev-docintel-1e963.cognitiveservices.azure.com/` | |
| `AZURE_COSMOS_ENDPOINT` | `https://primaryddev-cosmos.documents.azure.com:443/` | confirm exact env var name with dev |
| `COSMOS_DATABASE` | `Icd10_kb` | confirm env var name with dev |
| `COSMOS_CONTAINER` | `guideline_chunks` | confirm env var name with dev |
| `AZURE_FOUNDRY_PROJECT_ENDPOINT` | _(dev provides)_ | his Foundry project |
| `FOUNDRY_AGENT_STRUCTURING_ID` | _(dev)_ | id, not a secret |
| `FOUNDRY_AGENT_REASONING_ID` | _(dev)_ | |
| `FOUNDRY_AGENT_REVALIDATION_ID` | _(dev)_ | |
| `NEO4J_URI` | _(dev)_ | `neo4j+s://...databases.neo4j.io` |
| `NEO4J_USER` | `neo4j` | |
| `PINECONE_GUIDELINES_INDEX` | `icd10-guidelines` | phasing out |
| `PINECONE_INDEX_TERMS_INDEX` | `icd10-index-terms` | phasing out |
| `GUIDELINE_EMBEDDING_MODEL` | `all-MiniLM-L6-v2` | local model |
| `INDEX_EMBEDDING_MODEL` | `cambridgeltl/SapBERT-from-PubMedBERT-fulltext` | local model |
| `REQUIRE_API_KEY` | `true` | |
| `CORS_ORIGINS` | `*` | tighten before demo |
| `MAX_UPLOAD_MB` | `32` | |
| `MAX_REASONING_ATTEMPTS` | `2` | |

Note: the 3072 vector dimension suggests the dev may also move to Azure OpenAI
embeddings (text-embedding-3-large is 3072). If so, an Azure OpenAI account +
embedding deployment gets added later.

---

## Same values, two different homes

The app code always reads **environment variables** — it never calls the Key
Vault SDK directly.

| When the app runs... | Secrets come from... | Plain settings come from... |
|---|---|---|
| On the developer's laptop | his `.env` file | his `.env` file |
| In Azure | Key Vault (injected as env vars by Container Apps) | app settings |

---

## Auth methods per service (all key-based — no RBAC available)

| Service | How the app authenticates |
|---|---|
| Key Vault | managed identity (access policies — works without RBAC) |
| Blob Storage | key-based (storage key from vault) — managed identity blocked (needs RBAC) |
| Cosmos DB | connection string (`cosmos-connection`) — managed identity blocked (needs RBAC) |
| Doc Intelligence | key-based (pending confirmation) |
| Foundry | key/connection string (pending confirmation) |

The recurring rule: we cannot grant RBAC roles in this tenant, so every service
except Key Vault authenticates via a key/connection-string stored in the vault.
Managed identity (`DefaultAzureCredential`) fails with an RBAC error — the dev
must use connection-string/key auth in code.

---

## How to keep this doc alive

Update when:
- **You add or rename a secret** -> update the secrets table.
- **A new setting appears in the dev's config** -> add it with its value.
- **An auth decision lands** -> move a "future" secret into the real table and
  update the auth-methods table.
- **A resource is renamed** -> fix it in the "at a glance" table and everywhere
  its name appears as a value.
- **Before the demo** -> sanity-check every value; this doc is what the wiring
  is built from.

Re-check the vault any time:
`az keyvault secret list --vault-name primaryddevkv --query "[].name" -o table`

---

## Still open

- Confirm the exact env var names the dev's code reads for Cosmos (endpoint,
  connection, database, container).
- Blob and Foundry auth: confirm key-based, add `storage-key` /
  `foundry-connection` if needed.
- Pinecone being replaced by Cosmos — keep `pinecone-api-key` until his
  migration is fully done.
- Confirm whether the dev's code creates the Cosmos container itself or uses the
  one Terraform made.
- Confirm whether audit/artifact writing uses `cases` or only `charts`.
