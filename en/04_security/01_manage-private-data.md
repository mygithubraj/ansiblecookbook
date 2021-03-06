\newpage

# How do I store private data in git for Ansible?

I assume, you use git to manage your ansible projects history.

However, even if you use a private git repo on [GitHub](https://github.com), your data are accessable by GitHub employees and and federal law enforcement).

So how can we make sure, anyone can get access to your data, but the users you want?

Below I show you how this can be done:

## Solution

Requirements: [GnuPG](https://www.gnupg.org)

### Create a password file for ansible-vault

Create a password file for ansible-vault, I use pwgen just because I am lazy. After that, we make sure this plain text password file will never be added to our git repo.

~~~bash
$ pwgen 12 1 > vault-password.txt
$ echo vault-password.txt >> .gitignore
$ git add .gitignore
$ git commit -m "ignore vault-password.txt"
~~~

### Use GnuPG to encrypt the password file

Now we use GnuPG to encrypt the password file used for `ansible-vault`:

~~~bash
$ gpg --recipient 42FF42FF \
      --recipient 12345678 \
      --recipient FEFEFEFE \
      --recipient EFEFEFEF \
      --encrypt-files vault-password.txt
~~~

The above command will create an encrypted version of our password file in a new file vault-password.txt.gpg. As you can see we added a few recipient public keys. All owners of the corresponding private keys, and only them, are able to decrypt the encrypted password file.

*Hint: So you better be one of them.*

### Add the file vault-password.txt.gpg to your git repository

The encrypted password file gets into our repo:

~~~bash
$ git add vault-password.txt.gpg
$ git commit -m "store encrypted password"
~~~

### Decrypt and run playbook

~~~bash
$ gpg --decrypt vault-password.txt.gpg
$ ansible-playbook --vault-password-file vault-password.txt ...
~~~

### Create an alias for daily usage

For the daily usage, I used to use an alias `ansible-playbook-vault`:

~~~bash
$ alias ansible-playbook-vault='test -e vault-password.txt ||
gpg --decrypt-files vault-password.txt.gpg;
ansible-playbook --vault-password-file vault-password.txt $@'
~~~

This alias will automatically decrypt the vault-password.txt.gpg if necessary and uses the encrypted file with option `--vault-password-file`.

## Explanation

Ansible has a tool for encryption of var files like `group_vars` and `host_vars` sind version 1.5, [ansible-vault](http://docs.ansible.com/playbooks_vault.html).

However, with ansible-vault you can only encrypt and decrypt var files. Another downside is you have to remember the password and this password must be shared accross your team members.

That is why `ansible-playbook` has an other option for letting you write the password in a file and pass `--vault-password-file <filename>` to `ansible-playbook`. So you do not have to writing it over and over again every time you run a playbook or also meant for automated execution of ansible.

But as you can guess, the password is stored in plain text in a file and we likely want this file to be in a public git repo.

This is the part where asymetric cryptology and GnuPG comes into the game. GnuPG lets you encrypt any file without shareing a password, even for a group of people.
