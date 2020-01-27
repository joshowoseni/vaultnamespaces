# NAMESPACES IN VAULT

Create namespace dedicated to organizations (Security and Digital), where they can perform all necessary tasks within their tenant namespace.

Each namespace can have its own:
- Policies
- Auth Methods
- Secrets Engines
- Tokens
- Identity entities and groups

Login to Vault as root
export VAULT_ADDR=http://IPADDRESS
vault login root <Token>

# Step 1: Create namespaces
$ vault namespace create security
$ vault namespace create -namespace=security digital

`To check`
$ vault namespace list
security/

$ vault namespace list -namespace=security
digital/

# Step 2: Write Policy; There is org-level admin for security org and team-level admin for both digital and operations

`Policy for security org admin`
- create sec-admin.hcl
- create digital-admin.hcl

$ vault policy write -namespace=security sec-admin sec-admin.hcl
$ vault policy write -namespace=security/digital digital-admin digital-admin.hcl

# Step 3: Setup entities and groups

`First, you need to enable userpass auth method`
$ vault auth enable -namespace=security userpass

`Create a user 'bob'`
$ vault write -namespace=security auth/userpass/users/bob password="welcome1"

`Create an entity for Bob Smith with 'edu-admin' policy attached`
`Save the generated entity ID in entity_id.txt file`
$ vault write -namespace=security -format=json identity/entity name="Bob Smith" policies="edu-admin"
$ touch entity_id.txt; copy value of '.data_id'

`Get the mount accessor for userpass auth method and save it in accessor.txt file`
$ vault auth list -namespace=security -format=json
$ touch accessor.txt; add vaule of '.["userpass/"].accessor'

`Create an entity alias for Bob Smith to attach 'bob'`
$ vault write -namespace=security identity/entity-alias name="bob" canonical_id=$(cat entity_id.txt) mount_accessor=$(cat accessor.txt)

`Create a group, "digital Admin" in security/digital namespace with Bob Smith entity as its member`
$ vault write -namespace=security/digital identity/group name="digital Admin" policies="digital-admin" member_entity_ids=$(cat entity_id.txt)


# Step 4: Test the Bob Smith entity

(Persona: org-admin)

$ vault login -namespace=security -method=userpass username="bob" password="welcome1"

`To test bob can create namespaces`
`Set the target namespace as an env variable`
$ export VAULT_NAMESPACE="security"

`Create a new namespace called 'web-app'`
$ vault namespace create web-app
Success! Namespace created at: security/web-app/

`Enable key/value v2 secrets engine at edu-secret`
$ vault secrets enable -path=sec-secret kv-v2
Success! Enabled the kv-v2 secrets engine at: sec-secret/

FYI, unset VAULT_NAMESPACE variable once testing is done
$ unset VAULT_NAMESPACE

`To test bob can perform task in digital namespace`
`Set the target namespace as an env variable`
$ export VAULT_NAMESPACE="security/digital"

`Create a new namespace called 'vault-digital'`
$ vault namespace create vault-digital

Key     Value
---     -----
id      nQjeG
path    security/digital/vault-digital/


`Enable key/value v1 secrets digine at team-secret`
$ vault secrets enable -path=team-secret -version=1 kv
Success! Enabled the kv secrets engine at: team-secret/

$ unset VAULT_NAMESPACE