# CML Automation Lab

An agent skill that builds a small [Cisco Modeling Labs](https://developer.cisco.com/modeling-labs/) lab for practicing network automation on CML-Free: real Cisco IOS devices you can SSH into, plus an Ubuntu automation box with Python (Netmiko, Paramiko) and Ansible ready to use. How you automate them is up to you.

The skill is a portable [`SKILL.md`](SKILL.md), so any agent that supports skills and MCP servers can run it.

## What You Get

```
your LAN ── ext-conn-0 ── unmanaged-switch-0 ─┬─ sw1 (IOSvL2, the default)
                                              ├─ sw2 / r1 ... (optional, up to 4 Cisco devices)
                                              └─ host (Ubuntu automation box)
```

- **Cisco devices:** one IOSvL2 switch by default, or up to four devices of any Cisco IOS type your CML has an image for. Each gets a hostname, an `admin` user, SSH v2 as the only way in on the vty lines, a management IP from DHCP, and a saved config.
- **`host`:** Ubuntu 24.04 with a Python virtualenv (`~/venv`, activated at login) containing Netmiko, Paramiko 4.x and Ansible, which includes the `cisco.ios` collection. Useful when your machine can't reach the lab or runs Windows, as a throwaway place to install things, or to schedule jobs inside the lab.
- **External Connector:** in **bridge mode**, nodes join your LAN and are reachable from your machine. In **NAT mode**, they're reachable only from inside CML, so you work from `host`.

CML-Free runs at most 5 nodes at once. `host` counts towards that and the Unmanaged Switch and External Connector don't, so the lab can hold up to 4 Cisco devices.

## Prerequisites

- CML-Free installed and running, with its web UI reachable from your machine.
- For bridge mode on VMware Workstation with a Linux host: start the CML VM with [`cml-up.sh`](cml-up.sh) (see [Troubleshooting](docs/troubleshooting.md)).
- An agent that supports MCP servers and skills.
- [uv](https://docs.astral.sh/uv/), because the MCP server runs with `uvx`.
- Internet access from the lab, because `host` downloads its Python packages on first boot.

Built with CML 2.10.1 in VMware Workstation on a Linux host, using cml-mcp 0.32.2, the `iosvl2-2020` image and the `ubuntu-24-04-20260307` image.

## Setup

### 1. Connect the CML MCP server

The skill controls CML through [cml-mcp](https://github.com/xorrkaz/cml-mcp), which needs the `pyats` extra to send console commands. Without it, every configuration step fails.

Copy the `mcpServers` block from [`mcp.example.json`](mcp.example.json) into your agent's MCP config and fill in your CML URL and login. The config contains your CML password, so keep it out of git.

Restart the agent and ask it to check CML's status to confirm it's connected.

### 2. Install the skill

Clone this repo, then link or copy it to wherever your agent loads skills from. For example, from the clone: `ln -s "$PWD" <skills-dir>/cml-automation-lab`.

### 3. Set the lab password

```bash
cp .env.example .env
```

Set `LAB_ADMIN_PASSWORD` in `.env`. The skill reads `.env` from the folder you run the agent in, then from this folder, and asks you if neither has it. Your own scripts can read the same file. `.env` is gitignored, so it keeps the password out of the chat and out of git, but not away from the agent, which types it into the device config.

## Configuration

`.env`:

- `LAB_ADMIN_PASSWORD`: password for the `admin` user on the Cisco devices.

MCP server config:

- `CML_URL`: your CML address, for example `https://192.168.1.50`.
- `CML_USERNAME` / `CML_PASSWORD`: your CML web UI login.
- `CML_VERIFY_SSL`: `false` for CML's default self-signed certificate.
- `PYATS_USERNAME` / `PYATS_PASSWORD` (optional): the console login the server uses for every node. The default `cisco`/`cisco` matches `host`.

`cml-up.sh` environment:

- `CML_VMX` (optional): path to the CML VM's `.vmx` file. Defaults to `~/vmware/cisco-modeling-lab/cisco-modeling-lab.vmx`.

Lab logins:

| Node | Username | Password |
|---|---|---|
| Cisco devices | `admin` | `LAB_ADMIN_PASSWORD` |
| `host` | `cisco` | `cisco` |

`host` uses CML's default Ubuntu login because the MCP server logs into every node's console with that one login. In bridge mode, that means a weak SSH login on your LAN. To change it, edit `password:` in the skill's cloud-init and set `PYATS_PASSWORD` to match.

## Usage

Ask your agent to "build me a CML automation lab", or invoke the `cml-automation-lab` skill by name. It asks for the lab title, the Cisco devices and the connector mode, waits for your go-ahead on the plan, then builds, configures and verifies the lab and reports the IP addresses and logins. The first build takes a while: IOSvL2 needs a few minutes to boot, and `host` installs its packages on first boot.

- **Devices, from your machine (bridge mode):** `ssh admin@<device-ip>`. IOSv and IOSvL2 need extra options with modern OpenSSH (see [Troubleshooting](docs/troubleshooting.md)).
- **`host`:** `ssh cisco@<host-ip>` in bridge mode, or its console in the CML web UI in either mode.
- **Start, stop and wipe:** use the lab's controls in the CML web UI, or ask the agent. **Stop** powers the nodes off and keeps the devices' saved config and `host`'s disk. **Wipe** resets every node to a blank state; ask the agent to configure the devices again afterwards.

All IP addresses come from DHCP and can change after a restart. A DHCP reservation on your router keeps them stable.

## Troubleshooting

See [docs/troubleshooting.md](docs/troubleshooting.md) for bridge-mode networking on VMware, cloud-init failures, lost device config and SSH algorithm errors.
