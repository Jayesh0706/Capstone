# INFRA STATE — Master Reference (Primary Diagnosis Validation)

Single-file snapshot of the entire infrastructure. Paste into any AI/chat to
restore full context. Last updated: 2026-08-12 (added Cosmos DB).

> NAMING CAVEAT: resources were authored via company Copilot from scoped prompts,
> so exact resource/file/output names may differ slightly. Verify names in Azure
> before relying on them. Functionality and easy integration matter more than
> exact naming.

---

## 1. WHAT THIS PROJECT IS

Healthcare AI capstone (2-person team). Reads hospital chart PDFs and outputs
correctly sequenced ICD-10-CM diagnoses (principal + secondary) with guideline
citations. Repo/project: "Primary Diagnosis Validation".

Roles: this user = cloud/DevOps engineer (Azure infra via Terraform, secrets,
deploy, docs). Teammate = developer (all Python: FastAPI backend, worker,
pipeline, medical rules; React UI; prompts; Foundry agents; Neo4j; Cosmos/vector
code). Developer name in Teams: Kavya Chouhan.

---

## 2. HARD CONSTRAINTS (shaped every decision)

- Office laptop, restricted. HAS: Contributor, Cognitive Services Contributor,
  Storage Blob Data Contributor, Network Contributor, AI roles.
- Does NOT / will never have: Owner, RBAC-admin (roleAssignments/write), admin.
- Deletions need manager approval. No Cost Management view.
- Works via company Copilot for authoring files; hand-types between personal
  laptop (AI guidance) and office laptop.
- Company Azure access works only on the company laptop (conditional access).

IDs:
- Subscription: `c84b9ad1-e3bd-486e-ac95-d9288b9ff535`
- Tenant: `707118c9-65c0-45b3-a435-33efe37cf0cc`
- User (admin) object ID: `97befe21-c76a-4f9b-bd22-87164668ef5a`
- Backend app identity principal: `c1bf9fe5-f6fc-4692-99ac-6ac572eb876b`
- Developer object ID: `f4e63649-a156-42cf-b558-47be99fccdc9`
- Resource group (company, read-only in TF): `GENAI-B5-Project-T3-RG` (East US)

---

## 3. DEVELOPER'S REAL ARCHITECTURE (overrides older docs)

UI upload -> FastAPI backend -> blob (`charts`) -> queue -> worker -> Document
Intelligence OCR -> 3 AI Foundry agents (structuring, reasoning, revalidation;
he created them, has IDs) -> Neo4j Aura (external, his) + vector search ->
sequenced diagnosis output.

VECTOR DB CHANGE: migrating from Pinecone to **Azure Cosmos DB (NoSQL + vector
search)** for an all-Azure stack. Pinecone being phased out. His retrieval code
(Stages 3/5) is being rewritten for Cosmos — that's his work.

Embedding note: Cosmos vector dimension is 3072 = Azure OpenAI
text-embedding-3-large. Likely he's also moving to Azure OpenAI embeddings (from
local MiniLM/SapBERT). If so, an Azure OpenAI account + embedding deployment gets
added later.

Secret vs plain contract: 4 secrets now (`neo4j-password`, `pinecone-api-key`,
`backend-api-key`, `cosmos-connection`). Everything else plain env vars.

---

## 4. WHAT IS BUILT AND LIVE (all applied, verified)

Repo: `C:\Users\43327\Desktop\GenAI\projects\primary diagnosis validation\`.
Pushed to Azure DevOps. Branches: main/master (old), `dev` (integration, has
newer code from developer), feature branches. Currently working on
`feature/cosmos-infra`. TL wants: feature branch -> PR -> dev.
Structure: `infra/bootstrap`, `infra/modules/{storage,keyvault,registry,docintel,container_apps,cosmos}`,
`infra/envs/dev`, `docs/`. `.gitignore` excludes state, .env, *.tfvars.

Bootstrap (one-time, local state): storage `genaib5t3tfstate`, container
`tfstate`. NEVER rename/destroy.

envs/dev: azurerm ~>4.0, backend -> `genaib5t3tfstate/tfstate/dev.tfstate`,
`resource_provider_registrations = "none"`, backend config separate file.
Locals tags: project=primarydiagnosis, environment=dev, managed_by=terraform.
Values hard-coded in module calls (no tfvars — deliberate). RG via data block.

Applied resources (prefix `primaryddev`):
1. Storage `genaib5t3cases`. Containers `charts` (active) + `cases` (legacy,
   unused). Queue `cases-queue`.
2. Key Vault `primaryddevkv` — access-policy mode (enable_rbac_authorization =
   false). Secrets (real values): neo4j-password, pinecone-api-key,
   backend-api-key, cosmos-connection.
3. ACR `primaryddevacr` — Basic, admin_enabled=false (flip to true for pulls +
   creds to KV when swapping images).
4. Doc Intelligence `primaryddev-docintel` (FormRecognizer, S0). Endpoint
   `https://primaryddev-docintel-1e963.cognitiveservices.azure.com/`.
5. Container Apps: env `primaryddev-env`; backend `primaryddev-backend`
   (0.5cpu/1Gi, system identity, principal c1bf9fe5..., FQDN
   primaryddev-backend.gentlepond-2a87df60.eastus.azurecontainerapps.io);
   frontend `primaryddev-frontend` (0.25/0.5Gi). Both placeholder image
   quickstart, scale-to-zero, swappable via backend_image/frontend_image vars.
6. Cosmos DB `primaryddev-cosmos` — NoSQL, serverless,
   EnableNoSQLVectorSearch. Database `Icd10_kb`, container `guideline_chunks`
   (partition /id). Endpoint
   `https://primaryddev-cosmos.documents.azure.com:443/`. Vector index NOT in
   Terraform (azurerm can't express it) — dev's code creates the vector index
   at runtime. Connection string in KV as `cosmos-connection`.
7. KV access policies (all SEPARATE resources, no inline): admin (Get/List/Set/
   Delete/Purge), backend app (Get/List), developer (Get/List, object_id
   f4e63649...). Frontend has NO vault access.

NOT yet done: env-var wiring into containers, real images, ACR pull creds,
trigger wiring, observability, CI/CD pipeline.

---

## 5. KEY DECISIONS (don't relitigate)

- Monorepo + feature branch -> PR -> dev (TL's flow). No GitFlow.
- Provider config only in envs, never modules. Bootstrap flat.
- No tfvars — hard-coded in module calls (deliberate).
- Secrets set out-of-band via az CLI, never Terraform (out of state).
- Never destroy/rename genaib5t3tfstate. State never manually deleted.
- Match dev's names exactly (his config = contract).
- Don't build "maybes" (Azure AI Search, Azure embeddings) until he commits.
- Key Vault = access-policy mode (Contributor can manage; RBAC can't).
- ALL non-KeyVault services use key/connection-string auth (no RBAC for managed
  identity). Cosmos, Blob, Doc Intelligence, Foundry -> keys in vault. This is
  the recurring pattern; DefaultAzureCredential fails with RBAC error.
- Local dev: dev reads from .env (plain env vars), NOT Key Vault SDK.
- Cosmos vector index: dev's code creates it at runtime (azurerm can't; azapi
  was an option but we chose to let his code own the vector part).

---

## 6. GOTCHAS ENCOUNTERED & FIXED

- Copilot dropped a module call once -> always "show full file" after edits.
- Copilot caught storage_account_id deprecation and the KV access-policy
  inline/separate conflict.
- terraform import used after inline->separate admin policy conversion.
- Cosmos vector_embedding_policy / vector_index blocks NOT supported by azurerm
  -> removed them; dev's code creates the vector index.
- State lock got stuck after Ctrl+C interrupts -> fix: `terraform force-unlock
  <lock-id>` (NOT -lock=false). Lesson: let Terraform finish, don't Ctrl+C.
- Cosmos RBAC error for developer (Microsoft.DocumentDB/.../readMetadata) =
  his code used managed identity -> fix: use connection string from vault.
- Possible 403 on state storage if org disables shared-key -> add
  `use_azuread_auth = true`, `terraform init -reconfigure`.

---

## 7. NEXT-UP TASKS

From developer:
- Fix Cosmos auth: use connection string (cosmos-connection), not managed
  identity. [given him the string + it's in vault]
- Confirm env var names his code reads for Cosmos (endpoint/connection/db/
  container).
- Does his code create the Cosmos container, or use Terraform's?
- PDF upload path (via API or direct-to-blob?), blob auth (key-based?), worker
  (own container?), Foundry endpoint + agent IDs, Docker images ETA.

DevOps tasks:
1. [in progress] Commit + PR the Cosmos module (feature/cosmos-infra -> dev).
2. ACR: admin_enabled=true, creds -> KV, wire registry block in apps.
3. Env-var wiring into backend (KV secret refs + plain vars). The big task.
4. Image swap when ACR images land (2 lines + apply) -> infra functionally done.
5. Observability module (App Insights; deferred).
6. CI/CD pipeline (azure-pipelines.yml; SP received from admin). Build -> push
   to ACR -> deploy. Env-var wiring must be done first.

Rest of week: docs (keep updating), demo env (copy envs/dev -> envs/demo), cost
table (from published pricing — no Cost view), warmup/rollback/rehearsal.

---

## 8. STATUS SNAPSHOT

- Infra build: ~85% (construction done + live; wiring/images/pipeline pending).
- Full DevOps scope (build + CI/CD + docs + demo): ~55-60%.
- Nothing scary left — known work: wiring, CI/CD, docs, demo prep.
- "Infra done" = dev's real image runs in backend app, reads secrets + endpoints
  from wired env vars, touches blob + Cosmos + Doc Intelligence without errors.

---

## 9. WORKING RHYTHM

DevOps writes tightly-scoped Copilot prompts + explains what/why -> user runs on
office laptop -> reports back short facts or photos -> AI interprets -> next step.
Checkpoint discipline: check state list / plan summary line before proceeding.
One step at a time. Flat hand-typeable commands. Let Terraform finish (don't
Ctrl+C — causes state locks).
