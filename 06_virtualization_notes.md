# PDF 6 — Virtualization (VMs, Docker, Vagrant)

## What virtualization is

> Virtualization = creating a software-based version of something previously physical.

A **VM (Virtual Machine)** is software emulating an entire computer. From the inside it believes it has real hardware; in reality, software fakes the CPU/RAM/disk/network.

### Five flavors

| Flavor | What's virtualized | Example |
|---|---|---|
| Desktop | A whole PC running another OS | VirtualBox running Linux on Windows |
| Server | Production servers, often many per box | One physical box running 20 customer VMs |
| Network | Switches, routers, links | VLAN, VXLAN, SDN |
| Storage | Disks, filesystems | RAID, SAN |
| Application | Single app's environment | Docker containers |

---

## Why virtualize?

### 1. Consolidation -> save money and energy

Example transformation (from slides):

| | Before | After |
|---|---|---|
| Servers | 1000 | 50 |
| Cables | 3000 | 300 |
| Racks | 200 | 10 |
| Power feeds | 400 | 20 |

**~20x compression** in physical footprint. CFOs love this.

### 2. Simplify management

VM = just files -> can:
- Snapshot before risky changes (rollback)
- Clone for testing
- Live-migrate to another physical host
- Back up by copying files
- Spin up in seconds

---

## A VM is just files

Four files per VM:

1. **Configuration file** — VM settings (CPU, RAM, disk paths, network)
2. **Hard disk file(s)** — virtual disk (`.vmdk`, `.vdi`)
3. **State file** — saved state when suspended
4. **In-memory file** — RAM snapshot at suspend time

This is why backup/clone/snapshot/migrate all work -- they're just file operations.

---

## The hypervisor — the secret sauce

> **Hypervisor (VMM)** = program that lets multiple OSes share a single host.
> Each VM thinks it owns the hardware; the hypervisor actually manages it.

Hypervisor's jobs:
- Pretend to be hardware to each VM
- Allocate real CPU/RAM/disk/network to VMs
- Keep VMs isolated (one crash shouldn't take others down)
- Schedule which VM runs when

### Type 1 vs Type 2 (the exam question)

**Type 2 (hosted) — runs on top of a host OS, like a regular app.**
```
+-------------------------+
| Guest OS  | Guest OS    |
| (Win)     | (Linux)     |
+-----------+-------------+
|  Hypervisor (an app)    |
+-------------------------+
|  Host OS (your laptop)  |
+-------------------------+
|     Physical hardware   |
+-------------------------+
```
Examples: **VirtualBox**, VMware Workstation, Parallels.
Use case: desktop, dev, test.
Tradeoff: easy to install, slower due to host OS overhead.

**Type 1 (bare-metal) — IS the OS, runs directly on hardware.**
```
+---------+---------+---------+
| Guest 1 | Guest 2 | Guest 3 |
+---------+---------+---------+
|       Hypervisor (the OS)   |
+-----------------------------+
|      Physical hardware      |
+-----------------------------+
```
Examples: **VMware ESXi**, Hyper-V Server, **Xen**, **KVM**.
Use case: production servers, datacenters, AWS/Azure/GCP.
Tradeoff: max performance, harder to manage (it IS the OS).

### Quick comparison

| | Type 2 (hosted) | Type 1 (bare-metal) |
|---|---|---|
| Where it lives | On top of host OS | Directly on hardware |
| Examples | VirtualBox, VMware Workstation | ESXi, Hyper-V, Xen, KVM |
| Performance | Lower | High |
| Use case | Desktop, dev, test | Production, cloud |

---

## Vendor landscape

| Vendor | Product | Type |
|---|---|---|
| Oracle | **VirtualBox** | Type 2 (free) |
| VMware | Workstation / Player | Type 2 |
| VMware | **ESXi / vSphere** | Type 1 (enterprise) |
| Microsoft | Hyper-V | Type 1 (also Type 2 on desktop) |
| Citrix | XenServer | Type 1 |
| Linux | KVM (+ QEMU) | Type 1 |

For this course, **VirtualBox** is the one to know -- free, cross-platform, what Vagrant uses.

---

## VMware's killer features

### VMotion — live migration
Move a running VM between hosts with **no downtime**. RAM streams over while VM keeps running; brief freeze copies last delta; resumes on new host.

> **Connects to VXLAN:** VMotion needs the VM to keep its L2 context across hosts.
> If the hosts are on different L3 subnets, you need VXLAN (or similar overlay) to make it work.

Why operators use it:
- Maintenance (move VMs off host before reboot)
- Load balancing (move VMs to less-loaded host)
- Power saving (consolidate at night)
- Predicted hardware failure (move before it dies)

### Storage VMotion
Move the VM's **disk file** to different storage while VM keeps running.
- VMotion: compute moves, disk stays.
- Storage VMotion: disk moves, compute stays.

### High Availability (HA)
**Auto-restart VMs after the physical host crashes.** Cluster heartbeats each other; survivors restart the dead host's VMs.

| | VMotion | HA |
|---|---|---|
| Downtime | None | Brief (hard restart) |
| Requires | Healthy source host | Dead source host (it's the fallback) |
| When used | Planned moves | Unplanned failures |

### Memory reclamation
Over-commit RAM via **transparent page sharing** (dedupe identical pages across VMs), compression, and ballooning. Lets you promise more RAM than the host physically has.

---

## Guest OS examples

| Image | Use | Login |
|---|---|---|
| **Kali Linux** | Pre-loaded with pen-testing tools (Wireshark, Nmap, Metasploit) | `root` / `toor` |
| **Metasploitable** | Intentionally vulnerable Linux. Target for attack practice. | `msfadmin` / `msfadmin` |
| **Windows eval ISOs** | Test Windows-specific software without a license. | (set during install) |

> Never expose Metasploitable to a real network.

---

## VirtualBox networking modes (CRITICAL for labs)

### NAT (default)
Each VM behind its own private NAT router.
- VMs get internet.
- VMs **CANNOT** see each other.
- Useless for networking labs.

### NAT Network
All VMs share one NAT router. Same private LAN.
- VMs see each other.
- VMs reach internet.
- Host can't reach VMs directly.
- **Best for networking labs.**

### Bridged Adapter
Each VM appears as a real device on your real LAN. Gets an IP from your real DHCP server.
- VMs are peers of your host on the real network.
- Other machines on your LAN can reach VMs.
- Doesn't work if your WiFi/router refuses to hand out IPs (e.g. some campus networks).

### Other modes
- **Internal Network**: full isolation, no internet.
- **Host-only Adapter**: only host can reach VMs; no internet.

### Cheat sheet

| Goal | Mode |
|---|---|
| Just internet | NAT |
| **VMs talk to each other** (labs) | **NAT Network** |
| VMs visible on real LAN | Bridged |
| Isolated lab with no internet | Internal Network |
| Host-only access | Host-only Adapter |

---

## File transfer host <-> VM

- **Drag and drop**: `Devices -> Drag and Drop -> Bidirectional`
- **Shared folder**: pick folder on host, mount inside VM:
  ```
  mkdir ~/shared
  sudo mount -t vboxsf Downloads ~/shared
  ```
- **Cloud storage**: slow but always works

---

## Vagrant — the VM autopilot

> Vagrant = open-source tool that **automates the lifecycle of a development VM**.
> Describe the VM in a text file; Vagrant builds it.

Replaces "1 hour of clicking through installers" with:
```bash
vagrant init ubuntu/jammy64
vagrant up
```

Key facts:
- **Vagrant is NOT a hypervisor.** It drives one underneath (VirtualBox usually).
- The Vagrantfile is text -> versioned in Git -> identical VM on every dev's machine.
- Killer use case: **reproducible environments.** No more "works on my machine."

### Three pieces

1. **Provider** — the hypervisor underneath (VirtualBox, VMware, libvirt). Course uses VirtualBox.
2. **Box** — a pre-baked VM image template. Download once, stamp out copies.
   - Browse: https://app.vagrantup.com/boxes/search
3. **Vagrantfile** — Ruby text file describing what VM(s) you want.

### Minimal Vagrantfile

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.provision :shell, path: "bootstrap.sh"
end
```

Reads as: use Ubuntu 22.04 box, forward host:8080 -> VM:80, run bootstrap.sh on first boot.

### Commands to memorize

| Command | What it does |
|---|---|
| `vagrant init <box>` | Create starter Vagrantfile |
| `vagrant up` | Boot VM (downloads box, runs provisioning) |
| `vagrant ssh` | SSH into VM |
| `vagrant halt` | Clean shutdown (data persists) |
| `vagrant suspend` / `resume` | Pause/resume |
| `vagrant reload` | Restart, re-read Vagrantfile |
| `vagrant destroy` | **Delete VM completely** |
| `vagrant box add/remove <name>` | Manage cached boxes |
| `vagrant status` | Is it running? |

Lifecycle: `init` -> `up` -> work -> `halt` (end of day) -> `up` (next day) -> eventually `destroy`.

### Provisioning — auto-install software on first boot

`bootstrap.sh`:
```bash
#!/usr/bin/env bash
apt-get update
apt-get install -y nginx
systemctl enable nginx
```

Vagrantfile:
```ruby
config.vm.provision :shell, path: "bootstrap.sh"
```

Result: every `vagrant up` produces a fresh Ubuntu with nginx installed. Zero manual steps.

Other provisioners (for complex setups): Puppet, Chef, Ansible, Docker. Shell scripts cover 90% of needs.

### Networking config

| Config | Effect |
|---|---|
| `config.vm.network "forwarded_port", guest: 80, host: 8080` | host:8080 -> VM:80 |
| `config.vm.network "private_network", ip: "192.168.50.10"` | Static IP on host-only network |
| `config.vm.network "public_network"` | VM = peer on your real LAN (Bridged) |

### Multi-machine setups (POWERFUL)

One Vagrantfile, multiple VMs:

```ruby
Vagrant.configure("2") do |config|
  config.vm.define "web" do |web|
    web.vm.box = "ubuntu/jammy64"
    web.vm.network "private_network", ip: "192.168.50.10"
  end
  config.vm.define "db" do |db|
    db.vm.box = "ubuntu/jammy64"
    db.vm.network "private_network", ip: "192.168.50.11"
  end
end
```

`vagrant up` boots both. `vagrant ssh web` / `vagrant ssh db` to enter each.
This is how you build **mini-datacenter setups** for testing.
**The SDN lab uses exactly this pattern.**

### Why Vagrant matters for the course

- SDN hands-on lab uses Vagrant to spin up controller + topology VMs.
- Docker/Vagrant lab explicitly provisions a Docker host via Vagrant.
- For your **project**, package your test environment in a Vagrantfile so the grader can reproduce in 2 minutes.

---

## Docker concepts

### The "matrix from hell" problem

N services (web, DB, queue, workers, ...) x M environments (dev laptop, QA, staging, prod, cloud, customer DC) = combinatorial chaos.

Each cell: "will this service install and run here?" Every "?" is a potential weekend lost to "works on my machine, not yours."

### The shipping container analogy

Shipping industry had this exact problem in the 1950s. Multiplicity of goods x multiplicity of transport = chaos.

Solution: **the intermodal shipping container**. Standard steel box. Whatever's inside, the **outside** is always the same. Cranes, ships, trains, trucks all handle the same box.

> Docker is the intermodal shipping container for software.
> Standardize the outside; allow anything inside.

### Containers vs VMs (CRITICAL diagram for exam)

```
   VMs                          Containers

   App  App  App                App App App App
   Libs Libs Libs               Libs    Libs
   ┌──┐ ┌──┐ ┌──┐               ┌──────────────┐
   │OS│ │OS│ │OS│               │  Docker Engine│
   └──┘ └──┘ └──┘               ├──────────────┤
   ┌────────────┐               │   Host OS    │
   │ Hypervisor │               ├──────────────┤
   ├────────────┤               │   Server     │
   │  Host OS   │               └──────────────┘
   ├────────────┤
   │  Server    │
   └────────────┘
```

**One-line difference:** VMs each have a full guest OS. Containers share the host's kernel.

Consequences:

| | VMs | Containers |
|---|---|---|
| Boot time | Minutes (full OS boot) | Milliseconds (just a process) |
| Disk | GBs per VM | MBs per container |
| Density | Tens per host | **Thousands** per host |
| Isolation | Stronger (full OS boundary) | Weaker (shared kernel) |
| Cross-OS | Run Windows on Linux | All containers use host's kernel |

> **Containers are NOT lightweight VMs.** Fundamentally different mechanism.
> VM = fake hardware running real OS. Container = isolated process group inside real OS.

### Why containers are lightweight

1. **No guest OS** -- only app + libs in each container.
2. **Copy-on-write layered storage** -- 10 containers from the same image share the underlying files (read-only); changes stored as per-container diffs.

### The 5-word vocabulary

| Term | What it is | Analogy |
|---|---|---|
| **Docker Image** | Read-only blueprint (filesystem + libs + entry point) | A **class** in OOP |
| **Docker Container** | Running instance of an image | An **object** instantiated from a class |
| **Docker Engine** | The runtime (daemon + CLI) | Like the JVM for Java |
| **Dockerfile** | Text recipe to build an image | A class definition file |
| **Docker Hub (Registry)** | Central place to publish/pull images | GitHub but for images |

Flow:
```
Dockerfile --(docker build)--> Image --(docker run)--> Container

                  published to / pulled from
                      Docker Hub
```

Images named `name:tag`, e.g. `nginx:1.25`, `python:3.11-slim`.

### The two audiences

**Dev ("Dan")**: "set up once in Dockerfile, run anywhere." No more "works on my machine."

**Ops ("Oscar")**: "every container is identical from outside -- same start/stop/log/monitor." Crane doesn't care about cargo.

---

## Docker in practice

### Basic commands

```bash
docker image pull nginx:latest          # download image
docker image ls                         # list local images
docker container run -d -p 8080:80 --name web nginx:latest    # run
docker container ps                     # list running containers
docker container stop web
docker container rm web
docker image rm nginx:latest
docker build -t my-app:1.0 .            # build image from Dockerfile in current dir
docker image push my-user/my-app:1.0
```

### Common flags

| Flag | Means |
|---|---|
| `-d` | Detached (background) |
| `-p host:container` | Port mapping |
| `--name X` | Human-friendly name |
| `-v host:container` | Volume mount |
| `-e KEY=value` | Environment variable |
| `--rm` | Auto-delete when stopped |
| `-it` | Interactive terminal |

### One-liner web server demo

```bash
docker run -d -p 80:80 nginx
# now open http://localhost
```

No install, no config, no service management. The "wow" moment.

---

### Dockerfile (the recipe)

```dockerfile
FROM nginx:latest
LABEL maintainer="chaitanya@example.com"
COPY ./index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

Build: `docker build -t my-page:1.0 .`

### Common instructions

| Instruction | What it does |
|---|---|
| `FROM` | Base image (every Dockerfile starts with this) |
| `RUN` | Run a shell command at BUILD time (e.g. install software) |
| `COPY` | Copy file from host into image |
| `ADD` | Like COPY + can fetch URLs / unpack archives |
| `WORKDIR` | Set working directory for following commands |
| `ENV` | Set env variable in image |
| `EXPOSE` | Document container's listening port (doesn't actually open it) |
| `VOLUME` | Declare mount point for persistent data |
| `CMD` | Default command when container starts |
| `ENTRYPOINT` | Like CMD but harder to override |

### Each instruction = one layer

Top-to-bottom = stack of read-only layers. Changing a line invalidates that layer + all below it.

> Put slow-changing steps (FROM, package installs) NEAR THE TOP so they get cached.
> Put fast-changing steps (COPY source code) NEAR THE BOTTOM.

### ENTRYPOINT vs CMD

- ENTRYPOINT = the command to run
- CMD = default arguments to that command
- `docker run` args replace the CMD part.
- Simple Dockerfiles can just use CMD.

---

### Volumes — making data persistent

> By default, container writes disappear when you `docker rm` it.
> Volumes mount a host directory into the container so data persists.

```bash
docker run -d -v /data/db:/var/lib/postgres postgres
```

Format: `-v <host path>:<container path>`.

Dev pattern -- mount source code for live editing:

```bash
docker run -v $(pwd):/usr/src/app my-app
```

Uses:
- Persistent databases (data outlives the container).
- Live source code editing (no rebuild needed).
- Heavy I/O performance (native host FS faster than writable layer).

---

### Networking

By default, every container is on a **bridge network** -- virtual switch inside Docker. Containers on the same bridge can ping each other.

### Port mapping (the `-p` flag)

```bash
docker run -d -p 8080:80 nginx
```

"When something hits host:8080, forward to container:80." Required to expose to outside world.

```
       Outside world
             |
       Host:8080
             |
             v
       Container:80 (nginx)
```

### Custom networks

```bash
docker network create -d bridge my-net
docker run -d --network my-net --name web nginx
docker run -d --network my-net --name db postgres
```

Now `web` can resolve `db` by name (Docker provides DNS within the network). No IPs to memorize.

---

### Docker Compose -- multi-container apps

Single YAML file describing whole stack. One command to start everything.

```yaml
version: '3'
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    depends_on:
      - api

  api:
    build: ./api
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=database

  database:
    image: postgres:15
    volumes:
      - db-data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=secret

volumes:
  db-data:
```

Commands:
```bash
docker-compose up -d        # start everything
docker-compose down         # stop everything
docker-compose up -d --build  # rebuild after changes
```

**Your SDN project will probably use Docker Compose** to bundle the controller + topology + tooling.

---

### How this ties to the course

- **SDN labs** run controllers and topologies in Docker containers.
- **Kathará** (next emulator we cover) is implemented using Docker containers as virtual routers.
- **Your project** -- package as docker-compose.yml so grader runs in 30 seconds.

---

# PDF 6 Final Recap

| Concept | Bottom line |
|---|---|
| Virtualization | Software emulation of something physical |
| VM | "Computer in a computer." Full guest OS. |
| Hypervisor | Type 1 (bare-metal, production) vs Type 2 (hosted, desktop) |
| VMotion, HA | Live migration + auto-restart -- cloud-grade VM features |
| VirtualBox networking | NAT / **NAT Network** (labs!) / Bridged |
| Vagrant | VM autopilot -- Vagrantfile -> fully provisioned VM |
| Container | Isolated process group on host kernel. NOT lightweight VM. |
| Matrix from hell | N services x M envs = chaos. Containers solve it. |
| Containers vs VMs | Shared kernel -> ms boot, MB disk, **thousands per host** |
| Copy-on-write | Share base, store only diffs |
| Dockerfile | Recipe -> image. Each line = one layer. |
| Volumes | Persistent storage outlives container |
| Bridge + `-p` | Container-to-container by default; `-p` to expose externally |
| Docker Compose | Multi-container apps in one YAML |

**Big theme tying back to the course:** Docker = virtualization at the *application* layer (instead of hardware). Same softwarization pattern as SDN (control plane) and SDR (radio waveform). Different level of the stack, same philosophy: replace specialized hardware/setup with software you can version-control.
