# Connecting the QEMU host `.1` to the Alpine VM `.2`

## Roles

- Host `192.168.100.1`: PPP client; initiates SSH and `pppd`.
- Alpine VM `192.168.100.2`: SSH server and PPP peer.
- Link addresses: host `10.77.0.1`, VM `.2` `10.77.0.2`; MTU/MRU 1400.

The host does not need to run `sshd`; it only uses the SSH client. The local `pppd` creates a route to the peer without installing a default route or DNS. SSH authenticates the host; the root key on the VM is restricted to the `.1` source and a fixed wrapper.

## 1. Preparing PPP on the host

On the Devuan host, ensure that `pppd`, `ssh`, and `/dev/ppp` exist. If the device is missing:

```sh
sudo modprobe ppp_generic
test -c /dev/ppp || sudo mknod /dev/ppp c 108 0
sudo chmod 600 /dev/ppp
ls -l /dev/ppp
```

Also, verify that the host does not have an existing route that conflicts with `10.77.0.1` or `10.77.0.2`:

```sh
ip route get 10.77.0.2
```

If the result points to an already existing interface/route, choose a different private IPv4 pair and adjust both this profile and the wrapper in step 4.

## 2. Create an SSH key on the host and verify the VM's host key

On the host, create the private key as root:

```sh
sudo install -d -m 700 /root/.ssh
sudo ssh-keygen -q -t ed25519 -N '' -C pppsh-host-to-alpine2 \
-f /root/.ssh/pppsh-host-to-alpine2
```

On the VM (`.2`), as root, read the server's fingerprint:

```sh
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

On the host, collect the VM's SSH public key and compare the fingerprint before trusting it:

```sh
sudo sh -c 'ssh-keyscan -t ed25519 192.168.100.2 > /root/.ssh/pppsh-host-known_hosts'
sudo ssh-keygen -lf /root/.ssh/pppsh-host-known_hosts
sudo chmod 600 /root/.ssh/pppsh-host-known_hosts
```

The fingerprints must match. Transfer only the public key and the wrapper to the staging Alpine account:

```sh
cd ~/Projects/pppsh
sudo scp \
-o UserKnownHostsFile=/root/.ssh/pppsh-host-known_hosts \
-o StrictHostKeyChecking=yes \
/root/.ssh/pppsh-host-to-alpine2.pub \
lab/pppsh-remote-host \
alpine@192.168.100.2:/tmp/
```

`scp` will prompt for the Alpine account password. The host key has already been verified above; do not accept a different fingerprint without investigating it.

## 3. Prepare the `.2` VM as a PPP server

Log in to the `.2` VM, switch to the root user, and prepare the PPP device:

```sh
modprobe ppp_generic
test -e /dev/ppp || mknod /dev/ppp c 108 0
chmod 600 /dev/ppp
```

Install the prepared wrapper and add the public key with restrictions:

```sh
install -m 0700 /tmp/pppsh-remote-host /usr/sbin/pppsh-remote-host
install -d -m 700 /root/.ssh
touch /root/.ssh/authorized_keys
sed -i '/pppsh-host-to-alpine2/d' /root/.ssh/authorized_keys
sed 's|^|from="192.168.100.1",restrict,command="/usr/sbin/pppsh-remote-host" |'
``` \
/tmp/pppsh-host-to-alpine2.pub >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
rm -f /tmp/pppsh-host-to-alpine2.pub /tmp/pppsh-remote-host
```

Confirm that the root user can authenticate via key, without a password:

```sh
sshd -T | grep -E '^(permitrootlogin|pubkeyauthentication) '
```

The expected output is `permitrootlogin prohibit-password` and `pubkeyauthentication yes`. This is the configuration observed on the `.2` VM. Do not enable password-based login for the root user. The `restrict` option disables PTY and port forwarding for this key; `command=` forces the use of the PPP wrapper only. [OpenSSH key restrictions](https://man.openbsd.org/sshd.8)

## 4. Installation and connection using the CLI

On the host, from the repository root:

```sh
cd ~/Projects/pppsh
sudo install -m 0755 pppsh /usr/local/bin/pppsh
sudo mkdir -p /etc/pppsh/profiles
sudo install -m 0600 profiles/host-alpine2.conf \
/etc/pppsh/profiles/host-alpine2.conf
sudo pppsh connect host-alpine2
```

The command runs in the foreground; press `Ctrl-C` to disconnect. The profile uses the root-only private key created in step 2 and requires the host key that was verified. The CLI executes `pppd` and connects to the wrapper enforced on the server. The `pppd` `pty` carries the SSH stream; the remote wrapper uses `notty`. [`pppd` options](https://manpages.debian.org/trixie/ppp/pppd.8.en.html)

## 5. Verifying traffic and routes

While `pppd` is running, on the host:

```sh
ip address show dev ppp0
ip route
ping -c 3 10.77.0.2
```

On the `.2` VM:

```sh
ip address show dev ppp0
ip route
ping -c 3 10.77.0.1
```

There should be a route only for the PPP peer; do not use `defaultroute`. Traffic reaches both endpoints, but forwarding between the host and the VM's network is not enabled.

Press `Ctrl-C` on the host to terminate. Confirm that `ppp0` has disappeared on both sides and that the routes have returned to their previous state.

## Cleanup

- On the `.2` VM, remove only the line marked `pppsh-host-to-alpine2` from `/root/.ssh/authorized_keys` and delete `/usr/sbin/pppsh-remote-host`.
- On the host, delete `/root/.ssh/pppsh-host-to-alpine2` and its corresponding `.pub` file; remove the entry using `sudo ssh-keygen -R 192.168.100.2 -f /root/.ssh/pppsh-host-known_hosts`.
- If you installed the CLI, remove `/etc/pppsh/profiles/host-alpine2.conf` and `/usr/local/bin/pppsh` if that copy is no longer needed.
- Do not remove `/dev/ppp` or unload [the module].