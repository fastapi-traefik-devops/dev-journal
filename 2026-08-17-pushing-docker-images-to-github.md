# Pushing Docker Images to GitHub Container Registry

Meeting date: 2026-08-17

## Result of the Session

We prepared the shared Ubuntu VM to publish the project's Docker images to
GitHub Container Registry (GHCR):

- connected to the VM over SSH and joined the same `tmux` session;
- gave the VM access to the GitHub repository with an SSH key;
- cloned the project on the VM;
- reviewed the organization's settings for member package creation;
- created a classic GitHub Personal Access Token (PAT) with
  `write:packages` permission;
- logged Docker in to `ghcr.io`;
- installed the missing Docker Buildx component;
- built and pushed the backend image as
  `ghcr.io/fastapi-traefik-devops/fastapi-backend:v0.1.0`;
- verified the new `fastapi-backend` package on the organization's GitHub
  Packages page.

The frontend image was **not completed**. Its build filled the VM's root
filesystem, so the session stopped to fix storage first.

## Why We Are Publishing Images

During local development, Docker Compose can build images directly from the
project's Dockerfiles. Kubernetes normally runs on other machines, so it needs
a registry from which its nodes can pull already-built images.

```text
source code + Dockerfile
          |
       docker build
          |
      local image
          |
       docker push
          |
 GitHub Container Registry (ghcr.io)
          |
  Kubernetes pulls and runs it
```

An **image** is an immutable application template. A **container** is a running
instance of an image. GHCR is storage and distribution for images; it does not
run the application itself.

When the application changes, the normal release flow is to test it, build a
new image, assign it a new version tag, push it, and update the deployment to
use that tag.

## Working Together on the VM

We connected to the same remote shell using SSH and `tmux`:

```sh
ssh rootadmin@<vm-ip-address>
tmux attach
```

SSH creates the secure connection to the VM. `tmux` keeps a terminal session
alive after an SSH disconnect and lets two people view and type in the same
shell. This is convenient for pair work, but everyone attached can see commands
and output, so secrets must not be typed visibly there.

The repository was then cloned over SSH:

```sh
git clone git@github.com:fastapi-traefik-devops/fastapi-traefik-datascientest-project.git
cd fastapi-traefik-datascientest-project
```

## Access Used During the Session

Two different credentials solved two different problems:

| Credential | Used for | Does not automatically allow |
|---|---|---|
| SSH key | Clone, pull, and possibly push Git repository content | Publishing container images |
| Classic PAT with `write:packages` | Log in to GHCR and push packages | SSH access to the Git repository |

The VM's **public** SSH key was added to GitHub. The private key must remain on
the VM and must never be copied into GitHub, chat, or the repository. If the key
was added as a repository deploy key, it is limited to that repository; an
account SSH key can apply to all repositories the account may access.

**GHCR packages belong to the organization**, but authentication still represents
an individual GitHub user. Each person who pushes manually should use their own
token. GitHub's Container registry currently requires a **personal access token
(classic)** for command-line registry authentication. Uploading needs
`write:packages`; pulling a non-public image needs `read:packages`. If the
organization enforces SAML SSO, the token must also be authorized for that
organization.

Package permissions and repository permissions are related but are not always
the same. A container package pushed manually is not linked to its source
repository by default. After the first push, check the package's settings and:

1. connect it to `fastapi-traefik-datascientest-project`;
2. choose whether it should inherit repository access or use separate package
   permissions;
3. grant the repository access under **Manage Actions access** before a GitHub
   Actions workflow tries to update an already-existing package.

New command-line-published packages are private by default unless their
visibility is changed. A public GHCR image can be pulled anonymously; private
images require both a suitable token and user/package access.

## Commands and What They Mean

Run the commands from the cloned repository's root directory.

### 1. Check Buildx

```sh
docker version
docker buildx version
```

The VM used Ubuntu's `docker.io` packages, but its Buildx component was missing.
The package that worked with Ubuntu's repository was:

```sh
sudo apt-get update
sudo apt-get install docker-buildx
```

Docker's own APT repository uses a different package name:
`docker-buildx-plugin`. Do not assume both repository families use identical
package names or mix their packages accidentally.

**BuildKit** is Docker's modern build engine. **Buildx** is the Docker CLI
plugin that exposes BuildKit features. The backend Dockerfile uses a
BuildKit-only feature, so the legacy builder could not process it.

### 2. Log In Without Putting the Token in Shell History

```sh
read -rsp "GHCR token: " CR_PAT
echo
printf '%s' "$CR_PAT" | docker login ghcr.io -u dionis1 --password-stdin
unset CR_PAT
```

- `read -s` hides input while it is typed;
- `CR_PAT` exists only in the current shell;
- `--password-stdin` keeps the secret out of Docker's command arguments;
- `unset` removes the shell variable after login.

`Login Succeeded` proves that the credentials were accepted. It does not by
itself prove that the account can write a particular organization package; the
first push tests that authorization.

### 3. Build the Backend Image

```sh
DOCKER_BUILDKIT=1 docker build \
  -t ghcr.io/fastapi-traefik-devops/fastapi-backend:v0.1.0 \
  ./backend
```

The image reference has four useful parts:

| Part | Value | Meaning |
|---|---|---|
| Registry | `ghcr.io` | Server that stores the image |
| Namespace | `fastapi-traefik-devops` | GitHub organization |
| Image | `fastapi-backend` | Package/image name |
| Tag | `v0.1.0` | Human-readable release version |

`-t` assigns the complete image name and tag. `./backend` is the **build
context**: the files available to the Dockerfile's `COPY` and `ADD`
instructions. Docker also finds `./backend/Dockerfile` there by default. Using
`.` instead would send the repository root as the context and could copy the
wrong files or make the build unnecessarily large.

Verify the local result if needed:

```sh
docker image ls ghcr.io/fastapi-traefik-devops/fastapi-backend
```

### 4. Push and Verify

```sh
docker push ghcr.io/fastapi-traefik-devops/fastapi-backend:v0.1.0
```

The local tag tells Docker both the registry destination and package name.
Docker uploads content in layers and reuses layers already present in the
registry. We then found `fastapi-backend`, version `v0.1.0`, on the
organization's **Packages** page.

After the storage problem is fixed, the frontend follows the same pattern:

```sh
DOCKER_BUILDKIT=1 docker build \
  -t ghcr.io/fastapi-traefik-devops/fastapi-frontend:v0.1.0 \
  ./frontend

docker push ghcr.io/fastapi-traefik-devops/fastapi-frontend:v0.1.0
```

## Problems We Found

### BuildKit Enabled, but Buildx Missing

The first backend build required BuildKit. Retrying with
`DOCKER_BUILDKIT=1` exposed the real installation problem: Docker was present,
but the Buildx CLI component was absent. Installing Ubuntu's `docker-buildx`
package fixed it.

`DOCKER_BUILDKIT=1` changes the builder for that one command; it cannot supply a
missing plugin.

### No Space Left on the VM

The frontend build stopped because `/` had no free space. The VM disk/partition
was about 30 GB, but the root LVM logical volume and filesystem used only about
15 GB. This illustrates that these are separate layers:

```text
virtual disk -> partition -> LVM physical volume -> logical volume -> filesystem
```

Increasing a Proxmox virtual disk alone does not necessarily enlarge the Linux
partition, LVM logical volume, and filesystem. Each required layer must be
checked and expanded safely. Because a mistake can damage the VM and its keys,
this should first be practised on a clone or snapshot.

#### Direct CLI Output

```bash
root-admin@group1-fast-api:~$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              392M  1.1M  391M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   15G   14G   18M 100% /
tmpfs                              2.0G     0  2.0G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  200M  1.6G  11% /boot
tmpfs                              392M   12K  392M   1% /run/user/1000
root-admin@group1-fast-api:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   32G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   30G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0   15G  0 lvm  /
sr0                        11:0    1  2.6G  1 rom
root-admin@group1-fast-api:~$ sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
[sudo] password for root-admin:
  Size of logical volume ubuntu-vg/ubuntu-lv changed from <15.00 GiB (3839 extents) to <30.00 GiB (7679 extents).
  Logical volume ubuntu-vg/ubuntu-lv successfully resized.
root-admin@group1-fast-api:~$ sudo resize2fs  /dev/mapper/ubuntu--vg-ubuntu--lv
resize2fs 1.47.0 (5-Feb-2023)
Filesystem at /dev/mapper/ubuntu--vg-ubuntu--lv is mounted on /; on-line resizing required
old_desc_blocks = 2, new_desc_blocks = 4
The filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv is now 7863296 (4k) blocks long.

root-admin@group1-fast-api:~$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              392M  1.1M  391M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   30G   14G   15G  50% /
tmpfs                              2.0G     0  2.0G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/sda2                          2.0G  200M  1.6G  11% /boot
tmpfs                              392M   12K  392M   1% /run/user/1000
```

#### What the Commands Did

The first `df -h` showed the immediate problem: the root filesystem was only
15 GB, had 14 GB in use, and had just 18 MB available. It was effectively full.

`lsblk` then showed that the underlying storage was already larger. The virtual
disk `/dev/sda` was 32 GB and its LVM partition `/dev/sda3` was 30 GB, but the
root logical volume `ubuntu-vg/ubuntu-lv` used only 15 GB. Therefore, neither
Proxmox nor the disk partition needed another resize; the unused capacity was
inside the LVM volume group.

```sh
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

This enlarged the root logical volume from about 15 GB to 30 GB. `+100%FREE`
means "assign all currently unused extents in this volume group to this logical
volume." This solved the LVM allocation problem, but it did not yet enlarge the
filesystem stored inside the logical volume.

```sh
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

This expanded the ext-family filesystem while it remained mounted as `/`. The
final `df -h` verified the complete result: the root filesystem increased from
15 GB to 30 GB, available space increased from 18 MB to 15 GB, and usage
dropped from 100% to 50%. No reboot was required.

One trade-off is that `+100%FREE` leaves no unallocated space in the volume
group for another logical volume or an LVM snapshot. That is acceptable only
when assigning all remaining capacity to `/` is intentional.

Useful read-only diagnostics:

```sh
df -h
lsblk
sudo pvs
sudo vgs
sudo lvs
docker system df
```

`df -h` shows filesystem capacity; `lsblk` shows block-device relationships;
the LVM commands show physical volumes, volume groups, and logical volumes;
`docker system df` shows space used and reclaimable by Docker.

Do not run `docker system prune` blindly. It deletes unused Docker objects and
build cache; inspect what is reclaimable and understand which images and
containers are still needed first.

## Important Security Follow-up

During the meeting, the real PAT was pasted directly into a shell command and
saved in a plain-text file on the shared VM. Treat it as exposed:

1. delete/revoke that token in GitHub **Settings -> Developer settings ->
   Personal access tokens -> Tokens (classic)**;
2. create a replacement with an expiration and only `write:packages` (which
   includes package-reading capability) rather than broad repository access;
3. remove the token file from the VM and ensure it was never committed;
4. use the hidden-input login sequence above;
5. run `docker logout ghcr.io` when persistent registry access is no longer
   needed.

Docker may store login data in `~/.docker/config.json`. Without a configured
credential helper, it is only base64-encoded, not encrypted. On a shared or
long-lived VM, configure a credential store or prefer short-lived CI
credentials.

## Next Steps

1. Rotate the exposed PAT and log in safely.
2. Retry the frontend build and push `fastapi-frontend:v0.1.0` now that the
   root filesystem has enough free space.
3. Connect both GHCR packages to the source repository and confirm visibility,
   member access, and GitHub Actions access.
4. Replace manual publishing with a GitHub Actions workflow using its temporary
   `GITHUB_TOKEN`.
5. Create Kubernetes manifests for Minikube that reference the published image
   names. If images remain private, also configure an image-pull secret.
6. Prefer immutable release tags or image digests in Kubernetes instead of
   relying only on `latest`.

## Official References

- [GitHub: Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [GitHub: Package permissions](https://docs.github.com/en/packages/learn-github-packages/about-permissions-for-github-packages)
- [GitHub: Package access and visibility](https://docs.github.com/en/packages/learn-github-packages/configuring-a-packages-access-control-and-visibility)
- [GitHub: Keeping API credentials secure](https://docs.github.com/en/rest/authentication/keeping-your-api-credentials-secure)
- [Docker: Build context](https://docs.docker.com/build/concepts/context/)
- [Docker: BuildKit](https://docs.docker.com/build/buildkit/)
- [Docker: `docker login`](https://docs.docker.com/reference/cli/docker/login/)
- [Docker: `docker system df`](https://docs.docker.com/reference/cli/docker/system/df/)
- [Docker: Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

## Short summary !!!

### Create a GitHub Personal Access Token (PAT)

Docker needs credentials to push to GHCR — your SSH key (used for git) is unrelated and won't work here.

There's no way around this — GHCR only supports personal access tokens (classic) for `docker login`, and each token belongs to one account. So every teammate who wants to push manually does:

1. Go to GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. **Generate new token (classic)**
3. Give it a name like `ghcr-push`, set an expiration
4. Check these scopes: `write:packages` (also pulls in `read:packages` automatically)
5. Generate it and **copy the token immediately** — GitHub only shows it once

### Build and push images

```sh
cat ~/.ssh/GHTOKEN.txt  
echo 'ghp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX' | docker login ghcr.io --password-stdin -u dionis1

docker build -t ghcr.io/fastapi-traefik-devops/fastapi-backend:v0.1.0 ./backend
docker build -t ghcr.io/fastapi-traefik-devops/fastapi-frontend:v0.1.0 --build-arg VITE_API_URL=https://api.fastapi.local --build-arg NODE_ENV=production ./frontend

docker images


docker push ghcr.io/fastapi-traefik-devops/fastapi-backend:v0.1.0
docker push ghcr.io/fastapi-traefik-devops/fastapi-frontend:v0.1.0
```

`docker build` already works fine without `DOCKER_BUILDKIT=1` now because buildx is installed, we don't need the prefix anymore.

Even though `docker compose up` starts 5 containers (`db`, `adminer`, `prestart`, `backend`, `frontend`), only **2 of them are actually built from our code**.

| Service    | image:                            | build:       | What it actually is                                                                               |
| ---------- | --------------------------------- | ------------ | ------------------------------------------------------------------------------------------------- |
| `db`       | `postgres:12`                     | —            | pulled from Docker Hub, not your code                                                             |
| `adminer`  | `adminer`                         | —            | pulled from Docker Hub, not your code                                                             |
| `prestart` | `${DOCKER_IMAGE_BACKEND}:${TAG}`  | `./backend`  | **same image as backend** — just runs a different command (`bash scripts/prestart.sh`) against it |
| `backend`  | `${DOCKER_IMAGE_BACKEND}:${TAG}`  | `./backend`  | our code                                                                                          |
| `frontend` | `${DOCKER_IMAGE_FRONTEND}:${TAG}` | `./frontend` | our code                                                                                          |
`prestart` isn't a separate image — it points at the exact same `image:` name as `backend`, just with `context: ./backend` too. Compose builds it once and both services use that one image, just running different commands. So there's nothing extra to build there.

#### What `VITE_API_URL` is

> Good question — this is actually one of the more common gotchas people hit with frontend containers, worth understanding properly.

Vite is the build tool for your frontend (React/Vue/etc). When you run `vite build`, it produces static HTML/CSS/JS files — no server-side logic, just files a browser downloads and runs. Any env var prefixed `VITE_` gets **baked directly into that JS bundle at build time**, replacing `import.meta.env.VITE_API_URL` in the source code with the literal string value. After the build finishes, it's just a hardcoded string sitting in a `.js` file — there's no process reading environment variables at runtime anymore, because there's no server-side runtime at all; it's static files served by nginx (or whatever serves `frontend`'s container).

##### Why you need it

The frontend runs in the user's **browser**, not inside your Docker network. When the React/Vue app makes an API call ("log in", "fetch users"), that HTTP request goes out from the browser to whatever URL is baked into the bundle. If that URL is wrong, missing, or points to `localhost:8000` (a common dev default), the browser will try to reach that — and fail, because the actual backend lives somewhere else entirely from the browser's perspective.

##### Why you don't _see_ it with `docker compose up`

You don't need to set it — but you **are** still getting it, just automatically. Look at this part of your compose file again:

```yaml
frontend:
  build:
    context: ./frontend
    args:
      - VITE_API_URL=https://api.${domain/?Variable not set}

```

`docker compose up` reads `DOMAIN` from your `.env` file and substitutes it in automatically, every time it builds. So it's not that it's not needed — Compose is just doing that work invisibly for you, the same way it substitutes `POSTGRES_PASSWORD` and everything else `${...}`. When you ran plain `docker build ./frontend` by hand, none of that `.env`-reading and substitution machinery ran, so I had to pass `--build-arg` manually to replicate what Compose does for you automatically.

##### Does the public frontend depend on it? Yes — directly

This is the important part, and it has a real consequence for your Kubernetes/multi-environment work coming up:

**The exact `VITE_API_URL` value is permanently frozen into that specific image tag.** If you build the frontend once with `VITE_API_URL=https://api.fastapi.local`/ and push it as `v0.1.0`, that image will **always** try to call `api.fastapi.local` from the browser — no matter which cluster, namespace, or domain you later deploy that same image tag to. You cannot fix it by changing a Kubernetes env var or ConfigMap at deploy time, because there's no runtime process reading env vars in a static frontend container — it's just files.

**Concretely, this means:** when you get to Phase 4 (dev + prod environments) in your checklist, you can't just reuse one `fastapi-frontend:v0.1.0` image across both `dashboard-dev.fastapi.local` and `dashboard.fastapi.local` if the API domain differs between them — you'd need **two separate frontend builds**, one per environment, each with its own `VITE_API_URL` build arg. This is a real fork in the road worth flagging now, before you build the CI pipeline: either (a) build the frontend once per environment with different build args, or (b) restructure so the frontend calls a relative path like `/api` and let the Ingress/reverse-proxy route it to the right backend regardless of domain — that second approach is more "production-grade" precisely because it avoids baking environment-specific URLs into the image at all.

### Verify

#### Remote images
Go to https://github.com/fastapi-traefik-devops → **Packages** tab in the browser. You should see both `fastapi-backend` and `fastapi-frontend` listed with tag `v0.1.0`.

### How to rebuild & repush the images ?

#### Can I overwrite `v0.1.0`? Yes, technically.

```bash
docker build -t ghcr.io/fastapi-traefik-devops/fastapi-backend:v0.1.0 ./backend
docker push ghcr.io/fastapi-traefik-devops/fastapi-backend:v0.1.0

```

This works exactly as written — GHCR doesn't stop you from pushing a new image under an existing tag (unless someone explicitly turns on tag protection rules for the org). The old `v0.1.0` digest gets replaced by the new one, same name, different content.

#### Why this causes real, confusing bugs

The tag is just a pointer/label — it's not the actual identity of the image. Various tools **cache based on the tag string**, not the content, so overwriting silently creates stale-cache problems:

- **Docker locally**: if you already have `fastapi-backend:v0.1.0` pulled, `docker compose up` with that tag **won't automatically re-pull** — you'd need an explicit `docker compose pull` first, or `docker pull ...:v0.1.0` directly. Otherwise you keep running the _old_ code and won't understand why your changes aren't showing up.
- **Kubernetes**: this is the sharper trap. If a Deployment already has a pod running `image: ...:v0.1.0`, and the default `imagePullPolicy` behavior kicks in (`IfNotPresent` for anything that isn't literally `latest`), **the kubelet won't re-pull at all** — it sees the tag already cached on the node and reuses it. You push a new `v0.1.0`, apply the same YAML (no change, since the tag string is identical), and Kubernetes does _nothing_ — no rollout, same old pod keeps running. This is a very common "why isn't my fix showing up" moment.

#### The two real options

##### A - always bump the version

**Option A — always bump the version (recommended, matches what we discussed earlier)**

```bash
docker build -t ghcr.io/fastapi-traefik-devops/fastapi-backend:v0.1.1 ./backend
docker push ghcr.io/fastapi-traefik-devops/fastapi-backend:v0.1.1

```

Then update wherever that tag is referenced (`docker-compose.registry.yml` via `export TAG=v0.1.1`, or `k8s/04-backend.yaml`'s `image:` line). Every version is a distinct, addressable artifact — you can always tell exactly which code is running, and rollback is just "point back at v0.1.0."

##### B — force to actually re-pull

**Option B — force everything to actually re-pull the overwritten tag**, if you just want fast local iteration without bumping numbers constantly:

```bash
# Docker Compose:
docker compose -f docker-compose.registry.yml pull
docker compose -f docker-compose.registry.yml up -d --force-recreate

# Kubernetes:
kubectl rollout restart deployment/backend -n fastapi-app

```

`kubectl rollout restart` forces new pods, but it only helps if `imagePullPolicy: Always` is set on the container — otherwise the kubelet still reuses its local cached layer for that tag string. Worth adding that explicitly if you go this route:

```yaml
containers:
  - name: backend
    image: ghcr.io/fastapi-traefik-devops/fastapi-backend:v0.1.0
    imagePullPolicy: Always

```

#### What I'd actually suggest for now

Since you're specifically testing "did my rebuild take effect" across three scenarios, Option B's silent-cache trap will cost you more debugging time than it saves typing. Bump the tag (`v0.1.1`, `v0.1.2`, ...) each time you rebuild during this testing phase — it's one extra keystroke and removes an entire category of "why isn't this working" confusion while you're still building intuition for how each tool caches things. Once your CI pipeline is doing the tagging automatically (git SHA, from the workflow set up earlier), this stops being manual toil anyway.
