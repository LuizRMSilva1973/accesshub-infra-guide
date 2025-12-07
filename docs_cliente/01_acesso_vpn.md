# 01 - Acesso VPN (WireGuard)

## Pré-requisitos
- WireGuard instalado no dispositivo (Linux: `sudo apt install wireguard`, macOS/Windows: WireGuard Desktop).
- Arquivo de configuração gerado para cada usuário. O exemplo abaixo usa o peer inicial (10.8.0.2/32). Gere novos pares e publique no servidor conforme necessidade.
- Não é necessário editar `hosts`: um `dnsmasq` escutando em `<VPN_GATEWAY_IP>:53` fornece resolução interna (“split horizon”) para `<ACCESSHUB_DOMAIN>` e `vault.<ACCESSHUB_DOMAIN>` sempre que o perfil distribuir `DNS = <VPN_GATEWAY_IP>`.

```
[Interface]
PrivateKey = <chave_privada_cliente>
Address    = 10.8.0.2/32
DNS        = <VPN_GATEWAY_IP>   # resolve hostnames internos

[Peer]
PublicKey  = <WG_SERVER_PUBLIC_KEY>
Endpoint   = <VPN_PUBLIC_IP>    # ex.: 203.0.113.10:51820
AllowedIPs = 0.0.0.0/0, ::/0    # túnel completo: internet sai pela VPN
PersistentKeepalive = 25
```

> **Servidor (wg0)**: escuta em `<VPN_PUBLIC_IP>`, endereço interno `<VPN_GATEWAY_IP>/24`.

### Split tunnel (opcional)
Se preferir enviar somente o tráfego destinado à rede interna pela VPN (deixando a navegação comum fora do túnel), altere a linha `AllowedIPs` para:
```
AllowedIPs = 10.8.0.0/24, 127.0.0.1/32
```
Esse modo mantém o acesso aos serviços internos (Guacamole, Vault, Grafana) enquanto a internet continua usando o gateway local do usuário.

## Passos
1. Copie o arquivo `.conf` para `/etc/wireguard/wg0.conf` (ou importe via app WireGuard).
2. Ative o túnel:
   - Linux: `sudo wg-quick up client`
   - Desktop: clique em **Activate** no perfil.
3. Valide conectividade:
   - `ping <VPN_GATEWAY_IP>`
   - `dig @<VPN_GATEWAY_IP> <ACCESSHUB_DOMAIN> +short` (deve retornar o IP interno)
   - `curl https://<ACCESSHUB_DOMAIN>/guacamole/ -k` ou `https://<ACCESSHUB_DOMAIN>/vault/` (o host já resolve para o gateway VPN)
   - `curl -k https://<ACCESSHUB_DOMAIN>/grafana/login` para confirmar o acesso ao Grafana via proxy
4. Desative quando não estiver em uso: `sudo wg-quick down client`.

## Provisionamento de novos peers
1. Gere par de chaves: `wg genkey | tee client.key | wg pubkey > client.pub`.
2. No servidor (`/etc/wireguard/wg0.conf`), adicione:
   ```
   [Peer]
   PublicKey = <client_pub>
   AllowedIPs = 10.8.0.X/32
   ```
3. Recarregue: `sudo wg syncconf wg0 <(wg-quick strip wg0)` ou `sudo systemctl restart wg-quick@wg0`.
4. Entregue arquivo ao usuário final.

## Debug rápido
- `sudo wg show` → lista peers conectados e handshakes.
- `sudo tail -f /var/log/wireguard/wg0.log` → eventos derivados do rsyslog (também disponíveis no dashboard Grafana "AccessHub Observability").
- DNS interno: `dig @<VPN_GATEWAY_IP> <ACCESSHUB_DOMAIN>` ou `nslookup vault.<ACCESSHUB_DOMAIN> <VPN_GATEWAY_IP>`.
