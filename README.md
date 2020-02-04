# NAMESPACES IN VAULT

Purpose: This is to create namespace dedicated to organization, App, Business Units where they can perform all necessary tasks within their tenant namespace. These namespaces can have child namespaces
Each namespace can have its own:
- Policies
- Auth Methods
- Secrets Engines
- Tokens
- Identity entities and groups

### Environment prep. We would be using root
Login to Vault as root using the root token
* `export VAULT_ADDR=http://<IPADDRESS>:8200`
* `vault login <Token>`

### Step 1: Create namespaces
*  ### CLI Command
`$ vault namespace create security`
`$ vault namespace create -namespace=security digital`

```
$ vault namespace list
security/
```
```
$ vault namespace list -namespace=security
cloudsecurity/
```
*   ### API call using cURL
```
$ curl --header "X-Vault-Token: <TOKEN>" \
       --request POST \
       http://127.0.0.1:8200/v1/sys/namespaces/security
```
```
# Create a training namespace under security
$ curl --header "X-Vault-Token: <TOKEN>" \
       --header "X-Vault-Namespace: security" \
       --request POST \
       http://127.0.0.1:8200/v1/security/sys/namespaces/cloudsecurity
```


### Step 2: Write Policy; In this case, create policy for org-level admin and team-level admin for both security and cloudsecurity respectively.

- ### CLI Command
Create policy for both org-level admin and team-level admin
- create sec-admin.hcl
- create cloudsec-admin.hcl

`$ vault policy write -namespace=security sec-admin sec-admin.hcl`
`$ vault policy write -namespace=security/cloudsecurity cloudsec-admin cloudsec-admin.hcl`

- ### API call using cURL

```
# Create a request payload
$ tee sec-payload.json <<EOF
{
  "policy": "path \"sys/namespaces/security/*\" {\n  capabilities = [\"create\", \"read\", \"update\", \"delete\", \"list\", \"sudo\"]\n } ... "
}
EOF

# Create edu-admin policy under 'security' namespace
$ curl --header "X-Vault-Token: <Token>" \
       --header "X-Vault-Namespace: security" \
       --request PUT \
       --data @sec-payload.json \
       https://127.0.0.1:8200/v1/sys/policies/acl/sec-admin

# Create a request payload
$ tee cloudsec-payload.json <<EOF
{
 "policy": "path \"sys/namespaces/security/cloudsecurity/*\" {\n  capabilities = [\"create\", \"read\", \"update\", \"delete\", \"list\", \"sudo\"]\n  }  ... "
}
EOF

# Create cloudsec-admin policy under 'security/cloudsecurity' namespace
# This example directs the target namespace in the API endpoint
$ curl --header "X-Vault-Token: <Token>" \
       --request PUT \
       --data @cloudsec-payload.json \
       https://127.0.0.1:8200/v1/security/cloudsecurity/sys/policies/acl/cloudsec-admin
```


### Step 3: Setup entities and groups
-   ### CLI Command
First, enable userpass auth method
- `$vault auth enable -namespace=security userpass`

Create a user `bob` with permission
- `$ vault write -namespace=security auth/userpass/users/bob password="welcome1"`

Create an entity for Bob Smith with 'edu-admin' policy attached & Save the generated entity ID in entity_id.txt file
- `$ vault write -namespace=security -format=json identity/entity name="Bob Smith" policies="sec-admin"`
- `$ touch entity_id.txt; copy value of '.data_id'`

Get the mount accessor for userpass auth method and save it in accessor.txt file
- `$ vault auth list -namespace=security -format=json`
- `touch accessor.txt; add vaule of '.["userpass/"].accessor'`

Create an entity alias for Bob Smith to attach 'bob'
- `$ vault write -namespace=security identity/entity-alias name="bob" canonical_id=$(cat entity_id.txt) mount_accessor=$(cat accessor.txt)`

Create a group, "digital Admin" in security/digital namespace with Bob Smith entity as its member
- `$ vault write -namespace=security/digital identity/group name="digital Admin" policies="digital-admin" member_entity_ids=$(cat entity_id.txt)`


### Step 4: Test the Bob Smith entity

`$ vault login -namespace=security -method=userpass username="bob" password="welcome1"`

Set the target namespace as an env variable`
`$ export VAULT_NAMESPACE="security"`

Create a new namespace called 'web-app'
```
$ vault namespace create web-app
Success! Namespace created at: security/web-app/
```

Enable key/value v2 secrets engine at edu-secret
```
$ vault secrets enable -path=sec-secret kv-v2
Success! Enabled the kv-v2 secrets engine at: sec-secret/
```

Ensure you unset VAULT_NAMESPACE variable once testing is done
`$ unset VAULT_NAMESPACE`

To test bob can perform task in digital namespace
Set the target namespace as an env variable
`$ export VAULT_NAMESPACE="security/digital"`

Create a new namespace called 'vault-digital'
```
$ vault namespace create vault-digital`

Key     Value
---     -----
id      nQjeG
path    security/digital/vault-digital/
```


Enable key/value v1 secrets digine at team-secret
```
$ vault secrets enable -path=team-secret -version=1 kv
Success! Enabled the kv secrets engine at: team-secret/
```

`$ unset VAULT_NAMESPACE`

- ### API Call using cURL
Login as bob

```
$ curl --header "X-Vault-Namespace: security" \
       --request POST \
       --data '{"password": "Welcome1"}' \
       http://127.0.0.1:8200/v1/auth/userpass/login/bob | jq
```

Below is the response. Note the generated bob token 
```
{
  ...
  "auth": {
    "client_token": "s.THNIRijGjnLeL25vcFv5NNon.1Vi61",
    "accessor": "BMQXz9QVym1SZe8zJvBKbOii.1Vi61",
    "policies": [
      "default",
      "sec-admin"
    ],
    "token_policies": [
      "default"
    ],
    "identity_policies": [
      "sec-admin"
    ],
    "metadata": {
      "username": "bob"
    },
    ...
  }
}
```

Create a new namespace called 'web-app'
```
# Be sure to use generated bob's client token
$ curl --header "X-Vault-Token: s.THNIRijGjnLeL25vcFv5NNon.1Vi61" \
       --request POST \
       http://127.0.0.1:8200/v1/security/sys/namespaces/web-app

# Enable key/value v2 secrets engine at edu-secret
$ curl --header "X-Vault-Token: s.THNIRijGjnLeL25vcFv5NNon.1Vi61" \
       --request POST \
       --data '{"type": "kv-v2"}' \
       http://127.0.0.1:8200/v1/security/sys/mounts/sec-secret
```