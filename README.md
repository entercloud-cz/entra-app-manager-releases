# App Manager for Microsoft Entra

One complete register of every app registration in your Microsoft Entra tenant: what it is for, who is
accountable for it, and when its credentials run out. You hear about an approaching expiry in time to plan
the renewal, instead of discovering it when an application has already stopped working.

**This repository holds releases and nothing else.** The product's source is not here.

- **This page describes 3.6.0.** It is generated from that release's own guide — an older release's
  page is the `INSTALL.md` attached to it.
- **Every release:** <https://github.com/entercloud-cz/entra-app-manager-releases/releases>
- **The image:** `ghcr.io/entercloud-cz/entra-app-manager:v3.6.0`

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

### The three permissions, and what each one buys

| Permission | Type | What it buys | Refusing it costs |
|---|---|---|---|
| `Application.Read.All` | application, read | the register itself: applications, their credentials and expiry dates, their owners, their service principals — **and who has been given access to this deployment**, which is read from the sign-in registration's own role assignments | everything — this is the product |
| `User.Read.All` | application, read | turning an owner into a person with an address, offering your users in every field that names somebody, and naming the people on the access list | notices have nobody to go to, fields that name a person offer only what you record by hand, and the access list has ids without names |
| `Group.Read.All` | application, read | offering your mail-enabled groups in those same fields — "the owning team" is usually a distribution list — **and expanding a group that carries a role, all the way down** | groups cannot be named, and access granted through a group cannot be traced to the people it reaches |

**The access list needs no fourth permission and nothing configured, and both are deliberate.** Who can sign in is read from the two above:
the assignments from the same permission the register uses, the group expansion from the same one the pickers use.
What it does NOT read is Entra's own sign-in activity, which would need `AuditLog.Read.All` and an Entra ID P1
licence — so *last seen* on that list is this deployment's own record of somebody using it, and says so.

**And `Mail.Send` is NOT among them, which is the interesting part.** Sending notices is authorised in Exchange
instead, by a role assignment that names **one mailbox** — not by a tenant-wide permission narrowed afterwards.
Section 5 is that step. The reason it works this way is worth one sentence, because it is counter-intuitive: Entra
permissions and Exchange role assignments are a **union**, not nested. An app holding tenant-wide `Mail.Send` *and*
a mailbox-scoped assignment can send as anybody — the scope narrows nothing while looking exactly as though it
does. **The only way to have one mailbox is for the broad permission never to exist.**

**Not requested either, and this is the point:** no `Application.ReadWrite.OwnedBy`, no `Application.ReadWrite.All`,
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
| **And, only if you say yes to self-update:** a third identity, a custom role definition, one more queue and a fifth job | see *Letting the deployment update itself* below. Nothing here is created unless you ask for it |
| Azure SQL — server and one database | the register, the journal, everything recorded. **Entra-only authentication, no password anywhere** |
| Storage account | the work queues, and the key ring that keeps people signed in across a restart. Shared-key access disabled, so there is no connection string to leak |
| Container registry | created by the template. It holds the product image when the image has to be copied into your subscription (§3); with an anonymously pullable reference nothing is stored in it |
| Log Analytics workspace | the container logs |

**Two of those cost money whether anybody uses the product or not**: the database (Standard S0 by default) and the
one web replica. Everything else is small or usage-based. Price them for your region in the Azure pricing
calculator before you commit — this document deliberately quotes no figure it cannot keep current.

**One thing is created outside your subscription**: a **security group in your directory**, whose members are the
only people who can connect to this deployment's database directly. The product's own worker identity is a member,
which is what lets the deployment create its database users without anybody reaching the server from outside Azure.
The installer adds nobody else.

**No secret exists in a deployed stamp.** The database, the queues, the key ring and Microsoft Graph are all
reached with managed identity, and the sign-in registration authenticates with a federated credential rather than
a client secret. The application refuses to start if a connection string ever arrives with a password in it.

### Letting the deployment update itself — a question the installer asks

The product tells you when a newer release exists, whichever way you answer this. What the answer decides is
**whether it can apply one for you**, or whether applying one means running the installer again.

**Saying no costs you nothing you have not already accepted**: an update is `./install.ps1 -Only upgrade`, run by
somebody with Contributor on the resource group, and the product prints the exact command on the screen that told
you about the release. That is the whole of the difference.

**Saying yes adds one managed identity, and it is worth understanding what that identity can do.** It holds a
**custom role definition with four actions** — read and write on this resource group's container apps and jobs —
and nothing else: no access to your directory, no ability to change the database schema, no other queue. Azure has
no narrower permission for *may set an image*, so that right is genuinely the right to change what containers in
this resource group run. Nothing that faces the internet holds it: the web app cannot, the worker that reads your
directory cannot, and the one job that can holds no directory access of its own.

What it will and will not do, all of it enforced in the deployment rather than promised here:

- it applies **only a version we have published** — the request carries a version number and the image reference is
  built inside the deployment, so nothing that could be typed or injected chooses what runs;
- it **refuses a major release** and hands you the command instead, because a major version in this product means
  the upgrade needs a step outside it — a template deployment, a consent, something in Exchange;
- it **watches the new version come up** on the deployment's own health endpoint and **puts the previous image back
  automatically** if it does not;
- it **never touches your database**. A release that migrates still asks for the migration on the product's own
  upgrade screen, because when your schema moves stays your decision;
- every roll is recorded with who asked, from which version to which, and what came of it.

**A later re-run will not take it away by accident.** An ordinary `./install.ps1` on a deployment that already
updates itself reads that off the deployment and keeps it — including from a fresh download that has never seen your
answers file. It says so when it does.

**You can change your mind in one direction easily.** `./install.ps1 -SelfUpdate` adds it to a deployment that said
no — the whole command, not `-Only deploy`: the updater needs a database user, and that is created by the migrate job
in the bootstrap phase. A run that skips it says so and tells you what to run. Taking it away is a deliberate removal rather than re-running with the answer flipped: the template deploys
incrementally, so what it no longer declares keeps existing. Re-running with `-SelfUpdate:$false` stops the product
offering the control and leaves the identity, the role, the queue and the job in place; the commands to remove those
are in §8.

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
| `group` | Entra | enough to **create a security group**. Many tenants let any user; where that is turned off it needs **Groups Administrator** |
| `signin` | Entra | the same as `registration` |
| `bootstrap` | Azure | enough to add a group member and start a container app job. **No database access at all** — the work happens inside Azure, done by the product's own migrate job |
| `permissions` | Entra | **Privileged Role Administrator** or Global Administrator. Granting an application permission is consent, which is a larger right than deploying |
| `mail` | Exchange Online | **Exchange Administrator**, for the mailbox and the role assignment |
| `verify` | — | none |

One person holding all of them can run the whole thing in one command. If that person does not exist in your
organisation — and in most organisations they should not — run the phases separately; every phase is re-runnable
and reads the live state rather than a file left behind by the last one.

### On the machine you run it from

**Two things, and one of them you are already in.**

- **The Azure CLI** — [aka.ms/installazurecli](https://aka.ms/installazurecli) — signed in with `az login`.
- **A PowerShell**: Windows PowerShell 5.1, which every Windows administrator already has, or PowerShell 7.

**No PowerShell modules, and no connection to your database.** The installer talks to Azure and to Entra through
`az`, and the two database users the product needs are created **from inside Azure** by its own migrate job. Nothing
here needs TCP 1433 open from your machine — which is the port most corporate networks block, and which used to be
where this install stopped for reasons that had nothing to do with the product.

**There is no exception any more.** The mail phase used to need the `ExchangeOnlineManagement` module, and on a
Windows profile whose name carries a diacritic — `C:\Users\Ondřej…` — that module cannot sign in at all: the
authentication library it uses fails to load one of its own files from a path with an accented character, and no
option turns that off. The install now reaches Exchange the same way that module does underneath, through `az`.
Nothing to install, and nothing that fails on how your account is named.

### What to have decided

| Answer | Notes |
|---|---|
| Subscription and region | one region holds the whole stamp |
| The container image | the reference EnterCloud gives you, **with its version tag**. See §3 |
| The first Administrator | a person who signs in, applies the schema and configures the deployment. It grants them nothing in Entra |
| The database administrator group | a name — the installer creates the group. The product's worker identity is added to it, which is what lets the deployment create its own database users; **membership is also how a person reaches that database directly**, and nobody but the deployment is added |
| The notice mailbox | a shared mailbox that already exists. **Its display name is what every recipient sees** — see §5 |
| The address people will open | only if it is **not** the hostname Azure generates. Entra refuses a redirect URI it was never told about, so a custom domain has to be registered — the generated one always is, and nothing means "that one" |
| **Whether the deployment may update itself** | the installer asks, and the default is no. It is a decision about one managed identity's rights rather than about a feature — §1 says exactly what that identity can do, and either answer leaves you a working deployment that tells you when a release exists |

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

    ghcr.io/entercloud-cz/entra-app-manager:v3.6.0

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
# 1. the Entra administrator — the group that will administer the database, and the registration people sign in
#    against. The registration comes before the deployment because the web app refuses to start without its
#    client id
.\install.ps1 -Only group,registration -DatabaseAdminGroup sql-eam-prod-admins -AdministratorUpn anna@contoso.com

# 2. the subscription owner — the deployment itself
.\install.ps1 -Only preflight,deploy -SubscriptionId <id> -Location swedencentral

# 3. the Entra administrator again — the redirect URIs and the federated credential need the deployed hostname and
#    the deployed identity, which is why this is not part of step 1. -PublicFqdn is the address people will open;
#    leave it out if that is the hostname Azure generated
.\install.ps1 -Only signin -PublicFqdn eam.contoso.com

# 4. anybody with rights on the deployment — no database access, no port. This adds the worker identity to the
#    group and starts the product's own migrate job, which creates the database user and applies the schema
.\install.ps1 -Only bootstrap

# 5. the Privileged Role Administrator — consent
.\install.ps1 -Only permissions

# 6. the Exchange administrator — in the same sitting as step 5, see §5
.\install.ps1 -Only mail -NoticeMailbox appmanager@contoso.com

# 7. anybody
.\install.ps1 -Only verify
```

Steps 1–4 can be one command when the same person holds those rights; that is what plain `.\install.ps1` does.

`-NonInteractive` refuses instead of prompting, for a pipeline. `-Plain` swaps the icons for ASCII on a console
that cannot draw them.

### It remembers what you told it

After a run — including a run that stopped with a refusal — the script writes
`install-answers.<workload>-<environment>.json` **beside itself**, and offers those values back as the defaults next
time. **An upgrade is this script again**, so without that memory the person upgrading retypes a subscription id, a
region, a group name and a mailbox, and one typo deploys something *beside* the deployment they meant to change —
silently, with every command succeeding.

- **Names, ids and choices only.** The file is built from a closed list, so it cannot hold anything else, and there is no secret in this install to hold — no client secret exists anywhere in the deployment. Open it and see.
- **A value you pass on the command line always wins.** It is a memory, not a configuration file.
- **It is per deployment, twice over:** the file is named for the workload and environment, and it records them too — so a file that was copied or renamed is refused rather than offered. A remembered value from the wrong stamp is worse than no memory at all.
- **Delete it** to be asked everything again. A missing or unreadable file changes nothing about how the script behaves.
- Keep it out of source control if you keep the bundle in any.

### What each phase does

| Phase | What it does, and what it verifies afterwards |
|---|---|
| `preflight` | confirms the CLI and the sign-in, which tenant this will watch, and registers the resource providers the deployment needs |
| `registration` | creates the app registration people sign in against — called **App Manager for Microsoft Entra**, with the environment in brackets on anything but `prod` — declares the four app roles, creates its enterprise application, and assigns **Administrator** to the person you name. A registration an earlier version created under its old name is **found and renamed**, never duplicated. **No client secret is created, here or ever** |
| `deploy` | deploys the whole stamp — one pass, or three steps if the image has to be copied in. Refuses a placeholder image |
| `signin` | reads the hostname off the running app and registers it, asks for the address **people will actually open** and registers that beside it, and creates the federated credential that lets the web app authenticate **without a secret** |
| `group` | creates the security group that will administer the database, or finds it. Adds nobody but the deployment, and prints the one command that adds a person |
| `bootstrap` | adds the worker identity to that group, then starts the product's own **migrate job** — which creates the web tier's database user by SID and applies the schema. Reads the job's log back, because that is where the evidence is. **Touches no database and needs no port** |
| `permissions` | grants the three Graph permissions, then lists anything held **beyond** them |
| `mail` | checks the mailbox, creates the Exchange scope and the `Application Mail.Send` assignment, and asks Exchange whether the mailbox is inside it |
| `verify` | reads `/healthz` and `/api/schema` off the running deployment and prints what is left for a human |

### Why the registration comes before the deployment

The web container **refuses to start** without the client id of the registration people sign in against — a
deployment serving pages nobody can sign in to is worse than one that stops and says why. Creating the
registration first is what makes the deployment a single pass. The redirect URI has to wait until afterwards,
because the hostname is generated by the platform and is read off the running app rather than assembled from a
pattern.

**And if people will open a different address, say so when `signin` asks.** Entra matches a redirect URI exactly and
refuses one it was never told about — naming the URI, not the omission — so a deployment behind a custom domain
cannot be signed in to until that address is registered. Both are registered, sign-in and sign-out paths for each,
and the generated hostname is never replaced. Registering an address does not create it: the custom domain still has
to resolve to the deployment. Entra's **front-channel logout URL** is a single value and points at the address you
gave; that only matters when something else signs a person out, and sign-out from the product works on both.

### Until the permissions are consented

The projection fails with an authorisation error and the product's job health surface says so. That is the bound
working, not a fault. Nothing else about the deployment is affected — you can sign in, and the register is simply
empty.

---

## 5. Mail — one mailbox, and the assignment that makes it one

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

### The authority to send — and this is the whole control

**Nothing in this deployment holds tenant-wide `Mail.Send`.** So what this step creates is not a fence around a
broad right; it *is* the right, and it names one mailbox. Three objects, all created by the script:

| What | Why it exists |
|---|---|
| a pointer to the worker identity, in Exchange | a role assignment cannot name a principal Exchange has never heard of. Deleting the identity in Entra removes this automatically, so it cannot outlive what it points at |
| a **management scope** naming the mailbox by address | `PrimarySmtpAddress -eq 'appmanager@contoso.com'` — one mailbox, no group |
| an assignment of the role **`Application Mail.Send`**, scoped to it | the authority itself |

There is **no mail-enabled security group** in this any more. The old mechanism needed one because an application
access policy's scope had to be a group; a management scope is a filter, so it can name the mailbox directly. One
fewer object, no waiting for a group to replicate, and no rule about nested members falling outside the scope.

The script then **asks Exchange whether it bites**: the mailbox must come back *in scope* for that role, or the
phase stops. Verified rather than asserted, because an assignment that exists and does not work looks exactly like
one that does.

It used to ask you for a second address as well, and check that the scope *excluded* it. That is gone. The failure
it caught belonged to the old mechanism — a tenant-wide right narrowed afterwards, where a fence around nothing was
indistinguishable from a fence that held. There is no tenant-wide right now, the phase refuses to build a scope
beside one, and the filter is a single address the script writes itself.

> ⚠ **Upgrading from a release before 3.0.0?** Your deployment holds tenant-wide `Mail.Send` from the old
> mechanism, and it has to be **removed** — until it is, you have both authorities, which means no scoping at all.
> The `mail` phase refuses to go on and prints the one command that removes it. Notices cannot be sent in between,
> which is the safe direction: nothing delivered rather than something delivered as the wrong mailbox.

**What is not instant.** Exchange caches an application's permissions for between 30 minutes and 2 hours. The
check the script runs bypasses that cache, so its answer is trustworthy immediately; a test mail sent from inside
the product right afterwards may not be. If the product's test mail fails within the hour, wait and try again
before treating it as broken.

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

- **A new group membership is not instant, and starting the job again is the whole fix.** The bootstrap adds the
  product's worker identity to the database administrator group and then starts the migrate job, which authenticates
  as that identity — and a membership that has not reached the token yet fails as *login failed for user*, which reads
  like a wrong password in a product that has none. The job now says exactly that, in one sentence, beginning
  `LOGIN REFUSED:`, and the installer knows the difference between that ending and a real fault: it asks the first one
  again and stops on the second.

  **Nothing has to age out and nothing needs restarting by hand.** Measured on two stamps: an execution that was
  refused at 14:30:03 succeeded at 14:32:31 with nothing changed in between — a new execution asks for a new token,
  and the new token carries the group as soon as Entra has replicated it. Under three minutes both times. If it is
  still refused well past that, the membership or the server's administrator is what to check, and the message names
  both.
- **Nobody can connect to that database except members of the group.** That is the design (`0064`) and not an
  oversight: the installer adds only the deployment itself, so if you need to read the database by hand, add
  yourself — `az ad group member add --group <the group> --member-id <your object id>`. Nothing else has to change.
- **A resource provider that is not registered fails the deployment** with `MissingSubscriptionRegistration`, which
  names the namespace and not the remedy. The preflight phase registers them; registration itself can take a few
  minutes to complete in the background.
- **An Exchange permission change is cached for 30 minutes to 2 hours.** The script's own check bypasses that
  cache, so trust it; a test mail from inside the product within the hour may fail on the cache rather than on the
  configuration. Waiting is the fix, not re-running anything.
- ~~**The Exchange policy can take up to an hour**~~ — the old mechanism's trap, and a fresh policy could answer `Granted` for an address
  outside it while Exchange catches up. Re-run the mail phase until the outside address reads `Denied`. Do not
  treat the permission as narrowed before then.
- ~~**A new mail-enabled group is not immediately usable as a policy scope.**~~ There is no group in this any more.
- **The database's tier and its maximum size are not independent.** Azure SQL Standard accepts only a discrete list
  of sizes for each service objective, and nothing in either value says so — a legal-looking pair is refused with
  `InvalidMaxSizeTierCombination` at deploy time. The defaults (S0, 30 GB) are a pair that works. If you change
  `-SqlSkuName`, read the list for your region first:
  ```bash
  az sql db list-editions -l <region> --edition Standard --service-objective <objective> --show-details max-size
  ```
- **The Azure CLI can fail on your machine, and the installer now says so rather than blaming your tenant.**
  Measured on a customer's Windows machine: the same read failed and then passed, and each run of the installer got
  one step further until the install completed. The CLI has two known faults with this shape — it can crash while
  reporting an HTTP error, losing the error, and it can lose its own token-cache file to itself between commands.
  A **read** is therefore attempted three times before the installer gives up, and it says which attempt it is on;
  a **write** is never repeated. If it still stops, the message says whether the fault is this machine or your
  tenant. Where it is the machine, repair the CLI — `az version`, then reinstall it — and run the installer again:
  it resumes, and nothing you have already answered has to be typed twice.
- **Being behind is invisible unless somebody looks.** `https://<host>/healthz` answers with the build and commit
  that is actually serving. Nothing in the product nags you about a newer release.
- **Deleting the resource group deletes the record.** The register can be rebuilt from Entra in one refresh; the
  journal, the recorded owners and everything a person typed cannot — they exist only here. Export the journal from
  the Audit surface first (CSV or JSON) if you need to keep it.

---

## 8. Upgrading, rolling back, removing

**Upgrade — one command, and it is not the install:**

```powershell
./install.ps1 -Only upgrade -Image <new reference>
```

Four steps and no questions. It points the product's own migrate job at the new image and **runs it** — the plan,
each migration's declared cost and the backup line are printed before anything is touched — then rolls the web app
and all three background jobs, then asks the running deployment what it is serving.

- **The schema moves before the application, and that order is the point.** The web tier refuses to serve a schema
  behind its build, so the other way round means a deployment that stops serving until the migration lands.
- **`-WhatIf` walks the whole thing and changes nothing.** Worth doing first on a deployment people are using.
- **It refuses to be an install.** Pointed at a deployment that does not exist, it says so and stops.
- **It asks nothing you have already answered.** The region, the group, the registration and the mailbox are
  already true; an upgrade reads them rather than defaulting them, so it cannot move your stamp to another region or
  hand your database's administration to whoever is running the upgrade.
- **Every workload that runs the image is rolled together.** One left behind runs the previous release's code
  against the new schema. That includes the updater job, on a deployment that has one — it is asked about rather
  than assumed, so this command works the same on a deployment that never had self-update.

**Or, on a deployment that updates itself, press the button** (§1, *Letting the deployment update itself*). It does
the same roll and adds the two things a person at a keyboard would otherwise do: it watches the new version come up
and puts the previous image back if it does not. Two differences are worth knowing before you rely on it:

- **It does not migrate**, so a release that changes the database leaves the upgrade screen waiting — which is the
  same screen the command above would have left you if you had rolled the image by hand.
- **It refuses a major release.** A major version means the upgrade needs a step outside the product, so the panel
  hands you the command instead. The command is the path for those, always.
- **There are no down migrations.** Take a backup you would be willing to restore. The command says so before it
  applies anything.

**Roll back** — the same command with the previous reference. It is bounded: the schema records the oldest build
that can serve it, and a build below that floor refuses to start rather than reading columns it does not
understand. That refusal names the migration it is missing. Below the floor, the way back is forward.

**And if you roll an image some other way** — your own pipeline, or `az containerapp update` by hand — nothing is
lost: the deployment serves its **upgrade screen** and refuses everything else until an Administrator applies the
migration there. Nothing is ever applied because a container restarted.

**Removing self-update but keeping the deployment** — the template will not do it for you, because an incremental
deployment leaves what it no longer declares running. Re-run the installer with `-SelfUpdate:$false` first, so the
product stops offering the control, and then remove the four resources by hand:

```powershell
$rg   = 'rg-eam-prod'      # your resource group
$name = 'eam'; $env = 'prod'

az containerapp job delete --name "caj-$name-updater-$env" --resource-group $rg --yes
az identity delete          --name "id-$name-updater-$env" --resource-group $rg
az role definition delete   --name "App Manager image roll ($name-$env)"
az storage queue delete     --name update        --account-name <the storage account> --auth-mode login
az storage queue delete     --name update-poison --account-name <the storage account> --auth-mode login
```

Deleting the identity removes its role assignments with it. The database user it was given stays until somebody drops
it, and it can do nothing without an identity to authenticate as. Run `./install.ps1 -Only permissions` afterwards:
it reports the absence as normal, which is how you know the deployment is back to applying updates by command.

**Remove** — delete the resource group, then the four things that live outside it: the app registration
(**App Manager for Microsoft Entra**, with the environment in brackets on anything but production), the Graph permissions granted to the worker identity, the **database
administrator group**, and — in Exchange — the management scope, the role assignment and the service principal
pointer. Deleting the resource group deletes none of them. Exchange removes the pointer and its assignments by
itself when the managed identity goes, so what is genuinely left behind is the management scope and the group, and both refer to an identity
that no longer exists.

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
- **The installer never connects to your database**, and nothing outside Azure needs to: the users the product needs
  are created by its own migrate job, whose identity is a member of the group that administers the server.
- The **journal is append-only**, enforced where the writes happen rather than by convention: entries cannot be
  deleted and cannot be edited, and each act is recorded **before** it is attempted so a failure leaves a trace
  rather than a silence.
- It **cannot change what it runs** unless you asked for that at install time, and where you did, the right belongs
  to one identity that holds nothing else — it may set an image in this resource group, and it may not reach your
  directory, your schema, or anything else. It applies only versions we have published, refuses a release whose
  number says a person is needed, and puts the previous image back if the new one does not come up.
