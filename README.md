# Conversor de Temperatura — Deploy com Docker, Nginx e HTTPS na AWS

Este repositório documenta o processo de deploy de uma aplicação estática em uma instância EC2 da AWS usando Docker, Nginx e certificados SSL gratuitos do Let's Encrypt (via Certbot).

O objetivo do projeto é estudar e consolidar conceitos de deploy em produção, HTTPS, DNS e containerização.

---

## Arquitetura

Usuário → DuckDNS → AWS EC2 → Docker → Nginx → Aplicação
└── Certbot (Let's Encrypt)


---

## Configuração de Rede (AWS)

No Security Group da instância EC2, é necessário liberar as seguintes portas:

| Porta | Uso |
|------:|------|
| 22 | Acesso SSH à máquina |
| 80 | HTTP (validação do domínio e redirecionamento) |
| 443 | HTTPS |

---

## DNS

Foi utilizado o DuckDNS para apontar o domínio para o IP público da instância EC2.

Exemplo:

converter-temperature.duckdns.org → IP_DA_EC2


---

## Estrutura do Projeto

conversor-temperatura/
├── site/
│ └── index.html
├── nginx/
│ └── conf.d/
│ └── default.conf
├── certbot/
└── Dockerfile


---

## Dockerfile

```dockerfile
FROM nginx:alpine
COPY site /usr/share/nginx/html

Configuração do Nginx

Arquivo: nginx/conf.d/default.conf

server {
    listen 80;
    server_name converter-temperature.duckdns.org www.converter-temperature.duckdns.org;

    location /.well-known/acme-challenge/ {
        alias /var/www/certbot/.well-known/acme-challenge/;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name converter-temperature.duckdns.org www.converter-temperature.duckdns.org;

    ssl_certificate /etc/letsencrypt/live/converter-temperature.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/converter-temperature.duckdns.org/privkey.pem;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }
}

Build da Imagem

docker build -t conversor-temp .

Subindo o Nginx (HTTP)

docker run -d \
  --name nginx-http \
  -p 80:80 \
  -v $(pwd)/nginx/conf.d:/etc/nginx/conf.d \
  -v $(pwd)/certbot:/var/www/certbot \
  -v $(pwd)/site:/usr/share/nginx/html \
  nginx:latest

Gerando o Certificado com Certbot

docker run -it --rm \
  -v /etc/letsencrypt:/etc/letsencrypt \
  -v /var/lib/letsencrypt:/var/lib/letsencrypt \
  -v $(pwd)/certbot:/var/www/certbot \
  certbot/certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  -d converter-temperature.duckdns.org

Subindo o Nginx com HTTPS

docker stop nginx-http
docker rm nginx-http

docker run -d \
  --name nginx-https \
  -p 80:80 \
  -p 443:443 \
  -v /etc/letsencrypt:/etc/letsencrypt \
  -v $(pwd)/nginx/conf.d:/etc/nginx/conf.d \
  -v $(pwd)/certbot:/var/www/certbot \
  -v $(pwd)/site:/usr/share/nginx/html \
  nginx:latest

Testes

curl -I http://converter-temperature.duckdns.org
curl -I https://converter-temperature.duckdns.org

Renovação do Certificado

Os certificados do Let's Encrypt expiram a cada 90 dias.

docker run --rm \
  -v /etc/letsencrypt:/etc/letsencrypt \
  -v /var/lib/letsencrypt:/var/lib/letsencrypt \
  -v $(pwd)/certbot:/var/www/certbot \
  certbot/certbot renew

docker restart nginx-https

