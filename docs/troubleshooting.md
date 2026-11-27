# Troubleshooting

**"PyATS and Genie are required to send commands to running devices."**

The MCP server was installed without the `pyats` extra. Use the `--from cml-mcp[pyats]` args from [`mcp.example.json`](../mcp.example.json), then restart the agent.

**Bridge mode: devices get an IP but can't reach anything, and `host` never gets an IP**

VMware Workstation on a Linux host drops frames addressed to the lab nodes unless the CML VM can use promiscuous mode, which needs read/write access to the bridged network's device, `/dev/vmnet0`. DHCP half works: IOS asks for a broadcast reply, which gets through, but Ubuntu asks for a unicast reply, which doesn't.

Start the CML VM with [`cml-up.sh`](../cml-up.sh) instead of from VMware. It gives your group that access, then powers the VM on. The access lasts until your machine reboots and VMware resets it. To drop it sooner, power the CML VM off and run `sudo chmod g-rw /dev/vmnet0`.

- If the CML VM is already running, shut it down completely first. It only picks up the access at power-on, so restarting it, from inside CML or from VMware, isn't enough.
- If your bridged network isn't `vmnet0` (check VMware's Virtual Network Editor), change `VMNET` in the script.
- If `host` booted without network, its cloud-init failed too (see below).
- Bridging a VM over Wi-Fi usually doesn't work at all. Use NAT mode and work from `host` instead.

**`host`: `cloud-init status` ends in `status: error`**

Usually `host` had no network on its first boot, so it couldn't install its packages. `/var/log/cloud-init-output.log` shows the failing step. Fix the network first, then re-run cloud-init from `host`'s console:

```bash
sed -i '/source ~\/venv\/bin\/activate/d' ~/.bashrc
sudo cloud-init clean --logs --reboot
```

The `sed` removes the venv line the failed run added to `~/.bashrc`, because cloud-init adds it again. `host` reboots and installs its packages, which takes several minutes.

**Device config is gone after stopping the lab**

The config wasn't saved. The skill runs `write memory`; do the same after changing the config by hand.

**SSH: "no matching key exchange method found" (OpenSSH) or "Incompatible ssh peer (no acceptable kex algorithm)" (Paramiko, Netmiko)**

IOSv and IOSvL2 run IOS 15.x, which only offers SHA-1 key exchange and `ssh-rsa` host keys. Modern clients disable these algorithms or have removed them.

- **OpenSSH:** enable them per connection, or in `~/.ssh/config` for the lab's IPs:

  ```bash
  ssh -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa admin@<device-ip>
  ```

- **Paramiko, Netmiko:** Paramiko 5.x removed these algorithms, so no setting brings them back. Use Paramiko 4.x, which `host`'s virtualenv already has. Netmiko connects through Paramiko, so the same applies.
