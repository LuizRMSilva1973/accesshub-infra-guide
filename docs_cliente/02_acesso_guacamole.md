# 02 - Acesso ao Guacamole

## Fluxo
1. Conecte-se à VPN WireGuard (ver documento 01).
2. Acesse `https://<ACCESSHUB_DOMAIN>/guacamole/` (o DNS interno aponta o hostname para `<VPN_GATEWAY_IP>` e o NGINX entrega o certificado TLS). Caso precise falar direto com o Tomcat, `http://<VPN_GATEWAY_IP>:8080/guacamole/` continua disponível somente dentro da VPN.
3. Autentique-se com o usuário Guacamole (ex.: `guacadmin` ou LDAP/SAML, dependendo da configuração atual).
4. Os desktops/RDP/SSH publicados aparecerão no painel. Clique para abrir.

> Como o WireGuard distribui o DNS interno (`<VPN_GATEWAY_IP>`), o domínio público sempre resolve para a rede privada enquanto o túnel está ativo — nenhuma mensagem de certificado será exibida ao usar `https://<ACCESSHUB_DOMAIN>/guacamole/`. Se for necessário testar pelo IP, o navegador alertará sobre hostname diferente.
>
> A porta 8080 está bloqueada pelo UFW para qualquer origem que **não** seja a rede 10.8.0.0/24. Acesso pela internet é negado por padrão.

## Rotina de credenciais dinâmicas
- O arquivo `/etc/guacamole/guacamole.properties` **não** possui mais senhas estáticas.
- O serviço `guac-vault.timer` executa `/usr/local/bin/guac-vault.sh` a cada 30 minutos:
  - Faz login no Vault via AppRole (`/etc/guacamole/extensions/vault-integration.conf`).
  - Busca um novo usuário efêmero em `database/creds/guacamole-service` (TTL 1h, máximo 24h).
  - Atualiza `mysql-username`/`mysql-password` no `guacamole.properties` e reinicia Tomcat/guacd.
- Log das rotações: `/var/log/guac-vault-sync.log` (também enviado ao Loki).

## Administração e troubleshooting
- Reiniciar serviços manualmente:
  - `sudo systemctl restart tomcat9`
  - `sudo systemctl restart guacd`
- Ver logs em tempo real:
  - `/var/log/guacamole/tomcat9-catalina.out`
  - `/var/log/guacamole/guacd.log`
- O lease vigente fica em `/var/run/guac-vault.lease`. Para revogar manualmente: `vault lease revoke <lease_id>`.
