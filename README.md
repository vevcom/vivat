# vivat

Infrastructure for Sct. Omega Broderskab: what runs on the k3s cluster, deployed by
[Argo CD](https://argo-cd.readthedocs.io) from git. A change to the cluster is a pull request
to this repo, and Argo CD applies what `main` says.

```
clusters/
├── prod/           # The production cluster: root app, cluster resources and one overlay per service
│   ├── omegaweb/   # omega.ntnu.no and its sites: domains, certificate, backups, Phase secrets, S3 store
│   └── mail/       # Mail for omega.ntnu.no: Stalwart and the webmail, the relay, the certificate
└── local/          # A k3d cluster on a workstation, for trying changes first
platform/           # Operators the services depend on: cert-manager, CloudNativePG, Phase
cluster/            # Cluster-wide resources such as the Let's Encrypt issuer
```

## How it fits together

A service's repo holds the generic shape of one installation, a `deploy/base` kustomization that
names no domain, no certificate and no release. Everything about *our* installation is the
overlay here: it pulls that base at a release tag and fills in the rest. Each cluster has a root
application that syncs its directory in `clusters/`, itself included, and one `Application` per
service points at the service's overlay.

| Service | Repo | Overlay | Namespace |
|---|---|---|---|
| omegaweb | [omegaweb-next](https://github.com/vevcom/omegaweb-next) | `clusters/prod/omegaweb` | `omegaweb-next` |
| mail | [mail-infrastructure](https://github.com/vevcom/mail-infrastructure) | `clusters/prod/mail` | `mail` |

Applications sync in waves: the operators and cluster resources in wave -1, the services in
wave 0. Within a service, its own manifests decide the order, for example the database before
the migrations before the app.

## Releasing a service

Publishing a GitHub release of a service repo opens a pull request here that points its overlay
at the release: the ref the base is pulled at, and the image tags where the service builds images.
Merging that pull request is the deploy, and a rollback is a revert of it. Nothing merged to main
of a service repo reaches prod until a release names it, so a change to a service's manifests is
tried on `clusters/local` first. The pull requests are opened by a GitHub App on the vevcom org,
installed on this repo; each service repo holds its id as the variable `DEPLOY_APP_ID` and its
private key as the secret `DEPLOY_APP_PRIVATE_KEY`.

## Adding a service

1. Give the service a `deploy/base` in its repo: the generic shape, with placeholder hosts and
   nothing about where it runs.
2. Add an overlay under `clusters/prod/` that pulls that base at a release tag and sets the
   namespace, the hosts, the certificates and the release, and an `Application` for it next to
   `omegaweb.yaml`, listed in `clusters/prod/kustomization.yaml`.
3. Secrets come from the service's own Phase app through a `PhaseSecret` in the overlay. The
   service token the operator logs in with is created by hand once per cluster and namespace.
4. A private repo needs credentials Argo CD can read it with. A GitHub App on the vevcom org,
   stored as a `repo-creds` secret for `https://github.com/vevcom`, covers every repo at once.

Third-party software, such as Grafana, is installed from its upstream Helm chart like the
operators in `platform/`, with values kept in this repo.

## Setting up a cluster

Install k3s, then Argo CD, then apply the cluster's directory once:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --version 10.10.5 --namespace argocd --create-namespace --wait
kubectl apply -k clusters/prod
```

Argo CD then installs the operators and every service. The secrets that cannot come from git are
made by hand, once per cluster. Read every token with `read -s` so it stays out of the shell history.

```bash
kubectl create namespace omegaweb-next
kubectl create namespace mail
read -s PHASE_TOKEN
kubectl -n omegaweb-next create secret generic phase-service-token --from-literal=token="$PHASE_TOKEN"
kubectl -n mail create secret generic phase-service-token --from-literal=token="$PHASE_TOKEN"
kubectl -n omegaweb-next create secret docker-registry ghcr-pull --docker-server=ghcr.io \
    --docker-username=<user> --docker-password=<read:packages token>
```

## Trying it locally

omegaweb-next's `deploy/k3d/up.sh --argocd` sets up a k3d cluster that follows
`clusters/local`. `clusters/local/omegaweb.yaml` names the branch of omegaweb-next it deploys.
Mail has no local overlay: in development it runs as a compose stack next to omegaweb-next, see
mail-infrastructure's `dev/`.

## omegaweb

### Secrets

The app's environment lives in Phase. The Phase operator reads it into the `omegaweb-next-env`
secret, as set up in `clusters/prod/omegaweb/phase-secret.yaml`, and keeps it in sync. The
operator does not restart the app, so after changing a secret in Phase run
`kubectl -n omegaweb-next rollout restart deployment`.

### Domains

The sites' domains are in `clusters/prod/omegaweb/sites.json`, which every pod mounts and none
start without. A changed file rolls the pods. Every host in it, aliases included, is also in
`ingress-patch.yaml`, which routes it and lists it on the certificate; the two files must agree.
Each host has an A record pointing at the node, like the others.

`login.omega.ntnu.no` is the auth domain, the entry marked `"auth": true`: everyone signs in
there, and every site's domain gets its own session from it through the app's OpenID Connect
provider, see omegaweb-next's README. It belongs to no site and serves nothing but the sign in,
sign up and password pages. So Feide is registered once, as
`https://login.omega.ntnu.no/api/auth/callback/feide`, and a new site domain needs nothing at
Feide. Until the entry is live, with its A record, omega.ntnu.no signs users in, which the app
does on its own when the file marks no auth domain.

### The store

Uploaded images, ombul pdfs and the release countdown's files go to the S3 bucket in
`kustomization.yaml`, on NTNU's object storage at `https://s3.trd1.stack.it.ntnu.no`, with keys
from Phase: EC2 credentials from `openstack ec2 credentials create`, as `STORE_S3_ACCESS_KEY_ID`
and `STORE_S3_SECRET_ACCESS_KEY`. The database backups in `database-patch.yaml` use the same keys,
since EC2 keys reach every bucket in the project. Turn on versioning for the bucket, so a deleted
file can be brought back after a database restore:

```bash
aws s3api put-bucket-versioning --bucket <bucket> --versioning-configuration Status=Enabled \
    --endpoint-url https://s3.trd1.stack.it.ntnu.no
```

NTNU does not back the bucket up, so copy it somewhere else nightly with `rclone copy`.
Use `copy`, not `sync`, so deletes never reach the copy.

### Releases

`release.yml` in omegaweb-next builds the images with the repository secret
`NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` and the variable `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`, and
opens the pull request here with the deploy app's `DEPLOY_APP_ID` and `DEPLOY_APP_PRIVATE_KEY`.
Both GHCR packages stay private, since the server actions key is inside the images; the pods pull
with the `ghcr-pull` secret made by hand above.

## mail

[Stalwart](https://stalw.art) receives the mail for omega.ntnu.no, forwards each alias to the
members omegaweb-next lists behind it, sends what the app sends, and keeps a mailbox per alias,
which the [Bulwark](https://bulwarkmail.org) webmail at `webmail.omega.ntnu.no` shows to those who
receive the alias's mail, signed in with their omegaweb-next account. What that is in Kubernetes
is mail-infrastructure's `deploy/base`; `clusters/prod/mail` is our installation of it.
`site.ndjson` there is the plan the provision job applies after the base one: the domain, the
hostname `mail.omega.ntnu.no`, the certificate from the files cert-manager writes, NTNU's gateway
as the relay for outbound mail, the webmail's host as the domain of the accounts, and the
directory that signs users in through omegaweb-next, see
[Signing in and mailboxes](#signing-in-and-mailboxes).

### Secrets

The mail app in Phase holds one secret, `STALWART_ADMIN_PASSWORD`: the password of `admin`, which
Stalwart's admin ui at `https://mail.omega.ntnu.no/admin`, the provision job and omegaweb-next's
sync log in with. The operator writes it to the `mail-env` secret as set up in `phase-secret.yaml`.

### The application

omegaweb-next keeps the server's mailing lists in step with its aliases and sends its mail
through the server, so its Phase app has these, next to its other variables:

```
EMAIL_SERVER=stalwart.mail.svc.cluster.local
EMAIL_PORT=2525                           # the listener for the application, which asks for no login
EMAIL_SECURE=false
EMAIL_ADMIN_URL=http://stalwart.mail.svc.cluster.local:8080
EMAIL_ADMIN_USER=admin
EMAIL_ADMIN_PASSWORD=<STALWART_ADMIN_PASSWORD>
```

It has no account on the server and so no `EMAIL_USER` or `EMAIL_PASSWORD`: the directory below
could not sign one in. mail-infrastructure's base plan gives Stalwart a listener on port 2525 that
relays as any address without a login, on the service inside the cluster alone, and
`clusters/prod/mail/application-listener.yaml` lets only omegaweb-next's namespace reach it. That
policy is the whole of the port's security, and it holds because k3s enforces NetworkPolicy; a
cluster set up with `--disable-network-policy` would leave the port open to every pod.

Stalwart opens a listener when it starts. After the first sync of the release that brings the
listener, restart it once, or nothing answers on 2525:

```bash
kubectl -n mail rollout restart statefulset/stalwart
```

`EMAIL_WEBMAIL_URL` is no secret and is set in `clusters/prod/omegaweb/kustomization.yaml`
instead, see below.

### Signing in and mailboxes

Stalwart authenticates against omegaweb-next, which is an OpenID Connect provider on
`login.omega.ntnu.no`, see [Domains](#domains): `site.ndjson` defines the directory and makes it
the one authentication uses, and `bulwark-config` tells the webmail the same issuer. So the webmail
signs you in with your omegaweb-next account, and shows you the mailboxes of the aliases you
receive mail on. The token you sign in with only says who you are: omegaweb-next's sync makes your
account a member of those mailboxes, every minute, so `site.ndjson` tells Stalwart to read no
groups from the token (`claimGroups` null). Owning an alias in omegaweb-next is administering it
and gives no mailbox.

The webmail's host, `webmail.omega.ntnu.no`, is three things at once, and the places must agree:
the domain the signed-in accounts are on (`usernameDomain` and the accounts' domain in
`site.ndjson`), the client omegaweb-next issues the webmail's tokens to (`OAUTH_CLIENT_ID` and
`acceptedAudiences`), and omegaweb-next's `EMAIL_WEBMAIL_URL` in `clusters/prod/omegaweb`, from
which it derives all of that. The mailbox of `styret@omega.ntnu.no` is a group at that same
address, and what forwards its mail is a list on `forward.webmail.omega.ntnu.no`, a domain
omegaweb-next makes on the server and which has no DNS. `site.ndjson` has Stalwart refuse mail
for that domain from anyone not signed in, so only the copies the mailboxes send on reach those
lists. mail-infrastructure's base plan has it refuse a mail from someone signed in whose From
header is not the address it is sent as, so a member of a mailbox answers as that mailbox and as
no one else.

Once the directory is active, Stalwart sends every password login to it, and omegaweb-next has no
passwords to give: a mail client signs in with an app password made in the webmail, and the
recovery admin with its own. omegaweb-next sends without signing in, see
[The application](#the-application).

### DNS

Mail for omega.ntnu.no is received by NTNU's gateways (`mx.ntnu.no`), which filter spam and check
SPF, then hand it to the mail host by name: `sanctus.omega.ntnu.no`, so wherever its A record
points. They deliver from `129.241.56.0/23`, which is what the security group allows port 25 from.
Outbound mail leaves through `mailgw.ntnu.no`, which relays for NTNU's network by source address,
so the domain's SPF stays as it is. The records for the move:

```
mail.omega.ntnu.no.       A    <node ip>     # the server's name, on the certificate
webmail.omega.ntnu.no.    A    <node ip>
sanctus.omega.ntnu.no.    A    <node ip>     # moves the inbound mail here, last
```

Stalwart signs the mail it sends with DKIM and makes the keys itself. Publish the two
`_domainkey` TXT records it prints in the domain's zone file, which `stalwart-cli get Domain <id>`
shows under `dnsZoneFile`, or the admin ui under the domain's DNS records. `_dmarc.omega.ntnu.no`
already exists at `p=none`.

### Moving the mail off Dokploy

1. **Fill in the TODOs**: the Phase app id in `phase-secret.yaml`, and the first release of
   mail-infrastructure in `kustomization.yaml`, which its release workflow sets through a pull
   request once mail-infrastructure's deploy is merged and released.
2. **Create the hand-made secret**, the Phase service token in the `mail` namespace, as above, and
   put `STALWART_ADMIN_PASSWORD` in Phase.
3. **Add the A records** for `mail.` and `webmail.`, so the certificates get issued, and open port
   25 from the gateways in the security group.
4. **Point omegaweb-next at it**, with the variables above, and check in Stalwart's admin ui that
   its account and a list per alias appear within a minute.
5. **Try it**: send a mail to an alias straight at the node, `swaks --server <node ip> --to
   <alias>@omega.ntnu.no`, from a machine the security group allows, and see it forwarded. Sign in
   to the webmail with your own account and find it in the alias's mailbox.
6. **Switch**: move the A record of `sanctus.omega.ntnu.no` to the node, then stop the Postfix
   stack on Dokploy. The gateways remember a recipient they could not deliver to for a few hours,
   so do it when no alias is being edited.
7. **Delete `postfix/`** in mail-infrastructure.
