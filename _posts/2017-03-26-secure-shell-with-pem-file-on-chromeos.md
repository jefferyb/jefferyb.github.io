---
layout: post
title:  "Secure Shell with public key pair or certificate"
desc: "How connect using a public key pair or certificate using Secure Shell on ChromeOS"
keywords: "chromeos,ssh"
date: 2017-03-26
categories: [Linux]
tags: [linux]
icon: fa-code
---

Assuming you're using Secure Shell to SSH to a remote server,
you can import identity files from the connection dialog. 

Select the `"Import..."`, select your private key and public
key from the file picker, for example `"id_rsa"` and `"id_rsa.pub"`, for each identity.

If you have a key stored in a single `".pem"` file, you must split it into two files before importing.

In my case, I had an `openstack_keypair.pem`, an Openstack Nova generated keypair as pem
file. Since I had a private key (openstack_keypair.pem), I needed to
generate a public key (openstack_keypair.pub) for it. To generate the public
key, I did: 

```bash
ssh-keygen -y -f openstack_keypair.pem > openstack_keypair.pub
```

This command created the .pub file that I needed. With that, I renamed the openstack_keypair.pem
file to openstack_keypair and went back to Secure Shell, select the
"Import..." button, selected openstack_keypair and openstack_keypair.pub and
connected

