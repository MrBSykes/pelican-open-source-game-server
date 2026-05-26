# pelican-open-source-game-server
Pelican Panel on WSL2 — Home Lab Game Server Setup
Bryan Sykes | SykesHomeServer | Alexandria, VA | May 2026


Overview

This project documents the installation and configuration of a self-hosted game server management panel on a Windows home lab machine using Windows Subsystem for Linux 2 (WSL2). The stack includes Pelican Panel as the web UI, Wings as the container daemon, native Docker Engine, and Pi-hole for network-wide ad blocking.

This was not a plug-and-play setup. The core challenge, a WSL2 kernel limitation that prevents runc from writing io.weight to container cgroup directories required patching runc from source. This repository documents the full process including the root cause, every failed fix, and the definitive solution.


Stack
Service
Details
OS
Windows + WSL2 (Ubuntu 24)
Panel
Pelican Panel 1.0.0-beta34
Daemon
Wings (Pelican)
Container Runtime
Docker Engine (native) + runc 1.3.5 (patched)
Ad Blocker
Pi-hole
Game Servers
Minecraft (Paper) + Palworld



The Core Problem
Every container start/reinstall through Wings failed with:

OCI runtime create failed: runc create failed: unable to start container process:

error during container init: error setting cgroup config for procHooks process:

openat2 /sys/fs/cgroup/docker/[hash]/io.weight: no such file or directory

Root cause: The WSL2 kernel lists io as an available cgroup v2 controller but does not implement the io.weight interface file. runc tries to write to it unconditionally and crashes the container init.


The Fix — Patching runc from Source
# 1. Install build dependencies

sudo apt install golang-go make gcc libseccomp-dev pkg-config -y

# 2. Clone runc source matching your installed version

runc --version  # confirm version first

git clone --depth=1 --branch v1.3.5 https://github.com/opencontainers/runc.git

cd runc

# 3. Apply patch — make io.weight failure non-fatal

python3 -c "

content = open('vendor/github.com/opencontainers/cgroups/fs2/io.go').read()

old = '\"io.weight\", strconv.FormatUint(v, 10)); err != nil {\n\t\t\t\treturn err'

new = '\"io.weight\", strconv.FormatUint(v, 10)); err != nil {\n\t\t\t\tif !os.IsNotExist(err) { return err }'

open('vendor/github.com/opencontainers/cgroups/fs2/io.go', 'w').write(content.replace(old, new))

print('Done' if old in content else 'Pattern not found')

"

# 4. Build and replace

make

sudo cp /usr/bin/runc /usr/bin/runc.backup

sudo cp ~/runc/runc /usr/bin/runc

# 5. Hold runc to prevent apt from overwriting the patch

sudo apt-mark hold runc

# 6. Restart services

sudo systemctl restart docker && sudo systemctl restart wings


Windows Port Forwarding
WSL2 runs on an internal IP that changes on every reboot. Run these in PowerShell as Administrator on the Windows host to expose services on the LAN:

netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=8080 connectaddress=<WSL2_IP>

netsh interface portproxy add v4tov4 listenport=8081 listenaddress=0.0.0.0 connectport=8081 connectaddress=<WSL2_IP>

netsh interface portproxy add v4tov4 listenport=2022 listenaddress=0.0.0.0 connectport=2022 connectaddress=<WSL2_IP>

Get your current WSL2 IP with: hostname -I | awk '{print $1}'

⚠️ These rules must be updated after every Windows reboot as the WSL2 IP changes.


Docker Configuration
Native Docker Engine is required. Docker Desktop's WSL2 integration makes the cgroup problem worse.

/etc/docker/daemon.json:

{

  "default-cgroupns-mode": "host",

  "default-ulimits": {},

  "features": {

    "containerd-snapshotter": true

  },

  "exec-opts": ["native.cgroupdriver=cgroupfs"],

  "log-driver": "json-file"

}


Node Configuration
In Pelican Panel → Admin → Nodes → Basic Settings:

IP Address: 192.168.1.182 (Windows host IP, not WSL2 IP)
Port: 8081
Protocol: HTTP

In Allocations — set bind IP to 0.0.0.0, not 192.168.1.182. WSL2 cannot bind containers directly to the Windows host IP.


Game Servers
Server
Port
Connect Address
Status
Minecraft (Paper)
25565
192.168.1.182:25565
✅ Running
Palworld
8211
192.168.1.182:8211
✅ Running


⚠️ Both games require legitimate owned copies to connect. Palworld on Xbox/GamePass does not support third-party dedicated servers — Steam version required.


Post-Reboot Checklist
Get new WSL2 IP: hostname -I | awk '{print $1}'
Update port forwards in PowerShell (Admin) if IP changed
Start Docker: sudo service docker start
Start Wings: sudo systemctl start wings
Verify panel: http://192.168.1.182:8080
Start game servers from panel console


Known Issues
Issue
Notes
Node health globe shows red
Cosmetic UI bug. Wings is working. Confirmed via API response.
WSL2 IP changes on reboot
Port forwards must be updated manually after each reboot
runc patch lost on update
apt-mark hold runc prevents this (do not skip this step)
No HTTPS
Panel runs over HTTP — SSL requires a domain and certificate



Skills Demonstrated
WSL2 environment configuration and Linux service management
Self-hosted web application deployment (Pelican Panel, Nginx, SQLite)
Docker Engine installation and daemon configuration
Root cause analysis of a kernel-level cgroup v2 limitation
Go source code patching and compilation (runc from source)
Windows networking — netsh portproxy, Windows Firewall rules
Container runtime debugging (runc, OCI runtime errors)
Network-wide DNS ad blocking with Pi-hole
Technical documentation and home lab project write-up


Project Files
File
Description
SykesHomeServer_Pelican_Panel_Documentation.pdf
Full formatted project documentation
Pelican_Panel_Setup_Screenshots.zip
20 labeled screenshots from the setup process
README.md
This file




SykesHomeServer — Home Lab Project | securedbybryan


