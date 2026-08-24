# App Manager for Microsoft Entra

One complete register of every app registration in your Microsoft Entra tenant: what it is for, who is
accountable for it, and when its credentials run out. You hear about an approaching expiry in time to plan
the renewal, instead of discovering it when an application has already stopped working.

**This repository holds releases and nothing else.** The product's source is not here.

- **This page describes 1.0.0.** It is generated from that release's own guide — an older release's
  page is the `INSTALL.md` attached to it.
- **Every release:** <https://github.com/entercloud-cz/entra-app-manager-releases/releases>
- **The image:** `ghcr.io/entercloud-cz/entra-app-manager:v1.0.0`

---

This is the install path for a deployment **you** run, in **your** Azure subscription, watching **your** Entra
tenant. EnterCloud's own deployments are operated a different way — through pipelines, with a deploying principal
you do not have and should not need — and nothing in this guide assumes any of it.

Everything here is performed by one script, `install.ps1`, in eight phases. The phases exist because the
**rights differ** — Azure, Entra, Exchange and the database are four different administrators in most
organisations, and this way each runs their own part and nobody is handed a right they do not need.

---

## 1. What you are installing

**Watch and notify.** The product reads your directory and keeps a register of every App registration: its
credentials, their expiry, who owns it, what it can do. It raises findings — a secret expiring in three weeks, a
registration nobody claims, something that changed in the tenant. It tells the right people by mail. It writes an
append-only journal of everything it observed and everyone who configured it.

**It writes nothing to Entra.** Every record it makes ends in its own database. It asks for no permission that
could change your directory, so a compromise of this deployment cannot change your directory either — not because
our code declines to, but because Entra was never asked.

What it therefore does **not** have in this mode: requests, approvals, provisioning, credential collection. Those
surfaces are not greyed out — they are absent, and their addresses answer that this deployment does not have them.
They belong to the next mode, which is a different permission grant and a different conversation.

### The four permissions, and what each one buys

| Permission | Type | What it buys | Refusing it costs |
|---|---|---|---|
| `Application.Read.All` | application, read | the register itself: applications, their credentials and expiry dates, their owners, their service principals | everything — this is the product |
| `User.Read.All` | application, read | turning an owner into a person with an address, and offering your users in every field that names somebody | notices have nobody to go to, and fields that name a person offer only what you record by hand |
| `Group.Read.All` | application, read | offering your mail-enabled groups in those same fields — "the owning team" is usually a distribution list | groups cannot be named; users and recorded contacts still can |
| `Mail.Send` | application, send | sending the notices, as **one** mailbox and no other | nothing leaves the product; findings sit on a screen nobody opens |

**Not requested, and this is the point:** no `Application.ReadWrite.OwnedBy`, no `Application.ReadWrite.All`,
nothing that writes. The mode selector inside the product is a promise about behaviour; **the permission grant is
the boundary.** The install script reports it if the two ever disagree — if this deployment is found holding a
write permission, it says so at the end of the permissions phase, with what to remove.

Two narrower notes worth having in advance:

- `User.Read.All` and `Group.Read.All` are each **refusable on their own**, and the product states on the surface
  which kind it was refused rather than showing an empty list — a picker with no groups and a tenant with no groups
  look identical otherwise. A **contact recorded inside the product** needs no Graph permission at all, so a
  deployment granted neither can still name who to write to.
- `Directory.Read.All` would cover both and also read your devices, your directory roles and the rest. Do not grant
  it. The two narrow ones are the whole ask.

### What gets created in your subscription

One resource group, `rg-eam-prod` (the names come from `-Workload` and `-EnvironmentName`; these are the defaults):

| Resource | Why |
|---|---|
| Two user-assigned managed identities | one for the web app, one for the background worker. **The web one holds no Graph permission, ever** — that is what stops a compromise of the internet-facing tier from reaching your tenant |
| Container Apps environment, one web app | the product. The template pins a minimum of one replica, so this is a fixed monthly cost rather than a usage-based one |
| Four Container Apps jobs | a scheduler on a one-minute cron that only decides what is due, two workers that do the work, and a manual migrate job |
| Azure SQL — server and one database | the register, the journal, everything recorded. **Entra-only authentication, no password anywhere** |
| Storage account | the work queues, and the key ring that keeps people signed in across a restart. Shared-key access disabled, so there is no connection string to leak |
| Container registry | created by the template. It holds the product image when the image has to be copied into your subscription (§3); with an anonymously pullable reference nothing is stored in it |
| Log Analytics workspace | the container logs |

**Two of those cost money whether anybody uses the product or not**: the database (Standard S0 by default) and the
one web replica. Everything else is small or usage-based. Price them for your region in the Azure pricing
calculator before you commit — this document deliberately quotes no figure it cannot keep current.

**No secret exists in a deployed stamp.** The database, the queues, the key ring and Microsoft Graph are all
reached with managed identity, and the sign-in registration authenticates with a federated credential rather than
a client secret. The application refuses to start if a connection string ever arrives with a password in it.

---

## 2. Before you start

### The tenant is not a free choice

The worker authenticates to Graph as a **managed identity**, and a managed identity can only be granted
permissions in the tenant that owns its subscription. So this deployment watches the Entra tenant that owns the
subscription you install it into. One deployment, one tenant — there is no multi-tenant mode and no row anywhere
carrying a tenant identifier.

### Who has to be involved

| Phase | Where | Minimum right |
|---|---|---|
| `preflight` | Azure | Reader, plus **Contributor** if a resource provider still needs registering (a fresh subscription usually needs `Microsoft.App` and `Microsoft.Sql`) |
| `registration` | Entra | **Cloud Application Administrator**. (Application Developer may be enough, since whoever creates the registration becomes its owner — we have not measured it, so it is not what this document tells you to arrange.) |
| `deploy` | Azure | **Owner** on the subscription — or Contributor **plus** User Access Administrator / Role Based Access Control Administrator. The deployment creates role assignments (pull rights on the registry, three queue roles), and Contributor alone cannot create a role assignment |
| `signin` | Entra | the same as `registration` |
| `database` | the database | whoever the deploy named as the SQL Entra administrator — by default, you — and **TCP 1433 outbound** from the machine you run it on. No directory right at all |
| `permissions` | Entra | **Privileged Role Administrator** or Global Administrator. Granting an application permission is consent, which is a larger right than deploying |
| `mail` | Exchange Online | **Exchange Administrator**, for the mailbox, the group and the access policy |
| `verify` | — | none |

One person holding all of them can run the whole thing in one command. If that person does not exist in your
organisation — and in most organisations they should not — run the phases separately; every phase is re-runnable
and reads the live state rather than a file left behind by the last one.

### On the machine you run it from

- **PowerShell 7.2 or newer** (`pwsh`). Windows PowerShell 5.1 is refused.
- `Az.Accounts` and `Az.Resources` — always.
- `SqlServer` — for the database phase.
- `ExchangeOnlineManagement` — for the mail phase.

```powershell
Install-Module Az.Accounts, Az.Resources, SqlServer, ExchangeOnlineManagement -Scope CurrentUser
```

Each phase names the module it needs when it is missing, so you do not have to install the ones you will not reach.

### What to have decided

| Answer | Notes |
|---|---|
| Subscription and region | one region holds the whole stamp |
| The container image | the reference EnterCloud gives you, **with its version tag**. See §3 |
| The first Administrator | a person who signs in, applies the schema and configures the deployment. It grants them nothing in Entra |
| The database administrator | defaults to you. A **group** is better: a person leaves, and only this principal can create the database users |
| The notice mailbox | a shared mailbox that already exists. **Its display name is what every recipient sees** — see §5 |
| The sender group | a mail-enabled security group. It is what the Exchange policy is scoped to |
| An address outside that group | used once, to prove the policy actually refuses somebody. Any other mailbox in your tenant |

---

## 3. Getting the release

Everything you need is attached to one release:

**<https://github.com/entercloud-cz/entra-app-manager-releases/releases/latest>**

| File | What it is |
|---|---|
| `install.ps1` | the installer this guide is about. Already pointed at the image of the release it came from |
| `mainTemplate.json` | the deployment itself, compiled from the product's infrastructure code by the release run — so nothing has to be installed to read or deploy it |
| `INSTALL.md` | this guide, as it stood for **that** version |
| `NOTES.md` | what the release changes, whether it migrates the schema and what that costs |
| `SHA256SUMS` | checksums for the four above |

```powershell
# in an empty directory
$release = "https://github.com/entercloud-cz/entra-app-manager-releases/releases/latest/download"
foreach ($file in 'install.ps1', 'mainTemplate.json', 'INSTALL.md', 'NOTES.md', 'SHA256SUMS') {
    Invoke-WebRequest "$release/$file" -OutFile $file
}

# and check them before running anything
(Get-Content SHA256SUMS) | ForEach-Object {
    $expected, $name = ($_ -split '\s+', 2)
    $actual = (Get-FileHash $name.Trim() -Algorithm SHA256).Hash.ToLower()
    "{0,-20} {1}" -f $name.Trim(), $(if ($actual -eq $expected) { 'ok' } else { "MISMATCH: $actual" })
}
```

**Read `NOTES.md` before you install or upgrade.** It is the one place that says whether this release needs
something from you before you start, and whether it can be rolled back afterwards.

**An older version is a different release, not an older link.** Each one keeps its own installer, its own template
and its own copy of this guide, because a guide describing a version you are not installing is worse than none.

### The image

The product is one container image; the web app and all four jobs run it. It has to be **pullable by your
subscription before the deployment runs**, and there are two shapes that can take.

**A public reference, which is the ordinary case and needs no credential at all:**

    ghcr.io/entercloud-cz/entra-app-manager:v<version>

`install.ps1` from a release already points at the version it was published with, so you do not have to pass
`-Image` at all. The release notes carry the **digest** beside the tag; pass that instead if you would rather pin
the artefact than the name — a tag can be re-pointed and a digest cannot.

**A private reference plus a pull token.** Then the image has to be copied into the registry this stamp creates —
the stamp's containers authenticate pulls against their own registry and nowhere else. Pass `-ImportFrom` with the
source reference and the token, and the deploy phase does it in three steps: create the stamp, copy the image into
its registry server-side (seconds, and the image never touches your machine), deploy again pointing at the copy.
The first of those passes runs the platform's placeholder image, which the script otherwise refuses; it says so
while it does it.

**The script and the image are a pair.** `install.ps1` from a release carries the reference it was published with,
so `-Image` is optional; give it a different one and it stops rather than deploying, because a newer image can need a
step an older script does not perform. If that is deliberate — a rollback, or an image you copied into your own
registry — pass `-AllowVersionMismatch`.

**Always use a version tag, never `latest`.** A moving tag makes "what is installed" and "what do I roll back to"
unanswerable, and the script warns when it sees one.

---

## 4. The install

The whole thing, asking for what it needs as it goes:

```powershell
./install.ps1
```

Read it before you run it, or read what it would do first — this changes nothing at all:

```powershell
./install.ps1 -WhatIf
```

Split across the people who hold the rights. **The order matters and the script enforces it** — each of these
refuses with what is missing rather than proceeding, so a wrong order costs a message and not a half-install:

```powershell
# 1. the Entra administrator — first, because the web app refuses to start without this registration's client id
./install.ps1 -Only registration -AdministratorUpn anna@contoso.com

# 2. the subscription owner — the stamp itself
./install.ps1 -Only preflight,deploy -SubscriptionId <id> -Location swedencentral -Image <reference>

# 3. the Entra administrator again — the redirect URI and the federated credential need the deployed hostname
#    and the deployed identity, which is why this is not part of step 1
./install.ps1 -Only signin

# 4. whoever the deploy named as the database administrator, from a machine that can reach TCP 1433
./install.ps1 -Only database

# 5. the Privileged Role Administrator — consent
./install.ps1 -Only permissions

# 6. the Exchange administrator — in the same sitting as step 5, see §5
./install.ps1 -Only mail -NoticeMailbox appmanager@contoso.com `
                         -SenderGroup eam-notice-senders@contoso.com `
                         -OutsideAddress someone.else@contoso.com

# 7. anybody
./install.ps1 -Only verify
```

Steps 1–3 can be one command (`-Only registration,preflight,deploy,signin`) when the same person holds both rights;
that is what plain `./install.ps1` does.

`-NonInteractive` refuses instead of prompting, for a pipeline. `-Plain` swaps the icons for ASCII on a console
that cannot draw them.

### What each phase does

| Phase | What it does, and what it verifies afterwards |
|---|---|
| `preflight` | signs you in, confirms which tenant this will watch, registers the resource providers the deployment needs |
| `registration` | creates the app registration people sign in against, declares the four app roles, creates its enterprise application, and assigns **Administrator** to the person you name. **No client secret is created, here or ever** |
| `deploy` | deploys the whole stamp — one pass, or three steps if the image has to be copied in. Refuses a placeholder image |
| `signin` | reads the hostname off the running app, registers it as the redirect URI, and creates the federated credential that lets the web app authenticate **without a secret** |
| `database` | creates one database user per managed identity, by SID rather than by directory lookup, and then **asks the database whether they are there**. Opens a firewall rule for your address and removes it again even if the step fails |
| `permissions` | grants the four Graph permissions, then lists anything held **beyond** them |
| `mail` | checks the mailbox and the group, creates the Exchange application access policy, and proves it both ways |
| `verify` | reads `/healthz` and `/api/schema` off the running deployment and prints what is left for a human |

### Why the registration comes before the deployment

The web container **refuses to start** without the client id of the registration people sign in against — a
deployment serving pages nobody can sign in to is worse than one that stops and says why. Creating the
registration first is what makes the deployment a single pass. The redirect URI has to wait until afterwards,
because the hostname is generated by the platform and is read off the running app rather than assembled from a
pattern.

### Until the permissions are consented

The projection fails with an authorisation error and the product's job health surface says so. That is the bound
working, not a fault. Nothing else about the deployment is affected — you can sign in, and the register is simply
empty.

---

## 5. Mail — one mailbox, and the policy that makes it one

Notices go out through Graph as **one shared mailbox**. Three things have to be true, and only one of them is
visible from inside the product.

### The mailbox

It has to exist before the install; the script will not create it. That is deliberate: **Exchange overwrites any
`From` header the product sets, so what every recipient sees above mail about their credentials is the mailbox's
own display name.** A mailbox somebody created as "Azure Info" sends every notice as *Azure Info*. Name it after
the product:

```powershell
New-Mailbox -Shared -Name "App registration manager" -DisplayName "App registration manager" `
            -PrimarySmtpAddress appmanager@contoso.com
```

### The group

An Exchange application access policy can only be scoped to a **group**, and a group is also what survives the
second mailbox — adding a sender becomes a membership rather than a second policy for somebody to discover a year
later. The script offers to create it if it does not exist:

```powershell
New-DistributionGroup -Name "eam-notice-senders" -Type Security -Members appmanager@contoso.com
```

### The policy — and this is the actual control

**App-only `Mail.Send` is permission to send as anybody in your tenant.** Nothing in the product narrows it and
nothing in the product can see that it has not been narrowed. An Exchange Online application access policy is the
only thing that does, and Microsoft Graph has no API for one — it is Exchange PowerShell or nothing, which is why
this is a deployment step rather than a setting.

The script applies it and then **verifies it in both directions**: the mailbox inside the scope must test
`Granted`, and an address outside it must test `Denied`. That second check is the point rather than a flourish.
Exchange can take up to an hour to bring a policy into force, and **until it does, its absence looks exactly like
everything working** — because every notice is delivered either way.

> ⚠ **The dangerous state is the middle one:** `Mail.Send` granted and no policy yet. Nothing warns, every notice
> goes out, and this deployment can send as any mailbox in your tenant. Run the `permissions` and `mail` phases in
> **one sitting**, not on two afternoons.

If the app already has a policy scoped to some other group, the script stops rather than adding a second one. Two
scopes for one app is a question about intent, and a script is not the place to answer it.

---

## 6. After the install — four things, all inside the product

The script prints these as it finishes; this is the longer form.

**1. Sign in as the Administrator, and apply the schema.** A deployment whose database is behind its build serves
exactly one screen: what will be applied, what each migration is expected to cost, and a button. Read the costs
before pressing it, and have a backup you would be willing to restore — **there are no down migrations.** Rolling
the container back is supported; rolling the schema back is a restore. Nothing migrates because a container
started, which is why this is a screen and not a side effect.

**2. Administration → General.** Confirm the mode reads **Watch and notify** (it is the default). Name who this
deployment belongs to — the system administrator, who is the last recipient of every notification chain. Type the
address people open this deployment at: every notice carries one link, built from that address, and **notices stay
inactive until both it and the mailbox are set**, because a mail about credentials whose only button goes nowhere
is worse than plain text.

**3. Administration → Notifications.** Set the sender to your shared mailbox. Set who hears about an application
nobody claims. Set the three expiry thresholds — informative, warning, critical; an empty one means that notice is
off. Then **send the test mail**, which is what proves §5 end to end. The mailbox is set here rather than in
configuration on purpose: a setting that needed a redeployment to change would be a setting you had to ask us to
change for you.

**4. Administration → System.** The register is read on a cadence **you** set here, not one we deploy. Make the
projection refresh due once and watch it fill, then the expiry scan. Changing a cadence is journalled with the name
of whoever changed it.

Then hand out roles: **Auditor** for people who read the register and the journal, **Administrator** for whoever
configures the deployment. Both are assigned in Entra, on the enterprise application this install created. Somebody
signing in with no role is not locked out silently — they get a screen naming the role they lack and who to ask.
A role change takes effect at the **next token refresh**, not instantly; to cut access off immediately, disable the
account.

---

## 7. Things that will bite

- **TCP 1433 is blocked on many corporate networks**, and the database phase is the only one that needs it. It
  fails with a connection timeout that reads like a broken deployment. Run that one phase from somewhere that can
  reach it: `./install.ps1 -Only database`.
- **Only the SQL Entra administrator can create the database users.** If the deploy named a group, you must be in
  it. Nobody else — not even the subscription owner — can do this step.
- **A resource provider that is not registered fails the deployment** with `MissingSubscriptionRegistration`, which
  names the namespace and not the remedy. The preflight phase registers them; registration itself can take a few
  minutes to complete in the background.
- **The Exchange policy can take up to an hour**, and a fresh policy can still answer `Granted` for an address
  outside it while Exchange catches up. Re-run the mail phase until the outside address reads `Denied`. Do not
  treat the permission as narrowed before then.
- **A new mail-enabled group is not immediately usable as a policy scope**, for the same reason. Re-run the phase.
- **The database's tier and its maximum size are not independent.** Azure SQL Standard accepts only a discrete list
  of sizes for each service objective, and nothing in either value says so — a legal-looking pair is refused with
  `InvalidMaxSizeTierCombination` at deploy time. The defaults (S0, 30 GB) are a pair that works. If you change
  `-SqlSkuName`, read the list for your region first:
  ```bash
  az sql db list-editions -l <region> --edition Standard --service-objective <objective> --show-details max-size
  ```
- **Being behind is invisible unless somebody looks.** `https://<host>/healthz` answers with the build and commit
  that is actually serving. Nothing in the product nags you about a newer release.
- **Deleting the resource group deletes the record.** The register can be rebuilt from Entra in one refresh; the
  journal, the recorded owners and everything a person typed cannot — they exist only here. Export the journal from
  the Audit surface first (CSV or JSON) if you need to keep it.

---

## 8. Upgrading, rolling back, removing

**Upgrade** — deploy the new image, then let the product finish it:

```powershell
./install.ps1 -Only deploy,verify -Image <new reference>
```

The container starts either way. If the release carries a schema change, the deployment serves its upgrade screen
and an Administrator presses the button; `verify` says how many migrations are pending. Nothing is applied behind
your back and nothing is applied because a container restarted.

That command is short on purpose: an upgrade **reads back what this stamp already decided** rather than defaulting
it. The region comes from the resource group that exists, and the database administrator from the SQL server that
exists — so a re-run cannot quietly move the stamp to another region, and cannot hand the database's administration
to whoever happens to be running the upgrade.

**Roll back** — the same command with the previous reference. It is bounded: the schema records the oldest build
that can serve it, and a build below that floor refuses to start rather than reading columns it does not
understand. That refusal names the migration it is missing. Below the floor, the way back is forward.

**Remove** — delete the resource group, then the three things that live outside it: the app registration
(`entra-app-manager-<environment>`), the Graph permissions granted to the worker identity, and the Exchange
application access policy (`Remove-ApplicationAccessPolicy`). None of them is deleted by deleting the resource
group, and the access policy left behind refers to an identity that no longer exists.

---

## 9. What this deployment can and cannot do, in one place

- It can **read** your applications, users and mail-enabled groups, and it can **send mail as one mailbox**. That
  is the entire set.
- It **cannot write anything to Entra.** No permission it holds allows it.
- The **web container holds no Graph permission at all** — not a narrow one, none. Only the background worker
  talks to Graph, and the two run as different identities.
- **No credential value is ever stored.** For each credential the register keeps its type, name, expiry,
  fingerprint and the three-character hint Entra itself shows. A compromise of this database exposes the map of
  your identity surface and the first three characters of each secret; it does not expose a usable credential.
- **No secret exists in the deployment** — every connection is a managed identity, and the sign-in registration
  uses a federated credential.
- The **journal is append-only**, enforced where the writes happen rather than by convention: entries cannot be
  deleted and cannot be edited, and each act is recorded **before** it is attempted so a failure leaves a trace
  rather than a silence.
