# pppsh

`pppsh` creates a point-to-point IPv4 link by carrying PPP over an SSH session. SSH authenticates and encrypts the stream; `pppd` configures the PPP interface. The MVP connects the two endpoints only and does not change default routes, DNS, or forwarding.

## Scope and goal

This piece of software was created with the goal of applying knowledge of PPP to build a very simple type of VPN (perhaps one of the easiest to implement), and it's not intended to be a serious implementation. No one should use this software (unless it is for learning purposes), and no one will be held responsible for any losses or security breaches should any serious issues arise.

There is an official document from the Linux Documentation Project on the subject (though it is dated): https://tldp.org/HOWTO/ppp-ssh/


## Install

The client needs `pppd`, an SSH client, the PPP kernel device (`/dev/ppp`), and a profile installed by an administrator:

```sh
mkdir -p /usr/local/bin /etc/pppsh/profiles
install -m 0755 pppsh /usr/local/bin/pppsh
```


## Connect

Run as root because `pppd` must access `/dev/ppp` and configure the interface. The profile, private key, and host-key file must be root-owned; the profile and key must not grant permissions to group or other users:

```sh
pppsh connect NAME
```

The process stays in the foreground. Press `Ctrl-C` to disconnect; the SSH child and PPP interface should exit with it. SSH uses a 10-second connection timeout, periodic keepalives, and strict host-key checking; unknown or changed host keys are refused.
If both hosts are already configured for SSH, a profile (.conf) can be created directly. However, pppsh also provides a tool (`pppsh create`) that can handle this task as well.

## Profile format

Profiles live in `/etc/pppsh/profiles/NAME.conf`. The parser accepts only these literal `key=value` fields; it does not source or evaluate the file as shell code:

```ini
host=ip_host
user=root
identity_file=path_to_ssh_key (ex: /root/.ssh/pppsh_key)
known_hosts_file=path_to_known_hosts_file (ex: /root/.ssh/known_hosts)
local_ip=vpn_local_ip (ex: 10.77.0.2)
peer_ip=vpn_remote_ip (ex: 10.77.0.1)
mtu=1400
admin_user=insert_ssh_username_here
privilege=type_of_privilege (sudo or su)
source_ip=remote_host
```

The SSH key on the server must start a matching remote PPP peer. The interface addresses in the profile and server's forced command must match. Current limits: SSH host is an IPv4 address or DNS name, PPP is IPv4 only, MTU is 576–1500, and the SSH port is the default port 22.

## CLI profile management

Suppose a profile name "test".
The CLI also provides profile inspection and lifecycle commands. Run profile
management as root:

```sh
sudo pppsh profile list
sudo pppsh profile show test
sudo pppsh profile validate test
sudo pppsh profile remove test --local-only
```

Managed profile creation provisions the remote peer over SSH and displays the
Ed25519 host fingerprint. Type `yes` to trust and pin it. This is
trust-on-first-use; compare the fingerprint through another channel when
possible. The CLI asks for the administrative SSH user and privilege method
(`sudo` or `su`) when they are not supplied as options. The client needs `scp`
for the temporary remote installer. Before authentication, it explains which
SSH/SCP or privilege command is running and which account's password may be
requested. The VPN key is installed for remote root with a forced PPP command,
and each profile receives its own wrapper and key marker:

```sh
sudo pppsh profile create test \
  --host 192.168.100.2 --admin-user test_host \
  --local-ip 10.77.0.1 --peer-ip 10.77.0.2 --mtu 1400 \
  --source-ip 192.168.100.1 --privilege su
sudo pppsh profile list
sudo pppsh connect lab-peer
sudo pppsh profile remove lab-peer
```

`profile validate` performs local checks and does not open a remote PPP session.
Use `profile remove NAME --local-only` only when intentionally leaving the
server authorization in place. Profiles in the original `NAME.conf` format
remain usable; their referenced key files are not deleted by local-only removal.
The server must already permit root public-key authentication through
`/root/.ssh/authorized_keys` and provide `/usr/sbin/pppd`; the CLI does not
change SSH daemon policy.

## Background daemon

The daemon is managed directly by `pppsh`; no boot-time
configuration is installed. Run as root:

```sh
sudo pppsh start profile_name
sudo pppsh status profile_name
sudo pppsh logs profile_name
sudo pppsh logs profile_name --follow
sudo pppsh stop profile_name
```

`status=active` means the supervisor is alive and has started the connection
process; check the log for PPP negotiation success. Runtime state is stored in
`/run/pppsh/`, and root-only logs in `/var/log/pppsh/`. Logs append between
starts and are not rotated automatically. After reboot, run `pppsh start`
again.

## Limitations

There are already known limitations to VPNs of this type, but there are also limitations within the software itself.

### Problems with the PPP + SSH combination:

- Possible TCP meltdown
- There are several layers of encapsulation (ex: ETHERNET -> IP -> TCP -> SSH -> PPP -> ...), which increases overhead and reduces performance
- Encapsulation reduces the MTU
- SSH relies on TCP
- PPP hinders scalability
- No NAT Traversal
- Considerable complexity that isn't worth the result

### Problems with this implementation:

- `pppd` is being used without authentication
- The daemon does not check the connection status over time
- There is no routing for subnets
- There is no alternative route or configurable DNS
- IPv4 only
- The program, in its current state, has not been very well tested
- There may be bugs or inaccuracies, given the project's objective