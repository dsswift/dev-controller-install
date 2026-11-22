# Installing the dev controller

The controller is dev's part in a cluster. It keeps the record of every
environment, makes each one real, builds its images, deploys into it, and
answers the `dev` program. Nobody but the controller holds a cluster
credential.

`deploy/base` is a Kustomize component. An estate installs it through a
kustomization of its own, beside the settings object it makes.

## What you need

- Kubernetes 1.30 or later. The install uses `ValidatingAdmissionPolicy`.
- A Kustomize that renders a component after the generators of the
  kustomization that lists it. The install is tested with Kustomize 5.8 and
  with the one in `kubectl` 1.37.
- An OIDC issuer for people, with a public client for the `dev` program. The
  client needs the redirect address `http://localhost/callback`, on any port,
  and no client secret. The program listens on a free port of the loopback
  address and takes the sign-in back on the path `/callback`. An issuer that
  takes only one exact address gets `redirectUri` in the settings, such as
  `http://localhost:8250/callback`, and that same address registered.
- A way for people and pipelines to reach the controller over HTTPS: an
  Ingress or a gateway of your own in front of the `dev-controller` Service.
  The install makes none, since the host and the certificate are yours. It
  must take a large request body, hold a request open for as long as a
  build takes, and let a connection become a WebSocket: see
  [The gateway in front](#the-gateway-in-front).
- An ingress controller, with one wildcard DNS record and one wildcard
  certificate for the domain your environments' hosts go under.
- A registry, or a path in one, that holds only images built for
  environments. Only the controller's builder should be able to push there.
- A network plugin that enforces network policies. Each environment is
  walled in by one, and the builder is kept to the controller by another.

## What the install holds

| File | What |
|---|---|
| `namespace.yaml` | The `dev-system` namespace. |
| `crds/` | The `Environment` record, in group `dev.dsswift.github.io`. |
| `rbac.yaml` | The controller's service account and its rights: a small cluster-wide role, a role in its own namespace, and the environment role it binds to itself in each namespace it makes. |
| `deployment.yaml`, `service.yaml` | One pod, not root, with a read-only file system, and its Service. |
| `policies.yaml` | Three admission policies: one keeps environment images inside environments and by digest, one keeps environment pods on the nodes you picked, one keeps the controller to the namespaces it made. |
| `settings.yaml` | The documented settings file. Every setting is explained in it. |

The controller also runs one thing the files do not hold: its builder,
rootless BuildKit, as `dev-buildkit` in `dev-system`. It applies it at start,
places it with your `placement` setting, and keeps it as the settings say.
Rootless BuildKit needs a pod with no seccomp or AppArmor profile, so
`dev-system` cannot be held to the baseline pod security standard.

## Install

1. Copy `deploy/base/settings.yaml` beside a new `kustomization.yaml` and
   edit it. It says what each setting does.

2. Write the kustomization:

   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   components:
     # The install at the release you run. Its image is that release's.
     - https://github.com/dsswift/dev-controller-install//deploy/base?ref=v4.12.0 # x-release-please-version
   configMapGenerator:
     - name: dev-controller-settings
       namespace: dev-system
       files:
         - settings.yaml
       literals:
         # The registry, or the path in one, that holds only images built
         # for environments. The install renders it into the admission
         # policy, and the controller's builder pushes to it. Leave the
         # value empty if you have none: nothing can then be built.
         - environmentRegistry=registry.example.com/environments/
   ```

   The install is a component, not a base, so it is rendered together with
   your settings object. A kustomization with no `environmentRegistry` does
   not render.

   The component is fetched by its release tag from a public repository,
   with no token: any GitOps tool that renders Kustomize fetches it the way
   `kubectl apply -k` does. Each release of dev publishes `deploy/` there,
   tagged as the release, once the image it names is published and signed.
   Nothing in it is an estate's.

3. Put the secrets your settings name into place:

   - Each secret under `secrets.names`, in the namespace you named as
     `secrets.sourceNamespace`. They are copied into every environment.
   - `registry.pushSecret`, a docker config secret that may push to the
     environment registry, in `dev-system`.

   The install lets the controller read secrets in `dev-system`. For
   another namespace, give the `dev-controller` service account `get` on
   those secrets there.

4. Apply it, and wait for the controller and its builder:

   ```bash
   kubectl apply -k .
   kubectl rollout status deployment/dev-controller -n dev-system
   kubectl rollout status deployment/dev-buildkit -n dev-system
   ```

   The first apply can run ahead of the `Environment` record it has just
   added. If it does, apply once more.

5. Put your gateway or Ingress in front of the `dev-controller` Service, port
   80, with a certificate.

6. Check it. This needs no sign-in:

   ```bash
   curl https://dev.example.com/.well-known/dev-controller
   ```

7. In a project, name the controller in `dev.yaml`, then sign in:

   ```yaml
   controller:
     url: https://dev.example.com
     project: shop
   ```

   ```bash
   dev login
   dev deploy
   ```

## Moving from 4.10

The install keeps its shape, and a settings file written for 4.10 works as
it is. What an estate does once:

1. **Fetch the install, and drop your copy.** An estate that could not fetch
   the install from dev's own repository kept a copy of each release. The
   install is now published where anyone may fetch it by tag. Replace the
   copy's path under `components:` with the published one, delete the copy
   and the script that took it, and apply:

   ```yaml
   components:
     - https://github.com/dsswift/dev-controller-install//deploy/base?ref=v4.11.0
   ```

   The objects it renders are the copy's, so the apply changes only what
   4.11 changed. A tool that fetches a remote component with a login of its
   own still fetches this one: it is public. Check the render before you
   land it: `kubectl kustomize .` should succeed on a machine with no GitHub
   login.

2. **Apply the new record.** The `Environment` record gains each host's
   name and paths, and its `ENDS` column prints the end as a time instead of
   `<invalid>`. The apply in step 1 brings both. An environment made before
   keeps its hosts and their addresses.

3. **Let environments reach your telemetry intake.** Every environment's
   network policy still denies the cluster's own ranges. List what every
   environment may reach inside them under `network.egress` in your
   settings, each with its namespace and pod labels, or its range, and its
   ports. `deploy/base/settings.yaml` shows it. With none listed, nothing
   changes.

4. **Offer the ways in you want.** A service may now name its hosts and
   paths, pick a door, and set its own class and annotations within what
   you allow. With no `hosts.ingress.doors` and no `hosts.ingress.allow`,
   every host is routed by your one rule, as before, and a service that asks
   for more is refused with the setting that refused. Name a door for each
   way in a host may take, such as behind your sign-in proxy, and list under
   `allow` the classes and annotation keys a service may set.

5. **Tie pipelines to their projects.** A grant for a forge's whole group
   covers every project it lists for every repository it matches. Add
   `projectsFor` to such a grant, one row per repository with the projects
   it may act on, and a job of one repository is refused another's project,
   with the grant and the repository named. A grant with no rows works as
   before.

6. **On Rancher, give the controller `updatepsa`.** See
   [Rights an admission check asks for](#rights-an-admission-check-asks-for).
   An estate that gave it for 4.10 has nothing to do. One that did not now
   sees why its environments are `Blocked`, in the environment's status and
   in `dev env show`, instead of a `Pending` with no reason.

Pods learn the environment's addresses at their next deploy: see "Every pod
knows the environment's addresses" in the program's `README.md`.

## Moving from 4.9

The install keeps its shape. Three things change with it, and an estate
that installed 4.9 does each once:

1. Point the component at the new release in your kustomization, and apply.
   That brings the new rights in the environment role and the wider scope
   policy. The controller binds the role in every environment already, so
   the environments you have get the new rights with no other step.

   ```bash
   kubectl apply -k .
   kubectl rollout status deployment/dev-controller -n dev-system
   ```

2. Add `logs`, `forward`, and `exec` to the `actions` of each grant that
   should have them, in your `settings.yaml`, and apply again. A grant
   written for 4.9 covers none of the three until you do, so until then
   `dev logs`, `dev forward`, `dev exec`, `dev sync`, and `dev debug` are
   refused with the grant's name. A grant with `actions: ["*"]` covers them
   already. `exec` is a shell in the pod, and a sync needs it: give it only
   where you would give a shell.

3. Let a connection through your gateway become a WebSocket on the
   controller's route, and check its idle timeout is a minute or more. See
   [The gateway in front](#the-gateway-in-front). Check it from a signed-in
   machine, in a project, once a service is deployed:

   ```bash
   dev exec <service> -- true && echo streams work
   ```

Nothing else moves. An environment made by 4.9 is used as it is.

## Moving from 4.8

The install changed shape, so an overlay written for 4.8 needs four changes:

1. `resources:` becomes `components:` for the line that names `deploy/base`.
2. The settings object loses `behavior: replace`. It is yours alone now.
3. Add `deploy` to the `actions` of each grant that should build and deploy.
   A grant with `actions: ["*"]` covers it already.
4. Add what a deploy needs to your settings file: `hosts.ingress`,
   `registry`, and, if you set nodes aside for environments, `placement`.
   `deploy/base/settings.yaml` explains each.

The settings object's name now ends in a hash of its content, so a changed
setting restarts the controller by itself.

## The image

`ghcr.io/dsswift/dev-controller:<version>`, for `linux/amd64`, at the same
version as the `dev` program. Each release is signed keyless with cosign by
digest and carries an SBOM attestation. Check one before you run it:

<!-- x-release-please-start-version -->
```bash
cosign verify ghcr.io/dsswift/dev-controller:4.12.0 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '^https://github.com/dsswift/dev.yaml/'
```
<!-- x-release-please-end -->

Pin the image by digest in your kustomization if your cluster requires it.

The builder's image is `moby/buildkit`, the rootless one, at the version this
release was tested with. An estate that mirrors its images names its own
under `builder.buildkit.image`.

## Changing a setting

Edit `settings.yaml`, or the registry beside it, and apply:

```bash
kubectl apply -k .
```

The settings object gets a new name, and the controller restarts with it.
A settings file the controller cannot use stops it at start, and its log
says which line. A change to `environmentRegistry` also changes the
admission policy, in the same apply.

## The three admission policies

None reads a parameter. What a policy must know is in the policy, or on the
namespace it judges, so nothing that goes missing lets a pod through.

| Policy | What it refuses |
|---|---|
| `dev-environment-images` | A pod outside an environment, or in one of kind `test` or `prod`, whose image comes from the environment registry. And such an image by tag anywhere: it runs only by the digest the controller built. |
| `dev-environment-placement` | A pod in an environment that does not carry the node selector and the tolerations of your `placement` setting. With no such setting, nothing. |
| `dev-controller-scope` | Any write by the controller outside a namespace that carries the environment label, its own namespace aside. Any command it runs in a pod, and any connection it makes to a pod's port, outside such a namespace. And any role binding by the controller but the environment role, to itself. |

## The controller's rights

- **Everywhere**, narrowed by the scope policy to environments: make and
  remove an environment's namespace, write its quota, limits, network
  policy, and secrets, and bind the environment role.
- **In each environment it made**, through the environment role: read pods,
  pod logs, events, workloads, and Services, write the kinds a project's
  manifests may hold, run a command in a pod, and reach a pod's port. It
  reads no secret there.
- **In its own namespace**: hold its lease, read the secrets it copies and
  the registry's push secret, and run its builder.

It reads a pod or a pod's log, runs a command in a pod, and reaches a pod's
port in an environment and in no other namespace. Those last two are how a
signed-in caller's `dev exec`, `dev sync`, `dev forward`, and `dev debug`
are carried: the caller holds no cluster credential, and the controller
checks its grant before it opens anything.

### Rights an admission check asks for

Some clusters check a write with a webhook of their own, and some of those
ask the writer for a right beyond what the install gives. The controller
writes each environment's namespace with the pod security labels of the
baseline standard. Rancher's webhook lets a caller set a pod security label
on a namespace only if it holds `updatepsa` on Rancher's projects, and
refuses the namespace otherwise. On Rancher, give the controller that one
right:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dev-controller-pod-security-labels
rules:
  - apiGroups: [management.cattle.io]
    resources: [projects]
    verbs: [updatepsa]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dev-controller-pod-security-labels
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dev-controller-pod-security-labels
subjects:
  - kind: ServiceAccount
    name: dev-controller
    namespace: dev-system
```

The install does not carry it, since it names another product's API. The
scope policy still keeps every namespace the controller writes to the ones
it made.

Whatever refuses one of the controller's own writes, the environment says
so: its phase is `Blocked`, and its message, which the API returns and
`dev env show` prints, gives the object and the cluster's own words. The
controller tries again with backoff, so the environment becomes ready once
the right is given.

## The gateway in front

A build sends its build context as one request and gets the build's log back
on the same one. A deploy stays open until every service is healthy. So
whatever stands in front of the controller must:

- take a large request body. The controller takes a build context of up to
  2 GiB;
- hold a request open for as long as a build takes. The controller sends a
  line at least every fifteen seconds, so a timeout on silence of a minute
  is enough;
- pass the answer on as it comes, with no buffering.

Logs, commands, and forwards are streams. Each is a WebSocket: one HTTP/1.1
connection that is upgraded and then stays open. So the gateway must also:

- let a request with `Connection: Upgrade` and `Upgrade: websocket` through
  to the controller, on the paths under
  `/v1/environments/<name>/services/<service>/`. Most ingress controllers do
  this with no setting. Some gateways need WebSockets turned on for the
  route;
- keep such a connection open for as long as it is used. The controller
  pings every stream every ten seconds and the caller answers, so a timeout
  on silence of a minute is enough here too. A gateway that ends every
  connection after a fixed time, used or not, ends a stream then: the `dev`
  program opens another, and a forward or a sync carries on.

Each is a setting of your gateway's own, on the controller's route alone.
`/v1/streams.md` on the controller says exactly what travels on a stream.

A command run with `dev exec` is part of its request's address, as it is in
the cluster's own API. A gateway that logs addresses logs the command. So a
secret goes to a command on its input, never as an argument.

## Telemetry

The controller writes one JSON object per line to stdout. It sends traces and
metrics over OTLP once the pod has `OTEL_EXPORTER_OTLP_ENDPOINT`. Add it, and
`OTEL_RESOURCE_ATTRIBUTES` if you want, with a patch in your kustomization.
With no endpoint it sends nothing.

| Metric | What |
|---|---|
| `dev_controller.build_info` | The running version. |
| `http.server.request.duration` | Each request, by route, method, and status. |
| `http.client.request.duration` | Each call out: the cluster's API and the sign-in service. |
| `dev_controller.task.duration` | Each reconcile, each build, and each deploy, by outcome. |
| `dev_controller.auth` | Sign-in checks: ok, refused, expired. |
| `dev_controller.refusals` | Requests refused on policy, by reason. |
| `dev_controller.environments` | Environments by kind and phase. |
| `dev_controller.environment.changes` | Environments made, extended, removed, and expired. |
| `dev_controller.builds` | Builds asked for: built, cached, or failed. |
| `dev_controller.deploys` | Deploys, by outcome and the environment's kind. |
| `dev_controller.streams` | Streams into pods that ended, by kind (`logs`, `exec`, `forward`) and how they ended: `done`, `failed`, or `cut`. |
| `dev_controller.streams.open` | Streams into pods open now, by kind. |
| `dev_controller.stream.bytes` | Bytes streams carried, by kind and direction. |
| `process.cpu.time`, `process.memory.usage`, and Go's runtime metrics | The process itself. |

A build or a deploy answers on one request for as long as it runs, so its
route's duration is the length of the work. Its time is also recorded as a
task, which is the number to watch.

A stream into a pod answers for as long as it is open, too. It has no
duration to watch: what to watch is how many are open and how they end. A
count of `cut` is the controller ending streams itself, because a token ran
out or an environment was removed.

Who did what is in the log, never in a metric: each change writes a line with
`event` (such as `environment.created`, `image.built`, or
`environment.deployed`) and `actor`. Every stream writes two: `stream.opened`
and `stream.closed`, each with the actor, the environment, the service, the
pod, and the kind of stream. The second also says how long the stream was
open, how many bytes went each way, and why it ended. A command is logged
by its program and by how many arguments it has, never by the arguments:
one may hold a secret.

## Another namespace

The install goes into `dev-system`. The admission policies and the role
bindings name that namespace and the `dev-controller` service account. To use
another, patch every place that names them, the policies included.

## Removing it

Remove every environment first (`dev env rm`, or delete the `Environment`
objects and wait for them to go). That also takes each environment's images
out of the registry. Then delete what the kustomization applied, and the
`dev-buildkit` Deployment, Service, and network policy the controller made
in `dev-system`. An `Environment` left behind holds a finalizer that only the
controller clears.
