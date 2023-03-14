---
title: "Secrets as code with Mozilla SOPS"
date: 2023-03-01T00:00:00+00:00
publishDate: 2023-03-01T00:00:00+00:00
image: "/images/code.jpg"
tags: ['ADO', 'pipelines', 'automation', 'SOPS', 'secrets', 'gitops']
comments: false
type: post
category: blog post
author: Anthony Gibbons
authorImage: "/images/anthony.jpg"
---

# Secrets as code with Mozilla SOPS

## The problem

Modern cloud platforms move at speed. The rate of change required to stay competitive in any market requires development teams to ship new features at a high velocity. When releasing new features rapidly, it is often difficult to maintain and control the configuration and secrets that are required to make your application work. Part of this problem has been solved by moving configuration to code repositories therefore leveraging all of the advantages of git. However, secrets are often a pain point. Secrets can be difficult to maintain in a controlled and secure manner. Who should see the secret before it is stored? Where do you store the secret securely? How can you retrieve the secret for use in CI/CD? All of these can be cumbersome issues to resolve.

## We can store secrets in our git repositories

It's true. We can store *all* of our secrets in our git repositories. By doing so, we get all of the many benefits of git:

* Versioning
* Audit trail
* Pull Request control

to mention a few. But how do we do this in a secure manner? We use an open source tool called [Mozilla SOPS](https://github.com/mozilla/sops). SOPS stands for `Secret OPerationS`. It is is an editor of encrypted files that supports YAML, JSON, ENV, INI and BINARY formats and encrypts with AWS KMS, GCP KMS, Azure Key Vault, age, and PGP.

## So what does this look like?

Now that we can store our secrets in git, we can store them alongside the config for our components. An example could look like this:

![Secrets in git](/images/sops-1.png)

Here, we have the config and the secrets for our workload stored and versioned together. The workload here will read the secrets in from a `.env` file which is one of the formats supported by SOPS. 

The secret file looks like this:

![Secrets file](/images/sops-2.png)

We can see that the key:value pair is `apiKey` and the value is completely obfuscated. There is also metadata in the file that SOPS uses to understand how to decrypt the value. In this example, SOPS is backed off to an Azure Key Vault the details of which are stored in the metadata. 

## The advantage of using git

Now that we have our secret stored in a repository, we get all of the benefits of git for free. This screenshot shows what an audit trail can look like just by viewing the commit history:

![Audit trail](/images/sops-3.png)

We can see who committed a change to the file and when it was carried out. Also, we can leverage branch protection policies to ensure that proper review is carried out before merging to add/delete/update secrets.

## How does it work?

What makes this approach secure and easily consumable is the difference between encryption and decryption. Anyone with the public key can *encrypt* secrets but only those with access to the private key can *decrypt* the secret file. 

Since the encryption feature is safely available to all, the creation of secrets is now in the hands of developers, release managers and other personas. A simple command line operation will encrypt the file on disk, ready to be pushed to the repository:

`sops --encrypt --in-place ./secrets/.env`

SOPS will read the information from a `.sops.yaml` file in the repository to understand which system to use. An example of this file for Azure Key Vault:

```yaml
creation_rules:
  - azure_keyvault: https://my-kv.vault.azure.net/keys/sops/8773d4dd9cb1426fa9561726377188
```

This can be configured for other methods of encryption.

The person who creates the secret to begin with can encrypt it and make it available to the wider platform in a safe and secure manner. No more passing secrets around in plain text before they reach a key vault.

Typically, only your CI/CD process would have access to the private key in order to decrypt secrets just-in-time in automation. In our example, we are backing off to Azure Key Vault. Access control over the Key Vault is controlled by native Azure RBAC roles. Our CI/CD service principal has the `Key Vault Secrets Officer` role assigned and is therefore able to access the private key in order to decrypt secrets. Since we are using Azure roles to control access to the resources, it follows that is would be possible to use Azure Privileged Identity Management (PIM) to elevate permissions for troubleshooting any issues with secrets.

## Further information

* [Mozilla SOPS GitHub repo](https://github.com/mozilla/sops)
* [Demo video](https://www.youtube.com/watch?v=V2PRhxphH2w)