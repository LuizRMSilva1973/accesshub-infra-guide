# 03 - Operação do Vault

## Variáveis de ambiente
- **Clientes via VPN**: `export VAULT_ADDR="https://<ACCESSHUB_DOMAIN>/vault"`  
  O DNS interno resolve o hostname para `<VPN_GATEWAY_IP>` e o NGINX entrega o certificado TLS — não é necessário `VAULT_SKIP_VERIFY`.
- **Operações locais no servidor**: `export VAULT_ADDR="https://127.0.0.1:8200"` e `export VAULT_SKIP_VERIFY=true` (ou aponte `VAULT_CACERT=/etc/ssl/vault/vault.crt` se quiser validar o certificado interno).

Quando precisar de privilégios, exporte temporariamente `VAULT_TOKEN` com um token root/admin (nunca salve em arquivo).

> UI Web: `https://<ACCESSHUB_DOMAIN>/vault/` (via VPN). Fora ela o acesso é bloqueado pelo NGINX. A interface interna (`https://<VPN_GATEWAY_IP>/vault/`) continua funcional, mas mostrará alerta de hostname caso use IP.

## Motores habilitados
- `secret/` → KV v2 para segredos gerais (`vault kv put/get secret/<path>`)
- `database/` → credenciais dinâmicas MySQL (Guacamole)
- `ssh/` → pronto para emitir OTP/CA (opcional)
- `approle/` → acesso não interativo (Guacamole e perfis readonly)
- `file/` audit → `/var/log/vault/audit.log` (coletado pelo Loki)

### Policies relevantes
- `admin` → acesso total (use para tokens operacionais long lived)
- `guacamole` → `secret/guacamole/*` + `database/creds/*`
- `dev-readonly` → leitura dos segredos `secret/dev/*` + `database/creds/dev-readonly`

Listar: `vault policy list` / `vault policy read <policy>`

## Segredos KV (exemplo)
```
vault kv put secret/guacamole/db host=127.0.0.1 port=3306 database=guacamole_db
vault kv get secret/guacamole/db
```

## Credenciais dinâmicas de banco
```
# Guacamole (aplicação)
vault read database/creds/guacamole-service

# Perfil de desenvolvedor read-only
vault read database/creds/dev-readonly
```
- TTL padrão: 1h / Max TTL: 24h.
- Revogação manual: `vault lease revoke <lease_id>` ou `vault write -f database/rotate-root/guacamole-db`.

## AppRole
```
# Guacamole
vault read -field=role_id auth/approle/role/guacamole/role-id
vault write -f -field=secret_id auth/approle/role/guacamole/secret-id

# Dev readonly (opcional)
vault read -field=role_id auth/approle/role/dev-readonly/role-id
vault write -f -field=secret_id auth/approle/role/dev-readonly/secret-id
```
Configure os serviços para usar os pares RoleID/SecretID. O script `guac-vault.sh` já consome o AppRole `guacamole`.

## Auditoria
- `vault audit list -detailed`
- Arquivo: `/var/log/vault/audit.log` (permissão 0640, dono `vault:vault`).
- Todos os acessos aparecem no dashboard Grafana.

## Unseal e tokens (emergências)
1. Em parada do serviço, execute `vault operator unseal` três vezes (chaves 1-3).
2. Para tokens operacionais de longa duração: `vault token create -policy=admin -period=720h`.
3. Scripts úteis em `/root/vault/scripts/` (`init_unseal.sh`, `configure_vault.sh`).
