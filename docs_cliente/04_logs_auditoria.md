# 04 - Logs e Auditoria (SIEM leve)

## Stack instalada
- **Loki 3.6.2** (`/etc/loki/config.yml`, porta 3100)
- **Promtail 3.6.2** (`/etc/promtail/config.yml`)
- **Grafana 12.3.0** (via `https://<ACCESSHUB_DOMAIN>/grafana/`, somente VPN; porta 3000 permanece liberada internamente para troubleshooting)
- Datasource `Loki` e dashboard "AccessHub Observability" provisionados automaticamente.

## Fontes coletadas
| Serviço      | Caminho                                                   | Labels principais |
|--------------|-----------------------------------------------------------|-------------------|
| Syslog       | `/var/log/syslog`                                         | `job=syslog`      |
| Vault        | `/var/log/vault/audit.log`                                | `job=vault_audit` |
| Guacamole    | `/var/log/guacamole/tomcat9-catalina.out`, `guacd.log`, `vault-sync.log` | `job=guacamole`   |
| WireGuard    | `/var/log/wireguard/wg0.log`                              | `job=wireguard`   |
| NGINX        | `/var/log/nginx/*.log`                                    | `job=nginx`       |

Os diretórios `/var/log/guacamole` e `/var/log/wireguard` são preenchidos via `rsyslog` (arquivo `/etc/rsyslog.d/40-guacamole-wireguard.conf`).

## Acesso ao Grafana
1. Conecte-se à VPN (o DNS interno garante que `<ACCESSHUB_DOMAIN>` resolva para `<VPN_GATEWAY_IP>`).
2. Abra `https://<ACCESSHUB_DOMAIN>/grafana/` e autentique-se (usuário padrão `admin`/`<senha configurada na 1ª execução>`). Em caso de manutenção, `http://<VPN_GATEWAY_IP>:3000` continua acessível, porém sem TLS.
3. Dashboard inicial: **AccessHub Observability** com os seguintes painéis:
   - Auditoria Vault/Guacamole (rate de eventos por serviço).
   - Conexões WireGuard por minuto.
   - Erros/alertas agregados (syslog/nginx/guacamole).
   - Tabela de eventos recentes (filtro `vault_audit|guacamole|wireguard`).
4. Datasource Loki (`uid=loki-local`) já está provisionado; basta criar queries adicionais.

## Testes rápidos
```
# Consultar logs via API (a partir do servidor)
curl -G --data-urlencode 'query={job="guacamole"}' http://127.0.0.1:3100/loki/api/v1/query_range

# Usando logcli (opcional)
logcli query '{job="vault_audit"}' --since=1h
```
> O acesso às portas 3000/3100 só é liberado para 10.8.0.0/24 (VPN) e loopback.

## Manutenção
- `sudo systemctl status loki promtail grafana-server`
- Logs de serviço em `journalctl -u loki/promtail/grafana-server`.
- Posicionamento do Promtail: `/var/lib/promtail/positions.yaml`.
- Tuning de retenção: ajuste `schema_config`/`compactor` no `/etc/loki/config.yml` e execute `sudo systemctl restart loki promtail grafana-server`.
