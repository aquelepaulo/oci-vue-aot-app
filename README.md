# OCI Vue Static Site with API Gateway

Aplicacao Vue 3 + Vite para publicar um site estatico em OCI Object Storage e expor o acesso por OCI API Gateway.

O projeto segue a ideia do tutorial oficial da OCI:

https://docs.oracle.com/en/learn/oci-api-gateway-web-hosting/

## O que este app faz

- Gera um build estatico em `dist/`.
- Usa assets relativos (`./assets/...`) para funcionar atras de um deployment path do API Gateway.
- Inclui um botao de teste que chama `GET /health` no mesmo deployment do site.
- Permite sobrescrever o endpoint de API com `VITE_API_BASE_URL`, mas isso e opcional.

## Requisitos

- Node.js 20 ou superior
- npm
- OCI CLI autenticado
- Permissoes para Object Storage e API Gateway na sua tenancy OCI

## Rodar localmente

```bash
npm install
```

```bash
npm run dev
```

## Gerar build

```bash
npm run build
```

A saida fica em:

```text
dist/
```

## Configurar bucket

Crie um bucket no Object Storage. Ele pode ser privado, porque o API Gateway acessara os objetos por PAR.

Depois do build, envie o conteudo de `dist/` para o bucket. Exemplo com OCI CLI:

```bash
oci os ns get --profile SEU_PROFILE --auth security_token
```

```bash
oci os object put --namespace-name SEU_NAMESPACE --bucket-name SEU_BUCKET --file dist/index.html --name index.html --content-type text/html --force --profile SEU_PROFILE --auth security_token
```

```bash
oci os object put --namespace-name SEU_NAMESPACE --bucket-name SEU_BUCKET --file dist/assets/NOME_DO_ARQUIVO.css --name assets/NOME_DO_ARQUIVO.css --content-type text/css --force --profile SEU_PROFILE --auth security_token
```

```bash
oci os object put --namespace-name SEU_NAMESPACE --bucket-name SEU_BUCKET --file dist/assets/NOME_DO_ARQUIVO.js --name assets/NOME_DO_ARQUIVO.js --content-type application/javascript --force --profile SEU_PROFILE --auth security_token
```

Importante: os nomes dos arquivos em `dist/assets/` mudam a cada build quando o conteudo muda. Confira os nomes gerados antes do upload.

## Criar as PARs

Crie duas Pre-Authenticated Requests no bucket:

1. Uma PAR para o objeto `index.html`.
2. Uma PAR para o bucket, ou para o prefixo `assets/`, com permissao de leitura dos objetos estaticos.

Atencao para a data de expiracao da PAR. Quando a PAR expirar, o API Gateway deixara de conseguir buscar os arquivos e o site pode parar de carregar. Escolha uma expiracao adequada para o seu caso e documente essa data.

As URLs devem ficar parecidas com estas:

```text
INDEX_PAR_URL=https://objectstorage.REGIAO.oraclecloud.com/p/TOKEN_INDEX/n/NAMESPACE/b/BUCKET/o/index.html
```

```text
BUCKET_PAR_BASE_URL=https://objectstorage.REGIAO.oraclecloud.com/p/TOKEN_BUCKET/n/NAMESPACE/b/BUCKET/o/
```

Para testar direto:

```bash
curl -i "INDEX_PAR_URL"
```

```bash
curl -i "BUCKET_PAR_BASE_URL/assets/NOME_DO_ARQUIVO.js"
```

## Configurar API Gateway

Crie um API Gateway publico e um deployment.

### Rota do site estatico

Crie uma rota:

```text
Path: /{req*}
Methods: GET
Backend type: Dynamic routing
```

Configure multiplos backends:

```text
Default backend
Match: default
Backend type: HTTP
URL: INDEX_PAR_URL
```

```text
JS backend
Match type: Wildcard
Match expression: *.js
Backend type: HTTP
URL: BUCKET_PAR_BASE_URL/${request.path[req]}
```

```text
CSS backend
Match type: Wildcard
Match expression: *.css
Backend type: HTTP
URL: BUCKET_PAR_BASE_URL/${request.path[req]}
```

Se o app tiver imagens, fontes ou outros assets, adicione regras semelhantes para extensoes como `*.png`, `*.svg`, `*.woff2` e `*.ico`.

### Rota de health check

Crie uma segunda rota no mesmo deployment:

```text
Path: /health
Methods: GET
Backend type: Stock response
Status code: 200
Body: {"status":"ok"}
Header: Content-Type = application/json
```

O botao "Testar gateway" do app chama essa rota automaticamente.

## Rede

O API Gateway precisa estar em uma subnet com regras compativeis:

- Ingress TCP 443 para permitir acesso publico ao gateway.
- Egress TCP 443 para permitir que o gateway acesse o Object Storage pela URL HTTPS da PAR.

Se o site nao carregar, verifique primeiro:

- A URL do deployment termina com `/`.
- A PAR ainda nao expirou.
- A rota `/{req*}` esta usando as URLs corretas.
- Os arquivos existem no bucket com `Content-Type` correto.
- A subnet do gateway permite ingress/egress em TCP 443.

## Variavel de ambiente opcional

Por padrao, o app chama `/health` no mesmo deployment onde foi carregado.

Se quiser apontar o teste para outro endpoint, crie `.env.production` antes do build:

```bash
echo 'VITE_API_BASE_URL=https://seu-api-gateway.example.com/seu-deployment' > .env.production
```

Depois gere o build novamente:

```bash
npm run build
```

## Estrutura

```text
src/App.vue        Interface e teste do API Gateway
src/main.ts        Bootstrap Vue
src/styles.css     Estilos
vite.config.ts     Configuracao do Vite
AGENTS.md          Contexto para agentes de IA trabalharem no projeto
dist/              Build gerado, nao versionado
```
