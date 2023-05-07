+++
author = "Lewis Green"
title = "Notes for using FluxCD"
date = "2023-04-30"
description = "Notes when using FluxCD such as SOPS, GIT"
tags = [
    "kubernetes",
    "fluxcd"
]
+++

# FluxCD

### Useful Links

[FluxCD SOPS](https://fluxcd.io/flux/guides/mozilla-sops/)


## SOPS

We used age to encrypt SOPS secrets. You need to generate a key for encrypting. The easiest way was Ubuntu on WSL. 

Install age

```bash
apt install age
```

Then generate the key with

```bash
age-keygen -o age.agekey

#Public key: age1mswt72dszwuar5v0slf35t2c7y8px63eq2j4xe8e8fjylck8t3xsuqfe6f
```

We need to load the key into Kubernetes manually, it does not go into Git to prevent decryption.

```bash
cat age.agekey |
kubectl create secret generic sops-age \
--namespace=flux-system \
--from-file=age.agekey=/dev/stdin
```

On your Windows PC in C:\Users\lewis.green\AppData\Roaming\sops\age you can create a file keys.txt and put the key in there so SOPS can auto get the private key relating to the public key

```text
# created: 2023-05-01T11:37:50+01:00
# public key: age1mswt72dszwuar5v0slf35t2c7y8px63eq2j4xe8e8fjylck8t3xsuqfe6f
AGE-SECRET-KEY-12345678909000000000000000000000000000000000000000000000000
```

In the root of your FluxCD project create a file called ```.sops.yaml``` and put in it the below

```yaml
---
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: age1mswt72dszwuar5v0slf35t2c7y8px63eq2j4xe8e8fjylck8t3xsuqfe6f

```

To initially encrypt the secrets.yaml files you need to run

```bash
sops --encrypt --in-place .\config\secrets.yaml
```

Make sure to add this to .gitignore

```text
*/**/.decrypted~secrets.yaml
```

Then everytime you work on the secrets it auto changes in VS Code if you have the plugin installed. Link [here](https://marketplace.visualstudio.com/items?itemName=signageos.signageos-vscode-sops)