# git using a gpg key to sign commits

In this README, I'll show you how to import and use a gpg signing key to sign your commits with git.

## Importing

First of all, import your key:
```
$ gpg --import your_private.key
```

Find the ID of the key from which you wish to sign your commits with:
```
$ gpg --list-secret-keys --keyid-format LONG
```

Now tell git to sign all your commits using your key from the ID you got from the above command:
```
$ git config --global commit.gpgsign true
$ git config --global user.signingkey AAAAAAAAAAAAAAAA
```

Let's also tell git the location of the SSH key to use for GitHub:
```
$ git config --global core.sshCommand "ssh -i '~/location/for/id_rsa' -F /dev/null"
```

Tell git our name and our e-mail address:
```
$ git config --global user.name "My NAME"
$ git config --global user.email "myemail@address.com"
```

Git is now ready and is now able to sign commits.

## Exporting

Find the key you wish to export:
```
$ gpg --list-secret-keys
```

Then export your key:
```
$ gpg --export-secret-keys 0xAAAAAAAAAAAAAAAA > your_private.key
```

## Exporting public key

Find the key you wish to export:
```
$ gpg --list-secret-keys
```

Then export your key:
```
$ gpg --armor --export 0xAAAAAAAAAAAAAAAA
```

## Bonus: quickly generating a strong PGP key

First let's store our identity informations in a variable:
```
$ primary_uid="Test user <test@test.com>"
```
Obviously, replace `Test user` and `test@test.com` with your own informations.

Let's now generate the primary key using an ECC algorithm with a 521-bits key size:
```
$ gpg --batch --passphrase '' --quick-generate-key "$primary_uid" nistp521 cert never
```
This key is set to never expire.

Get the latest created key ID and store it into a variable:
```
$ last_gpg_key=$(gpg --list-options show-only-fpr-mbox --list-secret-keys | sed -r -n '$!d;s@^([^[:space:]]+).*@\1@g;p')
```

Since the key we created earlier is only able to certify, we'll need to create two keys that will be able to encrypt and sign.  
Those keys are set to expire in 3 years unlike the primary key.
```
$ gpg --batch --passphrase '' --quick-add-key $last_gpg_key nistp521 encrypt 3y
$ gpg --batch --passphrase '' --quick-add-key $last_gpg_key nistp521/ecdsa sign 3y
```

___

Now, let's say that you wanna sign your commits on GitHub without revealing your private e-mail address, you can add a new uid to your previously created key:
``` 
$ gpg --quick-add-uid $last_gpg_key "Another test user (Exemple comment) <anothertest@test.com>"
```
The comment is purely optional.

However, doing so will set this added uid as the primary identity to use when using the key, you'll need to set the identity we firstly created as the primary one:
```
$ gpg --quick-set-primary-uid $last_gpg_key "$primary_uid"
```

## Deleting a key

Get the key id you need to delete using
```
$ gpg --list-secret-keys --keyid-format LONG
```
then delete your key using this command:
```
$ gpg --expert --delete-keys KEY_ID
```