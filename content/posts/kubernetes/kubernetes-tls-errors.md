+++
author = "Lewis Green"
title = "Kubernetes TLS error"
date = "2023-05-07"
description = "When you get TLS errors in Kubernetes"
tags = [
    "kubernetes",
    "tls",
    "csr"
]
+++

# Kubernetes TLS errors

When running ```kubectl get logs -n kube-system kube-scheduler-geb``` I got TLS errors. You need to ensure the certificate requests are approved in Kubernetes.


```bash
kubectl get csr
```

Check if any pending

```bash
kubectl certificate approve csr-abcdefg
```