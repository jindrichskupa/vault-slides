---
theme: "beige"
trasition: "zoom"
highlightTheme: "Rainbow"
---

# .env is dead long live the vault

## Use your secrets safely

---

# How we use the secrets

---

<img src="/its-a-secret.jpeg"  width="700px">

---

### `mysql.php`

```php
<?php
$servername = "localhost";
$username = "username";
$password = "password";

// Create connection
$conn = new mysqli($servername, $username, $password);

// Check connection
if ($conn->connect_error) {
  die("Connection failed: " . $conn->connect_error);
}
echo "Connected successfully";
?>
```

---

### `config.php`

```php
<?php
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );

/** Database username */
define( 'DB_USER', 'wordpress' );

/** Database password */
define( 'DB_PASSWORD', 'Password123.' );

/** Database hostname */
define( 'DB_HOST', 'localhost' );
```

---

### `.env`

```bash
DB_NAME="wordpress"
DB_USER="wordpress"
DB_PASSWORD="Password123."
DB_HOST="localhost"
```

---

### docker

```bash
docker run -it --rm \
    --volumes-from some-wordpress \
    --network container:some-wordpress \
    -e WORDPRESS_DB_USER=wordpress \
    -e WORDPRESS_DB_PASSWORD=Password123. \
    # [and other used environment variables]
    wordpress:cli user list
```

---

### `passwords.txt`

```txt
wp1: wordpress / Password123.
wp2: wordpress / Password321.
```

---

### git

```bash
git add cert.pem cert.key
git commit -am "Add some certificates"
git push
```

---

### kubernetes

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: envs
  namespace: dev
data:
  USERNAME: QWhvakFob2pBaG9qQWhvago=
  PASSWORD: QWhvakFob2pBaG9qQWhvago=
  SECRET: QWhvakFob2pBaG9qQWhvago=
type: Opaque

```
---
base64 is not encryption

---

<img src="/one-password.jpeg"  width="700px">

---

## What is wrong with you?

* Plain text
* Visible (Everybody can read it)
* Durable (I can steal it and use it later)
* Hard to change (update, distribute, restart)

---

# Hashicorp Vault

---

## Vault to the rescue

* Service (REST API)
* Strong authentication (different methods)
* Role based
* Time limited & one time
* Not only secrets store

---

<img src="/vault-triangle.png">

---

## Use cases

* Automated PKI Infrastructure
* Data Encryption
* Database Credential Rotation
* Dynamic Secrets
* Identity-based Access
* Key Management
* Kubernetes Secrets
* Secrets Management

---

## Vault setup

* Storage
* Seal/unseal
* Root token
* Auth methods
* Users/roles

---

# Show me something!

---

## Vault start

* start as docker with zero config

```bash
docker run -it --name vault -p 8080:8200 vault:1.9.3
```

* save **Unseal key** and **Root token**
* run new terminal

```bash
export VAULT_TOKEN="s.something"
export VAULT_ADDR="http://localhost:8080"
# export VAULT_FORMAT="json"
open http://localhost:8080
```

---

## Secrets

* Key/Value
* Transit
* Databases
* PKI
* SSH
* AWS / Google Cloud / Azure
* https://www.vaultproject.io/docs/secrets

---

## Key/Value storage

* v1 - path + key/value
* v2 - as v1 plus versions

```bash
vault secrets enable -path=kv2/prj kv-v2
vault kv put kv2/prj/dev/envs foo=bar bar=foo
vault kv get kv2/prj/dev/envs
vault kv patch kv2/prj/dev/envs key=value
vault kv delete kv2/prj/dev/envs
vault kv destroy -versions=2 kv2/prj/dev/envs
vault kv metadata get kv2/prj/dev/envs
```

* Metadata - how many versions to hold, delete version after

---

## Key/Value storage (curl)

```bash
# v2
curl -s -XGET -H "X-Vault-Token: $VAULT_TOKEN" \
  $VAULT_ADDR/v1/kv2/prj/data/dev/envs | \
  jq .data.data

# v1
curl -s -XGET -H "X-Vault-Token: $VAULT_TOKEN" \
  $VAULT_ADDR/v1/kv2/prj/dev/envs | \
  jq .data.data
```

---

## Transit

* Encryption as a service

```bash
vault secrets enable transit
vault write -f transit/keys/mysecret

vault write transit/encrypt/mysecret \
  plaintext=$(echo -n "Cookie monster eats the children" | \
    base64 -w 0)

export VAULT_FORMAT=json

vault write transit/decrypt/mysecret \
  "ciphertext=vault:v1:1G+I8mQ7sZ45N8PE7GTcs3TpqupWDiLMn8K091z2F1mRJAGeEE6mxDNHBsDoS4S7kQE2iw+3oTs/R+pU" | \
  jq .data.plaintext | \
  tr -d \"\" | base64 -d

unset VAULT_FORMAT
```

---

## SSH

no more authorized_keys, use ssh CA signed keys

```bash
vault secrets enable -path=ssh-signer ssh
vault write ssh-signer/config/ca generate_signing_key=true

curl -o /etc/ssh/ca.pub \
  http://vault:8200/v1/ssh-signer/public_key
```

`/etc/ssh/sshd_config`

```
TrustedUserCAKeys /etc/ssh/ca.pub
```

---

### signer role

```bash
vault write ssh-signer/roles/admin -<<EOH
{
  "algorithm_signer": "rsa-sha2-256",
  "allow_user_certificates": true,
  "allowed_users": "*",
  "allowed_extensions": "permit-pty,permit-port-forwarding",
  "default_extensions": [ { "permit-pty": "" } ],
  "key_type": "ca",
  "default_user": "tunnel",
  "ttl": "30m0s"
}
EOH
```

---

### sign and use the key

```bash
ssh-keygen -t rsa -f $HOME/.ssh/id_rsa_vault \
  -C "vault@example.com"

vault write -field=signed_key ssh-signer/sign/admin \
    public_key=@$HOME/.ssh/id_rsa_vault.pub > signed_cert.pub

ssh -i signed_cert.pub -i ~/.ssh/id_rsa_vault tunnel@jumphost
```

---

## PostgreSQL

```bash
vault secrets enable database

vault write database/config/postgresql \
    plugin_name=postgresql-database-plugin \
    allowed_roles="pgrole" \
    connection_url="postgresql://{{username}}:{{password}}@postgres:5432/postgres?sslmode=disable" \
    username="postgres" \
    password="postgres"

vault write database/roles/pgrole \
    db_name=postgresql \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

---

### read temporary user credentials

```bash
vault read database/creds/pgrole
```

```json
{
  "request_id": "88d270f4-6aba-f6cd-7c14-3267fd77960b",
  "lease_id": "database/creds/pgrole/AOjEuPeaOP2rc1QBL5Yiowfg",
  "lease_duration": 3600,
  "renewable": true,
  "data": {
    "password": "h3o1hy-aJSR0xKI77cFJ",
    "username": "v-root-pgrole-YeqQe755Kx3jwtJq3iuL-1645704090"
  },
  "warnings": null
}
```

---

## PKI

---

### Root CA

```bash
vault secrets enable pki
# 10 years
vault secrets tune -max-lease-ttl=87600h pki

vault write -field=certificate pki/root/generate/internal \
  common_name="example.com" \
  ttl=87600h > CA_cert.crt

vault write pki/config/urls \
  issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
  crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
```

---

### Intermediate CA

```bash
vault secrets enable -path=pki_int pki
# 5 years
vault secrets tune -max-lease-ttl=43800h pki_int

vault write -format=json pki_int/intermediate/generate/internal \
  common_name="example.com Intermediate Authority" \
  | jq -r '.data.csr' > pki_intermediate.csr

vault write -format=json pki/root/sign-intermediate csr=@pki_intermediate.csr \
  format=pem_bundle ttl="43800h" \
  | jq -r '.data.certificate' > intermediate.cert.pem

vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
```

---

### Certificates

```
vault write pki_int/roles/example-dot-com \
     allowed_domains="example.com" \
     allow_subdomains=true \
     max_ttl="720h"

vault write pki_int/issue/example-dot-com common_name="test.example.com" ttl="24h"
```

---

## Policies

* path + operations

```bash
cat << EOF > dev-ro.policy
path "kv2/prj/dev/*" {
    capabilities = ["read", "list"]
}
EOF

vault policy write dev-ro dev-ro.policy
vault policy list
```

---

## Auth methods

* User/pass
* TLS certificates
* JWT/OIDC
* Kubernetes
* AWS
* https://www.vaultproject.io/docs/auth

---
supports roles (combination of policies)

---

## Plugins

* Extend vault with own secrets
* JWT Issuer


---

## Integrations

* Vault cli
* Golang and other SDKs/clients
* DIY: Just call some http
* Vault Agent
* ExternalSecrets (kubernetes)

---

## Examples

---

### kubernetes

```
export SERVICE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

export VAULT_TOKEN=$(curl -XPOST --data '{"role": "app-teas-dev", "jwt": "$SERVICE"}' \
  https://vault.eag.guru/v1/auth/kube/teas/red/login \
  | jq -r '.auth.client_token')

curl -XGET -H "X-Vault-Token: $VAULT_TOKEN" \
  'https://vault.eag.guru/v1/kv2/prj/data/dev/frontend/envs' \
  | jq .data.data
```

---

## Example 2

### gitlab pipeline

```
export VAULT_TOKEN="$(vault write \
  -field=token auth/gitlab-jwt-eag/login \
  role=pipeline-carvago \
  jwt=${CI_JOB_JWT})"

vault kv get -format json kv2/prj/dev/frontend/envs\
  | jq -r '.data.data | to_entries[] | "\(.key)=\(.value)"' \
  > .env
```

---

# Alternatives

* AWS Secret manager
* Azure Vault
* CyberArk
* and others

---

# Q&A

Thanks

---