# Plano de Testes de Homologação – AccessHub / VPN / Guacamole / Vault / Observabilidade  
Versão: 30/11/2025

Este plano descreve os testes necessários para homologar a solução end-to-end, cobrindo:

- DNS público e interno (split-horizon)
- HTTP → HTTPS e certificados TLS
- NGINX (proxy reverso)
- Renovação automática de certificados (Certbot)
- VPN WireGuard (cliente e servidor)
- Guacamole + Tomcat + MySQL
- HashiCorp Vault (UI, CLI, engines KV/Database, AppRole)
- Observabilidade (Grafana + Loki/Promtail)

Para cada teste, registre:

- **Comando executado**
- **Output (copia/print)**
- **Resultado:** ✅ Aprovado / ❌ Reprovado
- **Observações:** (erros, particularidades, prints de tela)

---

## 1. Testes de DNS e Camada Web

### 1.1. DNS público

**Objetivo:** Validar resolução DNS pública para `<ACCESSHUB_DOMAIN>`.

**Local:** estação do cliente (fora da VPN).

**Comando:**

```bash
dig <ACCESSHUB_DOMAIN> +short
````

**Resultado esperado:**

* Retornar o IP público da VPS, ex.: `<VPN_PUBLIC_IP>`.

---

### 1.2. DNS interno (split-horizon)

**Objetivo:** Validar resolução DNS interna via servidor da VPN (<VPN_GATEWAY_IP>).

**Local:** estação com VPN conectada **ou** servidor.

**Comando:**

```bash
dig @<VPN_GATEWAY_IP> <ACCESSHUB_DOMAIN> +short
```

**Resultado esperado:**

* Retornar um IP interno compatível com o design (ex.: `<VPN_GATEWAY_IP>`).

---

### 1.3. Redirecionamento HTTP → HTTPS

**Objetivo:** Garantir que todo acesso HTTP é redirecionado para HTTPS.

**Local:** estação do cliente.

**Comando:**

```bash
curl -I http://<ACCESSHUB_DOMAIN>
```

**Resultado esperado:**

* Status `301 Moved Permanently`.
* Header `Location: https://<ACCESSHUB_DOMAIN>/`.

---

### 1.4. Certificado TLS (cadeia e validade)

**Objetivo:** Verificar se o certificado HTTPS é válido e com cadeia completa.

**Local:** estação do cliente ou servidor.

**Comando:**

```bash
echo | openssl s_client -connect <ACCESSHUB_DOMAIN>:443 -servername <ACCESSHUB_DOMAIN> 2>/dev/null | tail -n 20
```

**Resultado esperado:**

* Presença de `Verify return code: 0 (ok)`.
* Cadeia de certificados corretamente apresentada.

---

### 1.5. Estado do NGINX

**Objetivo:** Garantir que o proxy reverso está saudável e pronto para produção.

**Local:** servidor.

**Comandos:**

```bash
sudo systemctl status nginx
sudo nginx -t
sudo tail -n 50 /var/log/nginx/access.log
sudo tail -n 50 /var/log/nginx/error.log
```

**Resultado esperado:**

* `systemctl status nginx` → `active (running)`.
* `nginx -t` → `syntax is ok` / `test is successful`.
* Em `access.log`, requisições 200/304 para `/grafana/`, `/vault/` e `/guacamole/` originadas de IPs da VPN (ex.: `10.8.0.2`).
* Em `error.log`, acessos externos indevidos (IPv6 / bot) resultando em `access forbidden by rule` (comportamento esperado).

---

### 1.6. Agendamento de renovação (Certbot)

**Objetivo:** Garantir que a renovação automática de certificados está configurada.

**Local:** servidor.

**Comando:**

```bash
systemctl list-timers | grep certbot
```

**Resultado esperado:**

* Timer `snap.certbot.renew.timer` listado com última execução recente e próxima execução agendada.

**Teste opcional (dry-run):**

```bash
sudo /snap/bin/certbot renew --nginx --dry-run
```

**Resultado esperado:**

* Mensagem indicando que todas as renovações de teste foram bem-sucedidas.

---

## 2. Testes da VPN WireGuard

### 2.1. Subir túnel no cliente

**Objetivo:** Validar que o cliente consegue estabelecer conexão WireGuard.

**Local:** estação do cliente.

**Comando:**

```bash
sudo wg-quick up client
```

**Resultado esperado:**

* Sem mensagens de erro.
* Interface `client` visível em `ip a`.

---

### 2.2. Handshake e tráfego

**Objetivo:** Comprovar handshake ativo e fluxo de dados pelo túnel.

**Local:** estação do cliente.

**Comando:**

```bash
sudo wg show
```

**Resultado esperado:**

* `latest handshake` recente (segundos/minutos).
* `transfer: X MiB/GiB received, Y MiB/GiB sent`.
* `endpoint` apontando para `<VPN_PUBLIC_IP>:51820`.
* `allowed ips` conforme política (ex.: `0.0.0.0/0, ::/0` ou split-tunnel).

---

### 2.3. Conectividade interna

**Objetivo:** Garantir acesso interno pela VPN.

**Local:** estação do cliente (com túnel ativo).

**Comandos:**

```bash
ping <VPN_GATEWAY_IP>
dig @<VPN_GATEWAY_IP> <ACCESSHUB_DOMAIN> +short
curl -k https://<ACCESSHUB_DOMAIN>/guacamole/ -I
```

**Resultado esperado:**

* `ping` responde com latência estável.
* `dig` retorna IP interno esperado (ex.: `<VPN_GATEWAY_IP>`).
* `curl` retorna `HTTP/1.1 200 OK` ou `304` para `/guacamole/`.

---

### 2.4. Logs da VPN no servidor

**Objetivo:** Validar que o servidor registra conexões do cliente.

**Local:** servidor.

**Comandos:**

```bash
sudo systemctl status wg-quick@wg0
sudo journalctl -u wg-quick@wg0 -f
```

**Resultado esperado:**

* Serviço `wg-quick@wg0` em `active (running)`.
* Entradas de log correspondendo aos handshakes e tráfego do peer do cliente.

---

## 3. Testes do Guacamole, Tomcat e MySQL

### 3.1. Status dos serviços

**Objetivo:** Garantir que todos os serviços necessários estão ativos.

**Local:** servidor.

**Comandos:**

```bash
sudo systemctl status tomcat9
sudo systemctl status guacd
sudo systemctl status mysql
```

**Resultado esperado:**

* Todos com `active (running)`.
* Para Tomcat, logs mostrando deployment de `guacamole.war` e leitura de `/etc/guacamole/guacamole.properties`.

---

### 3.2. Deployment do Guacamole

**Objetivo:** Comprovar que o Guacamole foi deployado corretamente pelo Tomcat.

**Local:** servidor.

**Comando:**

```bash
journalctl -u tomcat9 -n 100
```

**Resultado esperado (exemplos de mensagens):**

* `GUACAMOLE_HOME is "/etc/guacamole".`
* `Read configuration parameters from "/etc/guacamole/guacamole.properties".`
* `Extension "MySQL Authentication" (mysql) loaded.`
* `Deployment of web application archive [/var/lib/tomcat9/webapps/guacamole.war] has finished in [...] ms`
* Mensagens indicando server startup concluído.

---

### 3.3. Acesso à UI do Guacamole

**Objetivo:** Validar acesso web ao Guacamole pela VPN.

**Local:** estação do cliente (com VPN ativa).

**Passos:**

1. Abrir navegador e acessar:
   `https://<ACCESSHUB_DOMAIN>/guacamole/`
2. Verificar:

   * Página de login renderizada corretamente.
   * Cadeado HTTPS valido.

**Resultado esperado:**

* Tela de login do Guacamole carregada.
* Nenhum erro de certificado ou de proxy.

---

### 3.4. Autenticação no Guacamole

**Objetivo:** Validar login e integração com MySQL.

**Local:** estação do cliente (via VPN).

**Passos:**

1. Autenticar com usuário `guacadmin` ou usuário fornecido pelo cliente.
2. Abrir pelo menos uma conexão publicada (RDP/SSH).

**Resultados esperados:**

* Login bem-sucedido.
* Sessão de desktop/terminal apresentada.
* No `journalctl -u tomcat9`, log similar a:

  ```text
  INFO  o.a.g.r.auth.AuthenticationService - User "guacadmin" successfully authenticated from [10.8.0.2, 127.0.0.1].
  ```

---

## 4. Testes do Vault

### 4.1. Acesso via UI

**Objetivo:** Validar acesso à interface web do Vault via proxy.

**Local:** estação do cliente (via VPN).

**Passos:**

1. Acessar:
   `https://<ACCESSHUB_DOMAIN>/vault/`
2. Confirmar redirecionamento para `/vault/ui/`.
3. Verificar tela de login da UI do Vault.

**Resultado esperado:**

* UI do Vault carregando via HTTPS.
* Em `access.log` do NGINX, entradas 200 para `/vault/ui/` originadas de `10.8.0.2`.

---

### 4.2. Status do Vault via CLI

**Objetivo:** Garantir que o Vault está inicializado e unsealed.

**Local:** servidor ou estação com CLI Vault (via VPN).

**Comandos:**

```bash
export VAULT_ADDR="https://<ACCESSHUB_DOMAIN>/vault"
vault status
```

**Resultado esperado:**

* `sealed: false`
* `initialized: true`
* Sem erros de conexão TLS.

Opcionalmente:

```bash
curl -k https://127.0.0.1:8200/v1/sys/health
```

**Resultado esperado:**

* JSON com `sealed:false` e HTTP 200 (ou códigos configurados).

---

### 4.3. Engine KV – Segredo do Guacamole

**Objetivo:** Validar que o segredo de DB do Guacamole está acessível no Vault.

**Local:** CLI Vault autenticada com token adequado.

**Comando:**

```bash
vault kv get secret/guacamole/db
```

**Resultado esperado:**

* Retorno com `username`, `password` ou campos equivalentes usados pela app.

---

### 4.4. Engine Database – Credenciais dinâmicas

**Objetivo:** Comprovar emissão de credenciais dinâmicas para o Guacamole.

**Local:** CLI Vault.

**Comando:**

```bash
vault read database/creds/guacamole-service
```

**Resultado esperado:**

* Usuário/senha gerados dinamicamente.
* Campo `ttl` com valor adequado (ex.: `1h`, `24h`).

---

### 4.5. AppRole do Guacamole

**Objetivo:** Testar AppRole usado pela integração Guacamole ↔ Vault.

**Local:** CLI Vault.

**Comandos:**

```bash
vault read -field=role_id auth/approle/role/guacamole/role-id
vault write -f -field=secret_id auth/approle/role/guacamole/secret-id
```

**Resultado esperado:**

* `role_id` retornado.
* `secret_id` válido emitido para o role.

---

### 4.6. Auditoria do Vault

**Objetivo:** Garantir que as operações do Vault estão sendo auditadas.

**Local:** servidor.

**Comando:**

```bash
sudo tail -n 50 /var/log/vault/audit.log
```

**Resultado esperado:**

* Entradas correspondendo aos comandos `vault kv get`, `vault read`, etc., com timestamp e detalhes da requisição.

---

## 5. Observabilidade (Grafana, Loki, Promtail)

### 5.1. Acesso ao Grafana

**Objetivo:** Validar painel de observabilidade do AccessHub.

**Local:** estação do cliente (via VPN).

**Passos:**

1. Acessar:
   `https://<ACCESSHUB_DOMAIN>/grafana/`
2. Autenticar como `admin` (ou usuário configurado).
3. Abrir o dashboard **“AccessHub Observability”** (ou nome equivalente).

**Resultado esperado:**

* Dashboard exibindo métricas/logs de:

  * NGINX
  * WireGuard
  * Guacamole
  * Vault
* Dados recentes (última 1h) visíveis.

---

### 5.2. Teste de consulta de logs (Loki)

**Objetivo:** Validar ingestão de logs pelo Loki.

**Local:** servidor (ou host com acesso ao Loki).

**Exemplo com cURL:**

```bash
curl -G --data-urlencode 'query={job="guacamole"}' \
  'http://127.0.0.1:3100/loki/api/v1/query_range'
```

**Resultado esperado:**

* JSON contendo `streams` com entries recentes do job `guacamole`.

**Exemplo com `logcli` (se instalado):**

```bash
logcli query '{job="vault_audit"}' --since=1h
```

**Resultado esperado:**

* Lista de logs de auditoria do Vault nas últimas 1h.

---

### 5.3. Status de Loki, Promtail e Grafana

**Objetivo:** Garantir que todos os componentes da cadeia de observabilidade estão operacionais.

**Local:** servidor.

**Comandos:**

```bash
sudo systemctl status loki
sudo systemctl status promtail
sudo systemctl status grafana-server
```

**Resultado esperado:**

* Todos `active (running)`.

Se estiver usando `positions.yaml` no Promtail, opcionalmente:

```bash
sudo tail -n 50 /caminho/para/positions.yaml
```

**Resultado esperado:**

* Arquivo sendo atualizado (positions variando com o tempo).

---

## 6. Checklist de Evidências para Entrega ao Cliente

Para concluir a homologação, o relatório final deve conter, no mínimo:

1. **DNS e HTTPS**

   * Output do `dig` público e interno.
   * Output do `curl -I http://...` com redirecionamento 301.
   * Trecho do `openssl s_client` com `Verify return code: 0 (ok)`.

2. **Nginx**

   * Print do `systemctl status nginx`.
   * Output do `nginx -t`.
   * Trechos de `access.log` mostrando acessos via `10.8.0.2` a `/grafana/`, `/vault/ui/`, `/guacamole/`.
   * Trechos de `error.log` com `access forbidden by rule` para acessos externos, comprovando restrição.

3. **Certbot**

   * Print do `systemctl list-timers | grep certbot`.
   * (Opcional) Output do `certbot renew --dry-run`.

4. **VPN WireGuard**

   * Print do `sudo wg show` com `latest handshake` recente e estatísticas de tráfego.
   * Output de `ping <VPN_GATEWAY_IP>` e `dig @<VPN_GATEWAY_IP> ...`.
   * `curl -k https://<ACCESSHUB_DOMAIN>/guacamole/ -I`.

5. **Guacamole / Tomcat / MySQL**

   * `systemctl status tomcat9`, `guacd`, `mysql`.

   * Trechos do `journalctl -u tomcat9` mostrando:

     * Deployment de `guacamole.war`.
     * Leitura de `/etc/guacamole/guacamole.properties`.
     * Mensagem de autenticação bem-sucedida, ex.:

       ```text
       AuthenticationService - User "guacadmin" successfully authenticated from [10.8.0.2, 127.0.0.1].
       ```

   * Print da tela de login e de uma sessão de conexão aberta no Guacamole.

6. **Vault**

   * Print da UI de login (`/vault/ui/`).
   * Output do `vault status`.
   * Output do `vault kv get secret/guacamole/db`.
   * Output do `vault read database/creds/guacamole-service`.
   * Output do `vault read ... role-id` e `vault write ... secret-id`.
   * Trecho de `/var/log/vault/audit.log` com chamadas realizadas durante o teste.

7. **Observabilidade**

   * Print do dashboard do Grafana mostrando métricas/logs recentes.
   * Output de consulta ao Loki (`curl` ou `logcli`).
   * `systemctl status loki`, `promtail`, `grafana-server`.

