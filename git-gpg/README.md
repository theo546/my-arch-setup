# git using a gpg key to sign commits

In this README, I'll show you how to import and use a gpg signing key to sign your commits with git.

## Importing

First of all, import your key:
```
gpg --import your_private.key
```

Find the ID of the key from which you wish to sign your commits with:
```
gpg --list-secret-keys --keyid-format LONG
```

Now tell git to sign all your commits using your key from the ID you got from the above command:
```
git config --global commit.gpgsign true
git config --global user.signingkey AAAAAAAAAAAAAAAA
```

Let's also tell git the location of the SSH key to use for GitHub:
```
git config --global core.sshCommand "ssh -i '~/location/for/id_rsa' -F /dev/null"
```

Tell git our name and our e-mail address:
```
git config --global user.name "My NAME"
git config --global user.email "myemail@address.com"
```

Git is now ready and is now able to sign commits.

## Exporting

Find the key you wish to export:
```
gpg --list-secret-keys
```

Then export your key:
```
gpg --export-secret-keys 0xAAAAAAAAAAAAAAAA > your_private.key
```