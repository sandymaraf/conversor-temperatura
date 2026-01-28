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

seu-dominio.duckdns.org → IP_DA_EC2

## Estrutura do Projeto

conversor-temperatura/
├── site/
│   └── index.html
├── nginx/
│   └── conf.d/
│       └── default.conf
├── certbot/
└── Dockerfile


## Dockerfile

Arquivo: `Dockerfile`

    
    FROM nginx:alpine
    COPY site /usr/share/nginx/html


## Estrutura de Diretórios

`.
├── Dockerfile
├── site/
├── nginx/
│   └── conf.d/
│       └── default.conf
└── certbot/` 


## Dockerfile

Arquivo: `Dockerfile`

    FROM nginx:alpine
    COPY site /usr/share/nginx/html

 
## Configuração do Nginx

Arquivo: `nginx/conf.d/default.conf`

    `server { listen  80; server_name seu-dominio.duckdns.org www.seu-dominio.duckdns.org; # Diretório utilizado pelo Certbot para validação do domínio (ACME challenge)  location /.well-known/acme-challenge/ { alias /var/www/certbot/.well-known/acme-challenge/;
        } # Redireciona todas as requisições HTTP para HTTPS  location / { return  301 https://$host$request_uri;
        }
        
    } server { listen  443 ssl; server_name seu-dominio.duckdns.org www.seu-dominio.duckdns.org; ssl_certificate /etc/letsencrypt/live/seu-dominio.duckdns.org/fullchain.pem; ssl_certificate_key /etc/letsencrypt/live/seu-dominio.duckdns.org/privkey.pem; location / { root /usr/share/nginx/html; index index.html;
        }
    }`
 
 

## Build da Imagem Docker

`docker build -t conversor-temp .` 

----------

## Subindo o Nginx em Modo HTTP

Este passo é necessário inicialmente para permitir que o Certbot valide o domínio.

`docker run -d \   --name nginx-http \   -p 80:80 \   -v $(pwd)/nginx/conf.d:/etc/nginx/conf.d \   -v $(pwd)/certbot:/var/www/certbot \   -v $(pwd)/site:/usr/share/nginx/html \   nginx:latest`

----------

## Geração do Certificado SSL com Certbot

`docker run -it --rm \
  -v /etc/letsencrypt:/etc/letsencrypt \
  -v /var/lib/letsencrypt:/var/lib/letsencrypt \
  -v $(pwd)/certbot:/var/www/certbot \
  certbot/certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  -d seu-dominio.duckdns.org` 

----------

### O que este comando faz:

-   Utiliza a imagem oficial do Certbot
    
-   Armazena os certificados em `/etc/letsencrypt` no host
    
-   Usa o método `webroot` para comprovar a posse do domínio
    
-   O Let's Encrypt acessa:
    

`http://seu-dominio/.well-known/acme-challenge/...` 

-   Se a validação for bem-sucedida, o certificado é emitido
    

----------

## Subindo o Nginx com HTTPS

Primeiro, remova o container HTTP:

`docker stop nginx-http
docker rm nginx-http` 

Em seguida, suba o Nginx com suporte a HTTPS:

     docker run -d --name nginx-https -p 80:80 -p 443:443 -v /etc/letsencrypt:/etc/letsencrypt -v $(pwd)/nginx/conf.d:/etc/nginx/conf.d -v $(pwd)/certbot:/var/www/certbot -v $(pwd)/site:/usr/share/nginx/html nginx:latest


----------

## Testes de Funcionamento

`curl -I http://seu-dominio.duckdns.org
curl -I https://seu-dominio.duckdns.org` 

Resultado esperado:

-   HTTP deve responder com redirecionamento para HTTPS
    
-   HTTPS deve responder com status `200 OK`
    

----------

## Sobre o Certificado SSL

O certificado SSL é utilizado para:

-   Criptografar a comunicação entre cliente e servidor
    
-   Garantir a autenticidade do servidor
    
-   Evitar alertas de segurança no navegador
    

O Let's Encrypt fornece certificados gratuitos, porém:

-   Eles possuem validade de 90 dias
    

----------

## Renovação do Certificado

A renovação pode ser realizada com o comando:

`docker run --rm \
  -v /etc/letsencrypt:/etc/letsencrypt \
  -v /var/lib/letsencrypt:/var/lib/letsencrypt \
  -v $(pwd)/certbot:/var/www/certbot \
  certbot/certbot renew` 

Após a renovação, reinicie o Nginx:

`docker restart nginx-https` 

----------

## Observações Importantes

-   As portas 22, 80 e 443 devem estar liberadas no Security Group da AWS
    
-   O DNS deve estar corretamente apontando para o IP da instância EC2 antes da execução do Certbot
    
-   O Nginx deve estar em execução na porta 80 durante o processo de emissão do certificado
    

----------

## Resultado Final

A aplicação estará disponível em:

`https://seu-dominio.duckdns.org`
