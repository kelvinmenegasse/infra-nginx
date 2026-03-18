# Infra Nginx - Proxy Reverso Centralizado

Configuracao centralizada do Nginx para toda a stack nova-stack.

## Quick Start

```bash
# Criar rede compartilhada (se ainda nao existir)
docker network create app-network

# Iniciar
docker compose up -d

# Verificar
docker compose ps
```

## Estrutura

```
infra-nginx/
├── docker-compose.yml        # Docker Compose principal
├── nginx.conf                # Configuracao principal
├── conf.d/                   # Snippets reutilizaveis
│   ├── cors.conf             # Headers CORS
│   ├── proxy-params.conf     # Parametros de proxy padrao
│   ├── resolver.conf         # DNS resolver Docker (127.0.0.11)
│   ├── security-headers.conf # Headers de seguranca
│   ├── ssl.conf              # Configuracoes SSL/TLS
│   ├── error-pages.conf      # Paginas de erro customizadas
│   └── upstreams.conf        # Map de upstreams dinamicos
├── sites-available/          # Um arquivo por server block (30 sites)
│   ├── api.conf
│   ├── castracao.conf
│   ├── cep.conf
│   ├── cestabasica.conf
│   ├── cidadao.conf
│   ├── default.conf
│   ├── desenvolver-rh.conf
│   ├── escoladegoverno.conf
│   ├── esus.conf
│   ├── francosemfila.conf
│   ├── geo.conf
│   ├── gestaodepessoas.conf
│   ├── homologacao-*.conf
│   ├── indicadores.conf
│   ├── keycloak.conf
│   ├── mapas.conf
│   ├── memoria.conf
│   ├── nossoolhar.conf
│   ├── oficinas.conf
│   ├── opina.conf
│   ├── pmapi.conf
│   ├── pobrezamenstrual.conf
│   ├── tasks.conf
│   └── transporte.conf
└── cert/                     # Certificados SSL (criar manualmente)
```

## Variaveis de Ambiente

| Variavel | Padrao | Descricao |
|----------|--------|-----------|
| `NGINX_HTTP_PORT` | `80` | Porta HTTP |
| `NGINX_HTTPS_PORT` | `443` | Porta HTTPS |

## Adicionar Novo Site

1. Crie um arquivo em `sites-available/`:

```nginx
# sites-available/meu-app.conf
server {
    listen 80;
    server_name meu-app.exemplo.com;

    include conf.d/security-headers.conf;

    location / {
        set $upstream meu-app:3000;
        proxy_pass http://$upstream;
        include conf.d/proxy-params.conf;
    }
}
```

2. Reinicie o Nginx:

```bash
docker compose restart
```

## Snippets Disponiveis

### security-headers.conf
Headers de seguranca (X-Frame-Options, X-Content-Type-Options, etc.)

### proxy-params.conf
Configuracoes padrao para proxy reverso (Host, X-Real-IP, etc.)

### cors.conf
Headers CORS para APIs

### ssl.conf
Configuracoes SSL/TLS (protocolos, ciphers, HSTS)

### resolver.conf
DNS resolver do Docker para resolucao dinamica de upstreams

### upstreams.conf
Mapas de upstreams para resolucao dinamica:

```nginx
map $host $upstream_keycloak {
    default keycloak:8080;
}

map $host $upstream_backend {
    default backend:3000;
}
```

## SSL/TLS

1. Coloque os certificados em `cert/`:

```
cert/
├── fullchain.pem
└── privkey.pem
```

2. Use o snippet ssl.conf:

```nginx
server {
    listen 443 ssl http2;
    server_name exemplo.com;

    include conf.d/ssl.conf;
    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    # ...
}
```

## Rate Limiting

Zonas pre-configuradas em `nginx.conf`:

| Zona | Rate | Descricao |
|------|------|-----------|
| `auth_limit` | 10 req/s | Endpoints de autenticacao |
| `api_limit` | 100 req/s | Endpoints de API |

Uso:

```nginx
location /api {
    limit_req zone=api_limit burst=100 nodelay;
    # ...
}
```

## Logs

```bash
# Ver logs em tempo real
docker compose logs -f nginx

# Logs no container
docker exec nginx tail -f /var/log/nginx/access.log
```

## Health Check

```bash
curl http://localhost/health
```

## Rede

Este servico usa a rede `app-network` compartilhada com os demais servicos:

```yaml
networks:
  app-network:
    external: true
```

Certifique-se de que a rede existe antes de iniciar:

```bash
docker network create app-network
```

## Relacionamento com Outros Servicos

Este nginx serve como proxy reverso para:

- **Keycloak:** `auth.exemplo.com` -> `keycloak:8080`
- **Backend:** `api.exemplo.com` -> `backend:3000`
- **Frontend:** `exemplo.com` -> `frontend:80`
- **Grafana:** `grafana.exemplo.com` -> `grafana:3000`
- **Outros 26 sites** configurados em `sites-available/`
