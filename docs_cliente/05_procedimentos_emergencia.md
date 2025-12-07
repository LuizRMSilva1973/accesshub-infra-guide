# 05 - Procedimentos de Emergência

## 1. Vault indisponível
1. Verifique o serviço: `systemctl status vault`.
2. Se estiver selado (status `Sealed: true`):
   ```
   vault operator unseal
   vault operator unseal
   vault operator unseal
   ```
   Use 3 chaves distintas (compartilhadas entre administradores). Nunca armazene-as no servidor.
3. Sem acesso a tokens válidos? Gere um novo admin:
   ```
   export VAULT_TOKEN=<root>
   vault token create -policy=admin -period=720h
   ```
4. Audit trail: consulte `/var/log/vault/audit.log` e o dashboard Grafana para entender o que ocorreu.

## 2. Guacamole sem conectar ao banco
1. Rode manualmente `sudo /usr/local/bin/guac-vault.sh`. Ele gera nova credencial no Vault e reinicia Tomcat/guacd.
2. Consulte `/var/log/guac-vault-sync.log` para verificar se houve erro no AppRole ou ao reiniciar serviços.
3. Se precisar suspender as rotações automáticas: `sudo systemctl stop guac-vault.timer`. Reative depois com `sudo systemctl start guac-vault.timer`.

## 3. WireGuard fora do ar
1. `systemctl status wg-quick@wg0`
2. Reiniciar: `sudo systemctl restart wg-quick@wg0`
3. Conferir firewall: `sudo ufw status numbered` (regras 6,7,8,9 protegem VPN/Guacamole/Grafana).

## 4. Loki/Promtail/Grafana
- Reinício completo:
  ```
  sudo systemctl restart loki promtail grafana-server
  ```
- Checar erros recentes: `journalctl -u promtail -n 50` (permissões em arquivos são o problema mais comum).
- Se Loki não responder, confirme diretório `/var/lib/loki` e espaço em disco.
- Para validar a UI após um incidente, conecte-se à VPN e abra `https://<ACCESSHUB_DOMAIN>/grafana/` (proxy TLS via NGINX). Se o DNS do cliente estiver em cache, desligue/ligue o túnel ou rode `ipconfig /flushdns` (Windows) / `sudo dscacheutil -flushcache` (macOS). Em último caso, use diretamente `http://<VPN_GATEWAY_IP>:3000`.

## 5. Incidentes de segurança
1. Revogue imediatamente o lease do usuário comprometido (`vault lease revoke <lease>`).
2. Gere novos RoleID/SecretID para o AppRole afetado (`vault write -f auth/approle/role/<role>/secret-id`). Atualize `/etc/guacamole/extensions/vault-integration.conf` se for o Guacamole.
3. Atualize as regras do `ufw` conforme necessário (`ufw delete <n>` → `ufw allow ...`).
4. Extraia evidências via Grafana Loki (`AccessHub Observability`) e salve snapshots/dashboards.

## 6. Backup e restauração
- Vault (Raft): `sudo systemctl stop vault && sudo tar czf /root/vault-raft-backup-$(date +%F).tgz /opt/vault/data && sudo systemctl start vault`.
- Loki: copie `/var/lib/loki` antes de upgrades significativos.
- Configurações críticas: `/etc/guacamole`, `/etc/wireguard/wg0.conf`, `/etc/loki/config.yml`, `/etc/promtail/config.yml`, `/usr/local/bin/guac-vault.sh`, `/etc/grafana/provisioning/*`, `/var/lib/grafana/dashboards/*`.

Manter a documentação atualizada no repositório privado garante recuperação rápida.
