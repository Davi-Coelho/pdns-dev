# pdns-dev

Servidor DNS autoritativo baseado no [PowerDNS](https://www.powerdns.com/) para gerenciar múltiplas zonas DNS. Usa backend SQLite, expõe uma API REST para criação e atualização de registros, e inclui o [PowerDNS-Admin](https://github.com/PowerDNS-Admin/PowerDNS-Admin) como interface web.

A cadeia de resolução funciona assim:

```
Requisição externa
    → Cloudflare — autoritativo de example.com
    → ns1.example.com (máquina onde o container roda, porta 53)
    → PowerDNS dentro do container responde com o IP do subdomínio
```

---

## Pré-requisitos

- Docker e Docker Compose instalados na máquina servidora
- Portas 53 (TCP e UDP) disponíveis no host
- Traefik rodando na rede Docker `web`, com entrypoint `websecure` e certresolver `cloudflare` configurados
- Acesso ao painel do Cloudflare para configurar a delegação de zonas

> **Nota:** Em máquinas com `systemd-resolved`, a porta 53 pode estar ocupada. Solução:
> ```bash
> echo "DNSStubListener=no" | sudo tee -a /etc/systemd/resolved.conf
> sudo systemctl restart systemd-resolved
> ```

---

## Configuração inicial

```bash
cp .env.example .env
cp pdns.conf.example pdns.conf
```

Edite `.env` e preencha as variáveis:

| Variável | Descrição | Exemplo |
|---|---|---|
| `API_KEY` | Chave para autenticar chamadas à API REST do PowerDNS | `senha-segura` |
| `API_HOST` | Hostname pelo qual o Traefik expõe a API REST | `ns1.example.com` |
| `ADMIN_HOST` | Hostname pelo qual o Traefik expõe o PowerDNS-Admin | `pdns-admin.example.com` |
| `SECRET_KEY` | Chave secreta da sessão do PowerDNS-Admin (Flask) | `outra-senha-segura` |

Edite `pdns.conf` e defina o mesmo valor de `API_KEY` no campo `api-key`.

---

## Subindo o serviço

```bash
docker compose up -d
```

O PowerDNS-Admin estará disponível em `https://<ADMIN_HOST>`. No primeiro acesso, crie o usuário administrador pela interface.

---

## Configuração no Cloudflare

Para cada zona delegada, crie os seguintes registros na zona `example.com` no Cloudflare:

```
ns1.example.com.    A    <IPv4 do servidor>      (DNS only)
dev.example.com.    NS   ns1.example.com.     (DNS only)
lab.example.com.    NS   ns1.example.com.     (DNS only)
```

---

## Usando a API

A API é exposta pelo Traefik via HTTPS no host definido em `API_HOST`. Exemplos:

### Criar uma zona

```bash
curl -s -X POST "https://<API_HOST>/api/v1/servers/localhost/zones" \
  -H "X-API-Key: <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "dev.example.com.",
    "kind": "Native",
    "nameservers": ["ns1.example.com."]
  }'
```

### Criar/atualizar um registro

```bash
curl -s -X PATCH "https://<API_HOST>/api/v1/servers/localhost/zones/dev.example.com." \
  -H "X-API-Key: <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "rrsets": [{
      "name": "teste.dev.example.com.",
      "type": "A",
      "ttl": 3600,
      "changetype": "REPLACE",
      "records": [{"content": "1.2.3.4", "disabled": false}]
    }]
  }'
```

---

## Verificando o funcionamento

```bash
# Testa diretamente no servidor
dig A teste.dev.example.com @<ip-do-servidor>

# Verifica a cadeia de delegação completa
dig +trace teste.dev.example.com

# Confirma a delegação no Cloudflare
dig NS dev.example.com
```

---

## Persistência

- Dados do PowerDNS (SQLite): `./data`
- Dados do PowerDNS-Admin (SQLite): `./admin-data`

```bash
chmod -R a+w ./data ./admin-data
```

---

## Logs

```bash
docker logs -f pdns
docker logs -f pdns-admin
```
