# AccessHub Infra Guide (public)

Documentação sanitizada para operação da pilha AccessHub (VPN WireGuard, Guacamole, Vault e observabilidade). Nenhum segredo real é armazenado aqui; os valores de IP, hostnames, chaves e credenciais foram substituídos por placeholders.

## Conteúdo
- `docs_cliente/` — guias de arquitetura, acesso à VPN, Guacamole, Vault, observabilidade e procedimentos de emergência.
- `TESTES.md` — plano de homologação ponta a ponta com checklist e comandos.

## Como usar
1) Substitua os placeholders antes de entregar ao cliente:
   - `<ACCESSHUB_DOMAIN>`: domínio público (ex.: accesshub.seudominio.com).
   - `<VPN_PUBLIC_IP>`: IP/porta pública do WireGuard (ex.: 203.0.113.10:51820).
   - `<VPN_GATEWAY_IP>`: gateway interno da VPN (ex.: 10.8.0.1).
   - `<WG_SERVER_PUBLIC_KEY>`: chave pública do servidor WireGuard.
   - Credenciais padrão (ex.: `admin`) devem ser ajustadas conforme sua política.
2) Revise os comandos e ranges de rede para refletir sua topologia.
3) Gere novos perfis WireGuard e senhas; não reutilize valores exemplificados.

## Escopo
- Arquivos sensíveis (`clientes/`, `vault/`, chaves e configs reais) **não** fazem parte deste repositório.
- Binaries open-source (Guacamole, connectors, etc.) não foram incluídos para manter o repositório leve; use as releases oficiais.

## Licença
Uso interno do cliente. Ajuste conforme a política da sua organização.
