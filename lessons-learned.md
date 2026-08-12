# Lessons Learned — Infra / DevOps

A running record of the things that surprised us, went differently than planned,
or are just worth remembering. Handy for the write-up, and for anyone who wants
to reuse this setup on another project.

---

## The constraints ended up shaping the whole design

We were working in a locked-down Azure account: Contributor rights but no admin,
no ability to grant roles, and deletions need sign-off. Rather than treating
that as a wall, we redesigned around each limit:
- Key Vault runs in **access-policy mode** instead of RBAC, because we can
  manage access policies but can't make role grants.
- Services that only support role-based identity login (Blob, maybe Doc
  Intelligence / Foundry) fall back to **key-based login**, with the keys kept
  in Key Vault.
- Secrets go into the vault **by hand**, so their values never land in the
  Terraform state file.

The big takeaway: we checked our permissions *before* building, one rung at a
time (can I log in? what's my role? can I create one test resource?). That meant
every wall we hit came with a clear, specific fix or request — instead of a
confusing failure halfway through a deploy.

---

## The real app was different from the original plan

Once we saw the developer's actual config, a few things didn't match the
original design docs:
- **Pinecone** (an outside service) for vector search, not Azure AI Search — so
  there's nothing to build in Azure for it beyond storing its API key.
- **AI Foundry agents** that the developer had already created himself (we just
  reference them by ID), rather than us deploying models. Those agents aren't a
  Terraform thing — they're his app config, and he owns them.
- The blob container is called `charts` (what his code expects), not `cases`
  (what the original spec said). We matched his code.

Lesson: get the developer's real config early and build to what the *code*
actually expects, not to the original plan.

---

## Local dev vs cloud caught us out once

The developer hit a 403 from Key Vault while working locally. The real issue
wasn't permissions — it was that local code shouldn't be calling Key Vault at
all. The clean pattern is: the code reads plain environment variables
everywhere, and the *surroundings* provide them — a `.env` file on his laptop,
Key Vault-injected settings in Azure. Same code both places, and no laptop ever
needs vault access.

---

## Terraform things worth remembering

- **Provider config only goes in the env folders, never in the modules.** The
  modules just say "I need the Azure provider" and get it handed down.
- **Don't mix the two ways of granting vault access.** An inline access policy on
  the vault plus separate access-policy resources makes Terraform remove and
  re-add them on every run. We hit this and fixed it by using separate resources
  only.
- **`terraform import` saved us once.** After switching the admin access policy
  from inline to a separate resource, it already existed in Azure, so we imported
  it instead of trying to create a duplicate.
- **Placeholder images are your friend.** We built and tested the whole
  container-app setup with a Microsoft "Hello World" image, so everything was
  proven before the real app images existed. Swapping to the real ones is a
  two-line change.
- **Copilot as the file-writer worked well** — with tightly-scoped prompts and a
  habit of asking it to show the full file every time. It once quietly dropped a
  module call during an edit (caught by the show-the-file check), and separately
  it caught a real deprecation and the access-policy conflict on its own. It
  writes files; we run the commands and read what Azure says back.

---

## It stayed cheap the whole time

Everything is either usage-billed or scales to zero, so an idle environment
costs a few cents a day. That's why we felt fine leaving it running over a
weekend instead of tearing it down — which also would've run into the
deletion-approval rule anyway.

---

## Keeping this doc alive

Just add to it as you go. Every time something surprises you, breaks in an
interesting way, or you make a "we chose X over Y because..." decision, drop a
few lines in. It's much easier to jot it down in the moment than to reconstruct
the whole story at the end when the write-up is due.
