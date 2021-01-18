# ssh

Just a reminder on how to setup an SSH host so it can be used with SSHFS or Nautilus:

Open the `~/.ssh/config` file then add:
```
Host example
        HostName example.net
        User root
        Port 2222
        IdentityFile "~/location/for/id_ed25519"
```