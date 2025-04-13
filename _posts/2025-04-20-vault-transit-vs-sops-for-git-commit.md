---
layout: single
title:  "Vault Transit vs SOPS if committing ciphertext to git"
categories: [docker]
tags: [docker, vault]
excerpt: "Want to commit (encrypted) secrets? It's really only safe one way." #this is a custom variable meant for a short description to be displayed on home page
toc: true
---
### Summary
This post is written by ChatGPT. As a rule I don't do this. (In an AI-dominated world, what's the value of blogging about interesting use cases when a reader could just ask AI for answers, or a writer could simply generate posts the same way? For me, the answer is that I benefit from the act of writing). 

But I want this post to act as a sidenote reference for [another post]({% post_url 2025-04-20-install-hashicorp-vault-ubuntu %}) I'm writing today. So, from here on out, consider the author ChatGPT:

**Question:** Is it safe to commit encrypted data to git repos using Vault Transit or SOPS?

### ✅ SOPS – Yes, *designed for this*

SOPS was built specifically to enable encrypted secrets to live safely in Git repositories.

- ✅ Each secret is **field-level encrypted** inside a YAML/JSON file.
- ✅ Encryption metadata (KMS key references, etc.) is **embedded** in the file.
- ✅ Files remain readable in structure (e.g., you still see top-level keys).
- ✅ Secrets are decrypted only on-demand (e.g., in CI/CD or developer environments).
- ✅ You can audit, diff, version, and rollback encrypted files safely.

**Conclusion:**  
> 💯 SOPS is safe and intended for use with Git. Just make sure your KMS access is well-restricted.

---

### 🔶 Vault Transit – ⚠️ Safe *only* with careful wrapping

Vault Transit encrypts arbitrary blobs via API — not structured files. So if you:

1. Encrypt a secret value via the Vault API,
2. Store the ciphertext (e.g., base64 string) in a file,
3. Commit that file to Git,

... then it's *technically* safe, but there are caveats.

#### ⚠️ Considerations

- ❌ Vault Transit doesn't embed metadata like SOPS (key reference, version, etc.).
- 🔐 Ciphertexts aren’t easily diffable or auditable in Git.
- 🤕 Rewrapping secrets or rotating keys requires extra scripting or manual steps.

**Conclusion:**  
> 🟡 Vault Transit *can* be used safely with Git, but you need to build your own tooling and conventions around it. SOPS is safer and simpler by default.

---

### 🚦 Final Verdict

| Use Case                          | SOPS ✅ | Vault Transit ⚠️ |
|----------------------------------|--------|------------------|
| Encrypting app secrets in Git    | ✅ Perfect | ⚠️ Possible, but not ideal |
| GitOps / ArgoCD / Flux           | ✅ Common usage | 🔧 Requires additional work |
| Dynamic, ephemeral encryption    | ❌ Not suitable | ✅ Excellent fit |
| Structured YAML/JSON secrets     | ✅ Built for it | ❌ Not native |
| Secrets as encrypted files       | ✅ Yes | ⚠️ Requires manual wrapping |

---
