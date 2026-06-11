# HashiCorp Vault Cheat Sheet

This README turns Vault into a practical, start-to-finish cheat sheet. It explains what Vault does, how the main pieces fit together, and which commands you use in the usual order when setting up and operating Vault.

Use this as:

- A quick reference for daily Vault commands
- A beginner-friendly walkthrough of the Vault lifecycle
- A compact guide for auth, policies, secrets, and tokens

---

## 1. Why Vault?

![How Vault Helps](images/how-vault-helps.png)

Vault is used to securely store, control, and audit access to secrets such as:

- API keys
- Database passwords
- Cloud credentials
- Certificates
- Application secrets

Without Vault, secrets often end up in:

- Source code
- `.env` files
- Shared documents
- CI/CD variables with weak access control

Vault solves that by providing:

- Centralized secret storage
- Access control through policies
- Authentication methods for users and machines
- Secret leasing and revocation
- Audit logging
- Encryption as a service

In simple terms:

- Applications or users authenticate to Vault
- Vault checks attached policies
- Vault returns only the secrets they are allowed to access

Official HashiCorp reference:

- [Why use Vault](https://developer.hashicorp.com/vault/tutorials/get-started/why-use-vault)

That tutorial is useful if you want the product-level reasoning behind Vault, including:

- Secrets sprawl reduction
- Dynamic and time-boxed access
- Encryption workflows
- Identity-based access
- Cloud-agnostic deployment options

---

## 2. Vault Architecture

![Vault Architecture](images/vault-architecture.png)

Core Vault components:

- `Storage Backend`: Stores Vault data. Examples: integrated storage, Consul.
- `Vault Server`: The main service that receives API and CLI requests.
- `Auth Methods`: How users or apps prove identity. Examples: token, AppRole, Kubernetes, LDAP.
- `Policies`: Rules that define what actions are allowed.
- `Secrets Engines`: Components that store or generate secrets.
- `Audit Devices`: Record who accessed what and when.

Typical request flow:

1. A user or application sends a request to Vault.
2. Vault verifies identity using an auth method.
3. Vault evaluates the attached policy.
4. If allowed, Vault reads or generates the requested secret.
5. Vault returns the result and optionally records it in audit logs.

---

## 3. Token, Policy & Role Flow

![Token Policy Role](images/token-role-policy.png)

These three concepts are central to Vault:

- `Token`: A temporary credential used to access Vault.
- `Policy`: A set of permissions attached to a token.
- `Role`: A reusable definition that helps generate tokens for a user, app, or machine identity.

Simple flow:

1. An identity authenticates to Vault.
2. Vault maps that identity to a role or auth configuration.
3. Vault issues a token.
4. The token carries policies.
5. The policies decide which paths and operations are allowed.

Keep this model in mind:

- Auth answers: "Who are you?"
- Policy answers: "What can you do?"
- Token answers: "What are you using right now to access Vault?"

---

## 4. Prerequisites

Before using Vault, make sure you have:

- Vault installed
- A terminal session
- Permission to run a Vault server locally or access an existing one

Check the installation:

```bash
vault --version
```

Set the Vault server address:

```bash
export VAULT_ADDR="http://127.0.0.1:8200"
```

If TLS is not configured in a local lab setup, you may also see:

```bash
export VAULT_SKIP_VERIFY=true
```

Use `VAULT_SKIP_VERIFY=true` only for local testing or learning environments.

---

## 5. Start Vault Server

For local development, a common way is dev mode:

```bash
vault server -dev
```

What it does:

- Starts Vault quickly
- Auto-unseals Vault
- Creates a root token
- Uses in-memory storage

Important limitation:

- Dev mode is only for learning and testing
- Data is lost after restart

For a more realistic local server:

```bash
vault server -config=vault.hcl
```

Use this when you want:

- Persistent storage
- Manual initialization
- Manual unsealing
- A setup closer to production

---

## 6. Check Vault Status

Use this first whenever you connect to a Vault server:

```bash
vault status
```

This shows:

- Whether Vault is initialized
- Whether Vault is sealed
- Current storage type
- HA status
- Version information

Common meanings:

- `Initialized = false`: Vault still needs `vault operator init`
- `Sealed = true`: Vault must be unsealed before use
- `Sealed = false`: Vault is ready

---

## 7. Initialize Vault

Initialization is required once for a new Vault cluster:

```bash
vault operator init
```

Typical output includes:

- Unseal keys
- Initial root token

Recommended controlled version:

```bash
vault operator init -key-shares=5 -key-threshold=3
```

Meaning:

- `key-shares=5`: Generate 5 unseal key shares
- `key-threshold=3`: Any 3 of them can unseal Vault

Important:

- Save unseal keys securely
- Save the initial root token securely
- Do not leave them in plain text files on shared systems

---

## 8. Unseal Vault

After initialization, Vault is sealed. Unseal it with the required number of keys:

```bash
vault operator unseal
```

You will be prompted for an unseal key. Run it multiple times using different keys until the threshold is reached.

You can also provide the key directly:

```bash
vault operator unseal <unseal_key_1>
vault operator unseal <unseal_key_2>
vault operator unseal <unseal_key_3>
```

Check status again:

```bash
vault status
```

When `Sealed` becomes `false`, Vault is ready.

---

## 9. Login to Vault

Login using a token:

```bash
vault login
```

Or pass the token directly:

```bash
vault login <token>
```

For the initial admin login, use the root token returned during initialization.

You can also export the token:

```bash
export VAULT_TOKEN="<your_token>"
```

Check who you are:

```bash
vault token lookup
```

This shows:

- Token ID
- Policies
- TTL
- Creation time
- Renewable status

---

## 10. Enable a Secrets Engine

Vault stores or generates secrets through secrets engines.

List enabled secrets engines:

```bash
vault secrets list
```

Enable KV version 2 at path `secret/`:

```bash
vault secrets enable -path=secret kv-v2
```

Common secrets engine commands:

```bash
vault secrets enable kv
vault secrets disable secret/
vault secrets tune -max-lease-ttl=24h secret/
```

What these do:

- `enable`: Turns on a secrets engine
- `disable`: Removes the mount
- `tune`: Changes mount settings like TTLs

---

## 11. Store Secrets in KV

Write a secret:

```bash
vault kv put secret/myapp username="admin" password="mypassword"
```

Read a secret:

```bash
vault kv get secret/myapp
```

Read output as structured JSON:

```bash
vault kv get -format=json secret/myapp
```

Update a secret:

```bash
vault kv put secret/myapp username="admin" password="newpassword"
```

Read only one field:

```bash
vault kv get -field=password secret/myapp
```

Delete latest version:

```bash
vault kv delete secret/myapp
```

Destroy a specific version permanently:

```bash
vault kv destroy -versions=1 secret/myapp
```

List keys under a path:

```bash
vault kv list secret/
```

Metadata operations:

```bash
vault kv metadata get secret/myapp
vault kv metadata delete secret/myapp
```

---

## 12. Dynamic Secrets

Dynamic secrets are one of Vault's most important features.

Instead of storing one long-lived password and sharing it everywhere, Vault can generate credentials on demand and expire them automatically after a lease period.

Common dynamic secret use cases:

- Database usernames and passwords
- Cloud credentials
- PKI certificates

Why this matters:

- Credentials are short-lived
- Secret rotation becomes automatic
- Compromised credentials expire quickly
- Different apps can receive different credentials

Basic dynamic secret flow:

1. Enable a dynamic secrets engine
2. Configure the backend connection
3. Create a role describing what Vault may generate
4. Read credentials from that role
5. Vault returns leased credentials with a TTL

Example with the database secrets engine:

Enable the engine:

```bash
vault secrets enable database
```

Configure a database connection:

```bash
vault write database/config/my-postgresql-database \
    plugin_name=postgresql-database-plugin \
    allowed_roles="readonly" \
    connection_url="postgresql://{{username}}:{{password}}@127.0.0.1:5432/postgres?sslmode=disable" \
    username="postgres" \
    password="postgres"
```

Create a role that generates read-only credentials:

```bash
vault write database/roles/readonly \
    db_name=my-postgresql-database \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

Generate dynamic credentials:

```bash
vault read database/creds/readonly
```

Typical output contains:

- Generated username
- Generated password
- Lease ID
- Lease duration
- Renewable status

Lease-related commands:

```bash
vault lease lookup <lease_id>
vault lease renew <lease_id>
vault lease revoke <lease_id>
```

Important difference:

- `KV secrets` are usually stored by you and then read later
- `Dynamic secrets` are created by Vault when requested and expire automatically

If you are teaching Vault or documenting its core value, dynamic secrets should always be included because they are a major reason teams adopt Vault in the first place.

---

## 13. Create and Manage Policies

Policies decide what a token can do.

Example policy file `myapp-policy.hcl`:

```hcl
path "secret/data/myapp" {
  capabilities = ["read"]
}
```

Create the policy:

```bash
vault policy write myapp-policy myapp-policy.hcl
```

List policies:

```bash
vault policy list
```

Read a policy:

```bash
vault policy read myapp-policy
```

Delete a policy:

```bash
vault policy delete myapp-policy
```

Policy capability meanings:

- `create`: Create data
- `read`: Read data
- `update`: Modify data
- `delete`: Delete data
- `list`: List keys
- `sudo`: Elevated action on protected paths

Tip for KV v2:

- Policy paths usually use `secret/data/...` for reads and writes
- Listing often uses `secret/metadata/...`

Example list policy:

```hcl
path "secret/metadata/*" {
  capabilities = ["list"]
}
```

---

## 14. Create Tokens

Tokens are direct credentials for Vault access.

Create a token with a policy:

```bash
vault token create -policy=myapp-policy
```

Create a token with TTL:

```bash
vault token create -policy=myapp-policy -ttl=1h
```

Create an orphan token:

```bash
vault token create -orphan -policy=myapp-policy
```

Look up a token:

```bash
vault token lookup
vault token lookup <token>
```

Renew a token:

```bash
vault token renew
vault token renew <token>
```

Revoke a token:

```bash
vault token revoke <token>
```

Revoke a token and its children:

```bash
vault token revoke -mode=path auth/token/create
```

When to use direct tokens:

- Quick manual testing
- Short-lived admin work
- Temporary automation

When not to rely on them:

- Long-term app authentication
- Large production environments

For production apps, prefer auth methods like AppRole, Kubernetes, or cloud auth.

---

## 15. Enable AppRole Authentication

AppRole is a common machine-to-machine auth method.

Enable AppRole:

```bash
vault auth enable approle
```

List enabled auth methods:

```bash
vault auth list
```

Create an AppRole:

```bash
vault write auth/approle/role/myapp-role \
    token_policies="myapp-policy" \
    token_ttl="1h" \
    token_max_ttl="4h"
```

Read role configuration:

```bash
vault read auth/approle/role/myapp-role
```

Get the `role_id`:

```bash
vault read auth/approle/role/myapp-role/role-id
```

Generate a `secret_id`:

```bash
vault write -f auth/approle/role/myapp-role/secret-id
```

Login using AppRole:

```bash
vault write auth/approle/login role_id="<role_id>" secret_id="<secret_id>"
```

This returns a client token that the application can use.

Why AppRole is useful:

- Good for servers and automation
- Separates role identity from secret material
- Easier to rotate than hardcoded static tokens

---

## 16. Enable Userpass Authentication

For simple username/password-based login:

```bash
vault auth enable userpass
```

Create a user:

```bash
vault write auth/userpass/users/devuser \
    password="devpass" \
    policies="myapp-policy"
```

Login with userpass:

```bash
vault login -method=userpass username=devuser password=devpass
```

Read user details:

```bash
vault read auth/userpass/users/devuser
```

This is useful for:

- Small teams
- Lab environments
- Simple demos

---

## 17. Enable Audit Logging

Audit logs are critical in real Vault usage.

Enable a file audit device:

```bash
vault audit enable file file_path=/tmp/vault_audit.log
```

List audit devices:

```bash
vault audit list
```

Disable an audit device:

```bash
vault audit disable file/
```

Why this matters:

- Tracks access attempts
- Helps incident response
- Supports compliance requirements

---

## 18. Seal Vault Again

If you need to manually lock Vault:

```bash
vault operator seal
```

After sealing:

- Vault stops serving secrets
- Clients must wait until it is unsealed again

Check status:

```bash
vault status
```

---

## 19. Useful Operator Commands

Leader and HA information:

```bash
vault operator members
```

Generate a new root token flow if needed:

```bash
vault operator generate-root -init
vault operator generate-root
vault operator generate-root -decode=<encoded_token>
```

Rotate encryption key:

```bash
vault operator rotate
```

Rekey unseal keys:

```bash
vault operator rekey
```

Cancel rekey:

```bash
vault operator rekey -cancel
```

These are sensitive administrative operations. Use them carefully.

---

## 20. Useful Read Commands

General read helper:

```bash
vault read <path>
```

General write helper:

```bash
vault write <path> key=value
```

Write without fields, forcing generation:

```bash
vault write -f <path>
```

Delete data at a generic path:

```bash
vault delete <path>
```

List supported keys:

```bash
vault list <path>
```

These generic commands are useful when:

- Working with auth backends
- Using dynamic secret engines
- Calling Vault paths directly

---

## 21. Output Formatting

Vault output can be changed depending on your use case.

JSON output:

```bash
vault status -format=json
vault token lookup -format=json
```

Table output:

```bash
vault secrets list -format=table
```

Extract a single field:

```bash
vault read -field=token auth/approle/login
```

This is especially useful for shell scripts and automation.

---

## 22. End-to-End Example Flow

This is the shortest practical learning flow from start to finish.

### Step 1: Start Vault

```bash
vault server -dev
```

### Step 2: Set environment

```bash
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="<dev_root_token>"
```

### Step 3: Check status

```bash
vault status
```

### Step 4: Enable KV v2

```bash
vault secrets enable -path=secret kv-v2
```

### Step 5: Store a secret

```bash
vault kv put secret/myapp username="admin" password="mypassword"
```

### Step 6: Read the secret

```bash
vault kv get secret/myapp
```

### Step 7: Create a read policy

Create `myapp-policy.hcl`:

```hcl
path "secret/data/myapp" {
  capabilities = ["read"]
}
```

Load the policy:

```bash
vault policy write myapp-policy myapp-policy.hcl
```

### Step 8: Create a token with that policy

```bash
vault token create -policy=myapp-policy
```

### Step 9: Test access with that token

```bash
vault login <new_token>
vault kv get secret/myapp
```

If the token only has read permission, it should read successfully but should not be allowed to write new values.

---

## 23. Command Order Cheat Sheet

If you want the full flow in one glance, use this order:

1. `vault --version`
2. `export VAULT_ADDR="http://127.0.0.1:8200"`
3. `vault server -dev` or `vault server -config=vault.hcl`
4. `vault status`
5. `vault operator init`
6. `vault operator unseal`
7. `vault login <token>`
8. `vault secrets list`
9. `vault secrets enable -path=secret kv-v2`
10. `vault kv put secret/myapp key=value`
11. `vault kv get secret/myapp`
12. `vault secrets enable database`
13. `vault write database/config/my-postgresql-database ...`
14. `vault write database/roles/readonly ...`
15. `vault read database/creds/readonly`
16. `vault policy write myapp-policy myapp-policy.hcl`
17. `vault token create -policy=myapp-policy`
18. `vault auth enable approle`
19. `vault write auth/approle/role/myapp-role token_policies="myapp-policy"`
20. `vault read auth/approle/role/myapp-role/role-id`
21. `vault write -f auth/approle/role/myapp-role/secret-id`
22. `vault write auth/approle/login role_id="<role_id>" secret_id="<secret_id>"`
23. `vault audit enable file file_path=/tmp/vault_audit.log`
24. `vault operator seal`

---

## 24. Common Mistakes

- Using dev mode for production
- Forgetting to save unseal keys securely
- Writing policies for the wrong KV v2 path
- Distributing root tokens for normal use
- Hardcoding long-lived tokens in applications
- Running without audit logging in serious environments
- Treating dynamic secrets like static secrets and not respecting lease expiry

---

## 25. Quick Command Reference

### Server

```bash
vault server -dev
vault server -config=vault.hcl
vault status
```

### Init and Unseal

```bash
vault operator init
vault operator unseal
vault operator seal
```

### Login and Tokens

```bash
vault login <token>
vault token create -policy=myapp-policy
vault token lookup
vault token renew
vault token revoke <token>
```

### Secrets Engine

```bash
vault secrets list
vault secrets enable -path=secret kv-v2
vault secrets disable secret/
```

### KV Secrets

```bash
vault kv put secret/myapp key=value
vault kv get secret/myapp
vault kv list secret/
vault kv delete secret/myapp
```

### Dynamic Secrets

```bash
vault secrets enable database
vault write database/config/my-postgresql-database ...
vault write database/roles/readonly ...
vault read database/creds/readonly
vault lease lookup <lease_id>
vault lease renew <lease_id>
vault lease revoke <lease_id>
```

### Policies

```bash
vault policy write myapp-policy myapp-policy.hcl
vault policy read myapp-policy
vault policy list
vault policy delete myapp-policy
```

### Auth Methods

```bash
vault auth list
vault auth enable approle
vault auth enable userpass
```

### AppRole

```bash
vault write auth/approle/role/myapp-role token_policies="myapp-policy"
vault read auth/approle/role/myapp-role/role-id
vault write -f auth/approle/role/myapp-role/secret-id
vault write auth/approle/login role_id="<role_id>" secret_id="<secret_id>"
```

### Audit

```bash
vault audit enable file file_path=/tmp/vault_audit.log
vault audit list
vault audit disable file/
```

---

## 26. Final Summary

Vault usage becomes much easier if you remember this sequence:

1. Start Vault
2. Initialize it
3. Unseal it
4. Login
5. Enable a secrets engine
6. Store static secrets if needed
7. Generate dynamic secrets where possible
8. Create policies
9. Generate tokens or roles
10. Authenticate applications
11. Audit and operate Vault safely

If you are learning Vault for the first time, practice this order repeatedly:

`status -> init -> unseal -> login -> secrets -> policy -> token -> auth -> audit`

For the official conceptual introduction from HashiCorp, read:

- [Why use Vault](https://developer.hashicorp.com/vault/tutorials/get-started/why-use-vault)
