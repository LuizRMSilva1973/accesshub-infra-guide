# 00 - Arquitetura AccessHub

```
[Clientes VPN]
     |
     v
+-------------+        +-------------------+        +----------------+
| WireGuard   | 4822   | Apache Guacamole  | 8080   | Sistemas Internos|
| (10.8.0.0)  |------->| + Tomcat9 + guacd |------->| RDP/SSH/HTTP    |
+-------------+        +-------------------+        +----------------+
       |                        |
       |                        +--> usa credenciais efêmeras do Vault
       v
+----------------+        +------------------+        +-------------------+
| Loki/Promtail |<------- | Vault (Raft)     |<------>| MySQL Guacamole DB |
| + Grafana SIEM|         | KV + Database    |        | credenciais dinâm. |
+----------------+        +------------------+        +-------------------+
```

## Fluxo principal
1. O cliente estabelece túnel WireGuard e recebe IP 10.8.0.X.
2. Somente endereços 10.8.0.0/24 conseguem abrir o Tomcat (porta 8080) e Grafana (porta 3000).
3. O script `/usr/local/bin/guac-vault.sh` roda a cada 30 minutos via systemd timer:
   - Autentica no Vault usando AppRole dedicado ao Guacamole.
   - Solicita credencial efêmera (`database/creds/guacamole-service`).
   - Atualiza `guacamole.properties` e reinicia Tomcat/guacd com o novo usuário.
4. Vault fornece segredos via engines `kv-v2`, `database` e `ssh`. Todas as ações são auditadas em `/var/log/vault/audit.log` e enviadas para Loki.
5. Promtail coleta logs de Vault, syslog, Guacamole, WireGuard e NGINX, enviando-os ao Loki. Grafana (exposto ao usuário como `https://<ACCESSHUB_DOMAIN>/grafana/` e internamente na porta 3000) mostra dashboards pré-provisionados (pasta AccessHub).
6. O banco MySQL do Guacamole nunca recebe senhas estáticas: usuários efêmeros expiram em 1h (TTL máx. 24h).
7. Um `dnsmasq` interno (`<VPN_GATEWAY_IP>:53`, ex.: 10.8.0.1) resolve `<ACCESSHUB_DOMAIN>` e `vault.<ACCESSHUB_DOMAIN>` diretamente para o gateway VPN sempre que o cliente está na VPN, garantindo que todos os serviços sejam acessados pelo hostname oficial com certificado válido.

## Segurança e Zero Trust
- `ufw` permite 8080/tcp e 3000/tcp **apenas** para 10.8.0.0/24. Todo o restante é bloqueado.
- NGINX e Vault continuam escutando somente em loopback/TLS; acesso externo ocorre via VPN/Guacamole.
- Split-horizon DNS evita exposição indevida do hostname público: fora da VPN o domínio aponta para o IP da VPS (landing page), dentro da VPN resolve para `<VPN_GATEWAY_IP>` e libera Guacamole/Vault/Grafana através do proxy.
- AppRole do Guacamole tem políticas mínimas (`secret/guacamole/*` + `database/creds/*`). O root token **não** está salvo em disco.
- Loki + Promtail centralizam auditoria de Vault e eventos críticos para investigações rápidas.
