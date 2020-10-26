# Ansible Vault
https://docs.ansible.com/ansible/2.9/user_guide/vault.html
https://docs.ansible.com/ansible/latest/user_guide/vault.html

## `--vault-id [[label]@][source]`
Label passwords when storing them (plaintext files, password vaults, etc.). Then reference the label when encrypting and decrypting content. The label, but not the source (e.g. password file path, script, or `prompt`) is shown in plaintext with the encrypted content to help identify which password was used.

A password label can be duplicated in separate password files, so be careful. 

***

### Referencing a password file
Starting files: `passfile_prod`, `sensitiveinfo.txt`

```
$ cat passfile_prod
notsosecret!123A

$ cat sensitiveinfo.txt
This is all my sensitive info!
```

Ansible 2.10 introduces the ability to store multiple passwords in one file with preceding labels, like `passfile` below, but 2.10 is a big jump architecturally from 2.9, so I'll dig into that later.

https://docs.ansible.com/ansible/latest/user_guide/vault.html#storing-passwords-in-files
```
$ cat passfile
dev mydevpass
test mytestpass
qa my qapass
prod notsosecret!123A
```

***

`encrypt`
```
$ ansible-vault encrypt sensitiveinfo.txt --vault-id prod@passfile_prod
Encryption successful

$ cat sensitiveinfo.txt
$ANSIBLE_VAULT;1.2;AES256;prod
66623661623766383364336463616136343631366631343531656264386430663566373230386362
6233656630366337656432633539636436373830396561330a356531646165363463396437646335
38643961643765356631366630646539643333356232373338316235313530376235653730316666
3334636134336637630a663330313635336538326264343538333465393834343564346532376338
64366266396665343230623763636531646161393766626430393530323934386330
```

***

`view`

All of these will successfully show the decrypted file contents while leaving the target file encrypted. By default specifying an incorrect `vault-id` label will work as long as the correct password file is referenced. This behavior can be chaned by setting the environment variable `DEFAULT_VAULT_ID_MATCH=True` or setting `vault_id_match = True` in `ansible.cfg`). Correct lableing during decryption may be required when calling using a script as the password source, and correct labeling may be reqiured in Ansible 2.10 which can handle multiple passwords per file.

```
$ ansible-vault view sensitiveinfo.txt --vault-id prod@passfile_prod
This is all my sensitive info!

$ ansible-vault view sensitiveinfo.txt --vault-id dev@passfile_prod
This is all my sensitive info!

$ ansible-vault view sensitiveinfo.txt --vault-id @passfile_prod
This is all my sensitive info!

$ ansible-vault view sensitiveinfo.txt --vault-id passfile_prod
This is all my sensitive info!

$ cat sensitiveinfo.txt
$ANSIBLE_VAULT;1.2;AES256;prod
66623661623766383364336463616136343631366631343531656264386430663566373230386362
6233656630366337656432633539636436373830396561330a356531646165363463396437646335
38643961643765356631366630646539643333356232373338316235313530376235653730316666
3334636134336637630a663330313635336538326264343538333465393834343564346532376338
64366266396665343230623763636531646161393766626430393530323934386330

$ cat sensitiveinfo.txt
This is all my sensitive info!
```

***

`decrypt`

All of these will successfully decrypt the file in place. Even specifying an incorrect `vault-id` label will work as long as the correct password file is referenced. Correct lableing during decryption may be required when calling using a script as the password source, and correct labeling may be reqiured in Ansible 2.10 which can handle multiple passwords per file.

```
$ ansible-vault decrypt sensitiveinfo.txt --vault-id prod@passfile_prod
Decryption successful

$ ansible-vault decrypt sensitiveinfo.txt --vault-id dev@passfile_prod
Decryption successful

$ ansible-vault decrypt sensitiveinfo.txt --vault-id @passfile_prod
Decryption successful

$ ansible-vault decrypt sensitiveinfo.txt --vault-id passfile_prod
Decryption successful

$ cat sensitiveinfo.txt
This is all my sensitive info!
```

***

`label@script`

Get a vault password via a script. This could fetch or check out a password from an API, central password vault, etc. The script must accept `--vault-id [label]` as an argument to fetch the correct password.

This command...
```
$ ansible-vault create sensitiveinfo.txt --vault-id prod@getpass.py
```

...results in this script call...
```
getpass.py --vault-id prod
```

...which should return the password for ansible-vault to consume.

***

`@prompt`

Prompt for the password when encrypting or decrypting.
```
$ ansible-vault view sensitiveinfo.txt --vault-id @prompt
Vault password (default): notsosecret!123A
This is all my sensitive info!
```

***

### Running a playbook with vault-encrypted data
To run a playbook that contains or references vault-encrypted data the vault password or password file must be referenced, which can be done several ways.
```
$ ansible-playbook -i inventory --vault-id prod@passfile_prod
$ ansible-playbook -i inventory --vault-id prod@prompt
$ ansible-playbook -i inventory --vault-password-file passfile_prod
$ ansible-playbook -i inventory --ask-vault-file
$ ansible-playbook -i inventory --ask-vault-password
```

***

## `create` - Create a new (empty) vault-encrypted file
```
$ ansible-vault create inventory.txt
```

## `edit` - Edit a vault-encrypted file (in place)
```
$ ansible-vault edit inventory.txt
```

## `encrypt_string` Encrypt a string
Encrypt a string in-line (ONLY USE FOR TESTING; string will be present in shell history).
```
$ ansible-vault encrypt_string --vault-id prod@passfile 'this is my string' --name mystring

mystring: !vault |
      $ANSIBLE_VAULT;1.1;AES256;prod
      62313365396662343061393464336163383764373764613633653634306231386433626436623361
      6134333665353966363534333632666535333761666131620a663537646436643839616531643561
      63396265333966386166373632626539326166353965363262633030333630313338646335303630
      3438626666666137650a353638643435666633633964366338633066623234616432373231333331
      6564
```

Pipe a string to `ansible-vault encrypt_string` using `echo`  (ONLY USE FOR TESTING; string will be present in shell history).
```
$ echo -n 'letmein' | ansible-vault encrypt_string --vault-id test@a_password_file --stdin-name 'test_db_password'
```

Prompt for a string to be encrypted.
```
$ ansible-vault encrypt_string --vault-id prod@passfile --stdin-name 'mystring'

Reading plaintext input from stdin. (ctrl-d to end input, twice if your content does not already have a new line)
```

## Running a playbook with encrypted data

Variable File
```
$ echo secretmessagefile.yml
secretmessage: abc123
```

Encrypt File
```
$ ansible-vault encrypt secretmessagefile.yml --vault-id prod@passfile_prod
Encryption successful

$ cat secretmessagefile.yml
$ANSIBLE_VAULT;1.2;AES256;prod
61383236373562326639353065376439656165633432316461303335613534303865343863616661
3434316632653665363665373561336464383264636162610a373664353331363961383661383162
36313061396364393534316665616663623261376538336231636636643964356435633931626565
3831323263613135350a303366363439623734373331366439313137663835643736396362376635
30666136316561666362663531393734326239303738396133616231313263303831
```

Playbook
```yaml
# decrypt_secret_message.yml
- name: Export encrypted data
  hosts: localhost
  vars_files:
    - secretmessagefile.yml
  tasks:
    - name: Echo encrypted data to file
      shell: echo {{ secretmessage }} > notsosecretmessage.txt
```

Playbook attempts with and without vault password
```
$ ansible-playbook decrypt_secret_message.yml
ERROR! Attempting to decrypt but no vault secrets found
```
```
$ ansible-playbook decrypt_secret_message.yml --vault-id prod@passfile_prod

PLAY [Export encrypted data] ***************************************************

TASK [Gathering Facts] *********************************************************
ok: [localhost]

TASK [Echo encrypted data to file] *********************************************
changed: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

```
$ cat notsosecretmessage.txt
abc123
```

## `rekey` - Change password on existing vault-encrypted file(s)
```
$ ansible-vault rekey file1.txt file2.txt file3.txt
```

## `decrypt` - Decrypt file using password file
```
$ ansible-vault decrypt ~/playbooks/decrypt_me.yml --vault-password-file ~/playbooks/vault_pass.txt
Decryption successful
```

## `--ask-vault-password`
Ansible will prompt for vault password.

```
$ ansible-playbook playbook.yml --ask-vault-password
```

## `--ask-vault-file`
Ansible will prompt for vault file path.

```
$ ansible-playbook playbook.yml --ask-vault-file
```

## `--vault-password-file` - Run playbook with a password file reference
```
$ ansible-playbook -i inventory copy_secrets.yml --vault-password-file ~/playbooks/vault_pass.txt

PLAY [Copy secrets file to all hosts] *************************************

TASK [Gathering Facts] ****************************************************
ok: [web1]
ok: [web2]

TASK [Copy secrets file to host] ******************************************
changed: [web2]
changed: [web1]

PLAY RECAP ****************************************************************
web1                       : ok=2    changed=1    unreachable=0    failed=0
web2                       : ok=2    changed=1    unreachable=0    failed=0
```

## `vault_password_file` - View parameter in `/etc/ansible/ansible.cfg`
Setting the `vault_password_file` parameter creates a default location to look for a vault password when encrypting and decrypting data with `ansible-vault` and `ansible-playbook`.

```ini
$ grep -i "vault_password_file" /etc/ansible/ansible.cfg -B2

## If set, configures the path to the Vault password file as an alternative to
# specifying --vault-password-file on the command line.
vault_password_file = /home/thor/playbooks/vault_pass.txt
```

## Run playbook with a password file referenced in `/etc/ansible/ansible.cfg`
```
$ ansible-playbook -i inventory copy_secrets.yml

PLAY [Copy secrets file to all hosts] *************************************

TASK [Gathering Facts] ***************************************************************************
ok: [web2]
ok: [web1]

TASK [Copy secrets file to host] ***************************************************************************
ok: [web1]
ok: [web2]

PLAY RECAP ****************************************************************
web1                       : ok=2    changed=0    unreachable=0    failed=0
web2                       : ok=2    changed=0    unreachable=0    failed=0
```

## `no_log` for censoring sensitive output
Censor the log/verbose output of a specific task during runtime.

Original Playbook
```yaml
# decrypt_secret_message.yml
- name: Export encrypted data
  hosts: localhost
  vars_files:
    - secretmessagefile.yml
  tasks:
    - name: Echo encrypted data to file
      shell: echo {{ secretmessage }} > notsosecretmessage.txt
```

Playbook run with `-v`. Notice decrypted data `abc123` is shown in the verbose output.
```
$ ansible-playbook decrypt_secret_message.yml --vault-id prod@passfile_prod -v

PLAY [Export encrypted data] ***************************************************

TASK [Gathering Facts] *********************************************************
ok: [localhost]

TASK [Echo encrypted data to file] *********************************************
changed: [localhost] => {"changed": true, "cmd": "echo abc123 > notsosecretmessage.txt", "delta": "0:00:00.005562", "end": "2020-10-22 11:51:21.948004", "rc": 0, "start": "2020-10-22 11:51:21.942442", "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}

PLAY RECAP *********************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Updated Playbook with `no_log: true`
```yaml
# decrypt_secret_message.yml
- name: Export encrypted data
  hosts: localhost
  vars_files:
    - secretmessagefile.yml
  tasks:
    - name: Echo encrypted data to file
      shell: echo {{ secretmessage }} > notsosecretmessage.txt
      no_log: true
```

Playbook run with `-v`. Notice the "output has been hidden" message in the verbose output.
```
$ ansible-playbook decrypt_secret_message.yml --vault-id prod@passfile_prod -v

PLAY [Export encrypted data] ***************************************************

TASK [Gathering Facts] *********************************************************
ok: [localhost]

TASK [Echo encrypted data to file] *********************************************
changed: [localhost] => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": true}

PLAY RECAP *********************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```
## Default cipher
AES256