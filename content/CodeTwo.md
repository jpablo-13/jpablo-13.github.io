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
![[codeTwo.png]]
The website shows that CodeTwo provides a platform for developers to _write, save and **run** their JavaScript code_.
It also says it's open source, and offers a _Download App_ button. So let's download the project and inspect the code.

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

Now, it would be awesome if we could simply just use a quick javascript reverse shell