# OCI Vue Static Site with API Gateway

Aplicacao Vue 3 + Vite para publicar um site estatico em OCI Object Storage e expor o acesso por OCI API Gateway.

O projeto segue a ideia do tutorial oficial da OCI:

https://docs.oracle.com/en/learn/oci-api-gateway-web-hosting/

## O que este app faz

- Gera um build estatico em `dist/`.
- Usa assets relativos (`./assets/...`) para funcionar atras de um deployment path do API Gateway.
- Inclui botoes de teste que chamam `GET /health` e `GET /teste` no mesmo deployment do site.
- Permite sobrescrever o endpoint de API com `VITE_API_BASE_URL`, mas isso e opcional.

## Requisitos

- Node.js 20 ou superior
- npm
- OCI CLI autenticado, opcional para upload por linha de comando
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

Depois do build, envie o conteudo de `dist/` para o bucket.

### Opcao 1: upload manual pelo Console

Voce pode fazer upload pelo Console da OCI. O ponto mais importante e preencher o `Content-Type` correto para cada tipo de arquivo.

No bucket, faca os uploads em grupos:

```text
dist/index.html       -> bucket root
dist/assets/*.css     -> assets/
dist/assets/*.js      -> assets/
```

Para `index.html`:

1. Abra o bucket no Object Storage.
2. Clique em `Upload`.
3. Selecione apenas `dist/index.html`.
4. Em `Optional response headers and metadata`, adicione um item.
5. Em `Type`, selecione `Response header`.
6. Em `Name`, preencha `Content-Type`.
7. Em `Value`, preencha `text/html`.
8. Conclua o upload.

Para arquivos CSS:

1. Entre na pasta `assets/` do bucket, ou use `Object name prefix` com o valor `assets/`.
2. Selecione os arquivos `dist/assets/*.css`.
3. Em `Optional response headers and metadata`, adicione:

```text
Type: Response header
Name: Content-Type
Value: text/css
```

Para arquivos JavaScript:

1. Entre na pasta `assets/` do bucket, ou use `Object name prefix` com o valor `assets/`.
2. Selecione os arquivos `dist/assets/*.js`.
3. Em `Optional response headers and metadata`, adicione:

```text
Type: Response header
Name: Content-Type
Value: application/javascript
```

Nao deixe `Type` como `Metadata` para `Content-Type`. O correto e `Response header`, porque o navegador precisa receber esse header na resposta HTTP.

Se o app tiver outros tipos de arquivo, use estes valores comuns:

```text
*.json   application/json
*.svg    image/svg+xml
*.png    image/png
*.jpg    image/jpeg
*.jpeg   image/jpeg
*.ico    image/x-icon
*.woff2  font/woff2
```

Evite enviar arquivos de tipos diferentes no mesmo upload quando precisar configurar `Content-Type`, porque o mesmo header pode ser aplicado ao lote inteiro.

### Opcao 2: upload via OCI CLI

Exemplo com OCI CLI:

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

### Rota de teste para backend

Crie uma terceira rota no mesmo deployment:

```text
Path: /teste
Methods: GET
```

Para reproduzir o exemplo deste repo sem depender de outro servico, use:

```text
Backend type: Stock response
Status code: 200
Body: {"source":"stock-response","status":"ok"}
Header: Content-Type = application/json
```

O botao "Testar backend" do app chama essa rota automaticamente. A resposta esperada no front e algo parecido com:

```text
HTTP 200 OK

{
  "source": "stock-response",
  "status": "ok"
}
```

Se voce clicar em "Testar backend" antes de criar essa rota, o API Gateway pode cair na rota `/{req*}` e devolver o `index.html` do site com `HTTP 200`. Isso nao significa que o backend funcionou; significa que a rota `/teste` ainda nao esta configurada ou foi capturada pelo fallback do site estatico. O app mostra um aviso quando detecta esse HTML.

Se voce tiver um backend real em OKE, uma VM, um Load Balancer privado/publico ou outro servico HTTP, configure a mesma rota `/teste` com:

```text
Backend type: HTTP
URL: http://ENDERECO_DO_BACKEND:PORTA/CAMINHO
```

ou:

```text
Backend type: HTTP
URL: https://ENDERECO_DO_BACKEND/CAMINHO
```

Use `GET` se o endpoint do backend responder a GET. Se o backend estiver em OKE, geralmente a URL mais simples para o API Gateway apontar e a URL de um OCI Load Balancer na frente do Service Kubernetes. Se estiver em uma VM, use o IP privado, IP publico ou FQDN que seja alcancavel a partir da subnet do API Gateway.

Exemplos:

```text
http://10.0.2.15:8080/teste
```

```text
http://meu-load-balancer-privado:8080/teste
```

```text
https://api.exemplo.com/teste
```

Para backend privado na mesma VCN/subnet ou em subnet pareada, confirme:

- O Security List ou NSG da subnet do API Gateway permite egress para o IP/porta do backend.
- O Security List ou NSG do backend permite ingress vindo da subnet ou NSG do API Gateway.
- A porta configurada no backend HTTP do API Gateway e a porta real exposta pelo servico sao a mesma.
- O backend responde em HTTP/HTTPS no caminho configurado.

Quando configurado com backend HTTP, a resposta esperada no front e o status e corpo devolvidos pelo seu backend. Por exemplo, se o backend responder `{"app":"oke","status":"ok"}`, o painel vai mostrar esse JSON abaixo de `HTTP 200 OK`.

## Rede

O API Gateway precisa estar em uma subnet com regras compativeis:

- Ingress TCP 443 para permitir acesso publico ao gateway.
- Egress TCP 443 para permitir que o gateway acesse o Object Storage pela URL HTTPS da PAR.
- Egress para a porta do backend, se a rota `/teste` apontar para OKE, VM ou outro backend HTTP privado.

Se o site nao carregar, verifique primeiro:

- A URL do deployment termina com `/`.
- A PAR ainda nao expirou.
- A rota `/{req*}` esta usando as URLs corretas.
- Os arquivos existem no bucket com `Content-Type` correto.
- A subnet do gateway permite ingress/egress em TCP 443.
- A rota `/teste`, se usada com backend real, consegue alcancar o IP/FQDN e a porta do backend.

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
