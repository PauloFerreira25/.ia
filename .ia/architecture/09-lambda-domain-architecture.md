---
name: lambda-domain-architecture
description: "Arquitetura-alvo para novos projetos Lambda com múltiplos domínios. Leia antes de criar a estrutura de um projeto novo ou avaliar refactor do projeto atual."
---

# Arquitetura de Domínio para Projetos Lambda

> Este documento descreve o modelo arquitetural identificado como correto após avaliação dos trade-offs do projeto atual (GE Amizade). Aplique em projetos novos. Para o projeto atual, a migração não é obrigatória — o modelo atual é funcional para sistemas majoritariamente CRUD sem regras cross-domain complexas.
>
> **Pré-requisito antes de adotar:** validar que o `NodejsFunction` do CDK com esbuild resolve corretamente pacotes de workspace com subpath exports wildcard. Fazer um spike com um projeto mínimo antes de comprometer a arquitetura.

---

## O Problema que Este Modelo Resolve

O modelo anterior organiza por **camada primeiro, domínio depois**:

```
shared/schema/product/
shared/repository/product/
lambdas/v1/products/get-by-id/
```

Isso cria três problemas:

1. **Regras cross-domain duplicadas** — quando dois Lambdas precisam da mesma regra de negócio, ela se duplica ou exige shared service com constructor bloat
2. **Dependências transitivas manuais** — `file:` requer declarar explicitamente cada dependência indireta
3. **Dispersão por domínio** — entender o domínio `product` exige navegar três diretórios raiz distintos

---

## Estrutura de Projeto

Organize por **domínio primeiro, camada depois**. A raiz `app/` é o workspace npm:

```
app/                                ← workspace root (package.json com workspaces listados pacote a pacote)
  product/                          ← @<project>/product
    src/
      schema/index.ts               ← Zod schemas + tipos inferidos
      port/
        repository.ts               ← contratos entre repository e service
        service.ts                  ← contratos expostos para orquestradores
      repository/
        find-by-id.ts
        save.ts
        get-stock-transact-item.ts
      service/
        get-by-id.ts
        decrease-stock.ts
        get-stock-transact-item.ts
    package.json

  order/                            ← @<project>/order
    src/
      schema/index.ts
      port/
        repository.ts
        service.ts
      repository/
        save.ts
        get-put-transact-item.ts
      service/
        create.ts
        get-put-transact-item.ts
    package.json

  checkout/                         ← @<project>/checkout (orquestrador — sem repository)
    src/
      schema/index.ts
      service/
        create.ts
    package.json

  shared/                           ← pacotes que cruzam domínios (cada um independente)
    infra-dynamo/                   ← @<project>/infra-dynamo
    error/                          ← @<project>/error
    util/                           ← @<project>/util

  lambdas/                          ← composition roots — dentro do workspace
    http/
      v1/
        products/
          get/                      ← @<project>/lambda-http-v1-products-get
            src/handler.ts
            package.json
          post/                     ← @<project>/lambda-http-v1-products-post
          _id/                      ← _id = path param {id}
            get/                    ← @<project>/lambda-http-v1-products-id-get
            delete/                 ← @<project>/lambda-http-v1-products-id-delete
        checkout/
          post/                     ← @<project>/lambda-http-v1-checkout-post
    sqs/
      order/
        process/                    ← @<project>/lambda-sqs-order-process
    cron/
      order/
        expire/                     ← @<project>/lambda-cron-order-expire

cdk/                                ← infraestrutura — fora do app/
```

---

## package.json do Workspace

O `app/package.json` lista cada pacote explicitamente — sem glob. Isso evita que diretórios intermediários (como `shared/`) sejam interpretados como pacotes:

```json
{
  "private": true,
  "workspaces": [
    "product",
    "order",
    "checkout",
    "shared/infra-dynamo",
    "shared/error",
    "shared/util",
    "lambdas/http/v1/products/get",
    "lambdas/http/v1/products/post",
    "lambdas/http/v1/products/_id/get",
    "lambdas/http/v1/products/_id/delete",
    "lambdas/http/v1/checkout/post",
    "lambdas/sqs/order/process",
    "lambdas/cron/order/expire"
  ]
}
```

---

## package.json do Domínio

Cada domínio é um pacote npm com **wildcard exports** por camada. Nenhuma manutenção manual — arquivo novo em qualquer camada é automaticamente exposto:

```json
{
  "name": "@<project>/product",
  "version": "0.0.0",
  "type": "module",
  "engines": { "node": "24" },
  "exports": {
    "./schema":       "./dist/schema/index.js",
    "./port/*":       "./dist/port/*.js",
    "./repository/*": "./dist/repository/*.js",
    "./service/*":    "./dist/service/*.js"
  }
}
```

`schema` usa `index.js` (único arquivo, estável). `port`, `repository` e `service` usam wildcard — cada arquivo é endereçável diretamente.

Os `exports` apontam para `./dist/` — cada pacote compila para `dist/` antes de ser consumido. O `dist/` é gerado pelo `tsc --build` e não entra no git.

---

## TypeScript

Cada pacote é auto-responsável — compila independente para `dist/` e declara explicitamente suas referências. `tsc --build` respeita a ordem de dependências e compila apenas o que mudou.

**`app/tsconfig.base.json`** — opções compartilhadas, sem `include` e sem `outDir` (paths relativos são resolvidos pelo `tsconfig.json` de cada pacote, não pelo base):
```json
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "verbatimModuleSyntax": true,
    "declaration": true
  }
}
```

**`app/product/tsconfig.json`** — herda a base, define `outDir` e `composite` localmente, declara referências para suas dependências:
```json
{
  "extends": "../tsconfig.base.json",
  "compilerOptions": {
    "composite": true,
    "rootDir": "src",
    "outDir": "dist"
  },
  "include": ["src"],
  "references": [
    { "path": "../shared/infra-dynamo" },
    { "path": "../shared/error" }
  ]
}
```

**`app/tsconfig.json`** — raiz do workspace, referencia todos os pacotes para `tsc --build` funcionar de um único ponto. Deve listar todos os pacotes do workspace:
```json
{
  "files": [],
  "references": [
    { "path": "./shared/infra-dynamo" },
    { "path": "./shared/error" },
    { "path": "./shared/util" },
    { "path": "./product" },
    { "path": "./order" },
    { "path": "./checkout" },
    { "path": "./lambdas/http/v1/products/get" },
    { "path": "./lambdas/http/v1/products/post" },
    { "path": "./lambdas/http/v1/products/_id/get" },
    { "path": "./lambdas/http/v1/products/_id/delete" },
    { "path": "./lambdas/http/v1/checkout/post" },
    { "path": "./lambdas/sqs/order/process" },
    { "path": "./lambdas/cron/order/expire" }
  ]
}
```

**`.gitignore`** — `dist/` gerado localmente e no CI, nunca commitado:
```
**/dist/
```

---

## As Camadas

### schema/index.ts

Zod schemas + tipos TypeScript inferidos. Única fonte de verdade para estrutura e validação da entidade. Não importa nada de outras camadas.

```typescript
// product/src/schema/index.ts
import { z } from 'zod'

export const ProductSchema = z.object({
  id:        z.string().uuid(),
  name:      z.string().min(1),
  price:     z.number().positive(),
  stock:     z.number().int().gte(0),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
})

export type Product = z.infer<typeof ProductSchema>

export const CreateProductSchema = ProductSchema.omit({ createdAt: true, updatedAt: true })
export type CreateProductInput   = z.infer<typeof CreateProductSchema>
```

### port/repository.ts

Assinaturas bound das operações de infraestrutura — o que o service recebe do repository (sem config, sem DynamoDB visível). Importa schema do mesmo domínio e `TransactWriteItem` do AWS SDK quando a operação é transacional.

```typescript
// product/src/port/repository.ts
import type { Product } from '../schema/index.js'
import type { TransactWriteItem } from '@aws-sdk/lib-dynamodb'

export type FindProductById             = (id: string)                        => Promise<Product | null>
export type SaveProduct                 = (product: Product)                  => Promise<void>
export type UpdateProductStock          = (id: string, stock: number)         => Promise<void>
export type GetProductStockTransactItem = (id: string, newStock: number)      => TransactWriteItem
```

### port/service.ts

Assinaturas das operações de domínio — o que um service orquestrador recebe de outro service. Importa schema do mesmo domínio e `TransactWriteItem` do AWS SDK quando expõe operação transacional.

```typescript
// product/src/port/service.ts
import type { Product, CreateProductInput } from '../schema/index.js'
import type { TransactWriteItem } from '@aws-sdk/lib-dynamodb'

export type GetProductById           = (id: string)                      => Promise<Product>
export type CreateProduct            = (input: CreateProductInput)        => Promise<Product>
export type ProductStockTransactItem = (id: string, newStock: number)     => TransactWriteItem
```

### repository/

Factory functions (`make*`) que recebem `DynamoConfig` e retornam a função bound tipada pelo port de repository. Cada arquivo é independente — esbuild inclui apenas os importados.

```typescript
// product/src/repository/find-by-id.ts
import { GetCommand } from '@aws-sdk/lib-dynamodb'
import type { DynamoConfig } from '@<project>/infra-dynamo'
import type { Product } from '../schema/index.js'
import type { FindProductById } from '../port/repository.js'

export function makeFindProductById(config: DynamoConfig): FindProductById {
  return async (id) => {
    const result = await config.client.send(
      new GetCommand({ TableName: config.tableName, Key: { id } })
    )
    return (result.Item as Product) ?? null
  }
}
```

```typescript
// product/src/repository/get-stock-transact-item.ts
import type { DynamoConfig } from '@<project>/infra-dynamo'
import type { TransactWriteItem } from '@aws-sdk/lib-dynamodb'
import type { GetProductStockTransactItem } from '../port/repository.js'

export function makeGetProductStockTransactItem(config: DynamoConfig): GetProductStockTransactItem {
  return (id, newStock): TransactWriteItem => ({
    Update: {
      TableName:                 config.tableName,
      Key:                       { id },
      UpdateExpression:          'SET stock = :s',
      ExpressionAttributeValues: { ':s': newStock },
    },
  })
}
```

### service/

Factory functions (`make*`) universais — **toda função que recebe deps usa `make*`**, sem exceção. Services de domínio recebem repository ports. Services orquestradores recebem service ports de outros domínios — nunca repositories.

```typescript
// product/src/service/get-by-id.ts
import { NotFoundError } from '@<project>/error'
import type { Product } from '../schema/index.js'
import type { FindProductById } from '../port/repository.js'
import type { GetProductById } from '../port/service.js'

type Deps = { findById: FindProductById }

export function makeGetProductById(deps: Deps): GetProductById {
  return async (id) => {
    log.debug({ id }, 'getProductById')
    const product = await deps.findById(id)
    if (!product) throw new NotFoundError('Product not found')
    return product
  }
}
```

```typescript
// product/src/service/get-stock-transact-item.ts
import type { GetProductStockTransactItem } from '../port/repository.js'
import type { ProductStockTransactItem } from '../port/service.js'

type Deps = { getTransactItem: GetProductStockTransactItem }

export function makeProductStockTransactItem(deps: Deps): ProductStockTransactItem {
  return (id, newStock) => deps.getTransactItem(id, newStock)
}
```

**Service orquestrador** — recebe service ports de outros domínios, nunca repositories. Quando coordena uma operação transacional, importa `Transact` de `infra-dynamo` — exceção intencional explicada na seção [Operações Transacionais](#operações-transacionais):

```typescript
// checkout/src/service/create.ts
import type { GetProductById, ProductStockTransactItem } from '@<project>/product/port/service'
import type { OrderPutTransactItem } from '@<project>/order/port/service'
import type { Transact } from '@<project>/infra-dynamo'

type Deps = {
  getProduct:        GetProductById
  stockTransactItem: ProductStockTransactItem
  orderTransactItem: OrderPutTransactItem
  transact:          Transact
}

export function makeCreateCheckout(deps: Deps): CreateCheckout {
  return async (input) => {
    log.debug({ input }, 'createCheckout')
    const product  = await deps.getProduct(input.productId)
    if (product.stock < input.quantity) throw new ValidationError('Insufficient stock')
    const order    = { id: randomUUID(), productId: input.productId, quantity: input.quantity }
    const newStock = product.stock - input.quantity
    await deps.transact([
      deps.orderTransactItem(order),
      deps.stockTransactItem(input.productId, newStock),
    ])
  }
}
```

### lambdas/ (Composition Root)

**Regra obrigatória:** toda instanciação de factory acontece no nível do módulo, nunca dentro da função handler. O handler recebe apenas dados de entrada — zero deps.

A Lambda é o principal lugar onde infraestrutura e domínio se conectam. A exceção é o service orquestrador em operações transacionais — ver seção [Operações Transacionais](#operações-transacionais).

**Convenções:**
- Cada método HTTP é um pacote npm independente — um bundle por Lambda, sem código compartilhado entre rotas
- Nome do pacote espelha o path: `@<project>/lambda-http-v1-checkout-post`
- Path params usam prefixo `_`: `_id` representa `{id}` na URL
- Como o CDK referencia o `entry` de cada Lambda (path relativo vs `require.resolve`) está em aberto — validar no spike

```typescript
// lambdas/http/v1/checkout/post/src/handler.ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb'
import { DynamoDBDocumentClient, TransactWriteCommand } from '@aws-sdk/lib-dynamodb'
import type { TransactWriteItem } from '@aws-sdk/lib-dynamodb'
import { makeFindProductById }             from '@<project>/product/repository/find-by-id'
import { makeGetProductStockTransactItem } from '@<project>/product/repository/get-stock-transact-item'
import { makeGetProductById }              from '@<project>/product/service/get-by-id'
import { makeProductStockTransactItem }    from '@<project>/product/service/get-stock-transact-item'
import { makeGetOrderPutTransactItem }     from '@<project>/order/repository/get-put-transact-item'
import { makeOrderPutTransactItem }        from '@<project>/order/service/get-put-transact-item'
import { makeCreateCheckout }              from '@<project>/checkout/service/create'

const client        = DynamoDBDocumentClient.from(new DynamoDBClient({}))
const productConfig = { client, tableName: process.env.PRODUCTS_TABLE! }
const orderConfig   = { client, tableName: process.env.ORDERS_TABLE! }

function makeTransact(c: DynamoDBDocumentClient) {
  return async (items: TransactWriteItem[]) => {
    await c.send(new TransactWriteCommand({ TransactItems: items }))
  }
}

// camada 1: repository → service
const getProduct        = makeGetProductById({ findById: makeFindProductById(productConfig) })
const stockTransactItem = makeProductStockTransactItem({ getTransactItem: makeGetProductStockTransactItem(productConfig) })
const orderTransactItem = makeOrderPutTransactItem({ getTransactItem: makeGetOrderPutTransactItem(orderConfig) })

// camada 2: service → orquestrador
const createCheckout = makeCreateCheckout({
  getProduct,
  stockTransactItem,
  orderTransactItem,
  transact: makeTransact(client),
})

// handler — zero deps, só dados de entrada
export const handler = async (event: APIGatewayJwtEvent): Promise<APIGatewayProxyResultV2> => {
  log.debug({ requestId: event.requestContext?.requestId }, 'handler')
  try {
    const body = parseBody(event)
    await createCheckout(body)
    return formatResponse(event, 201, { data: null })
  } catch (error) {
    return handleError(event, error)
  }
}
```

---

## Build e Deploy das Lambdas

Cada Lambda é responsável pelo próprio bundle. O CDK não bundla — só faz upload do artefato já gerado.

### Build por Lambda

`package-build.sh` é registrado como bin no `package.json` do workspace root. O npm o coloca em `node_modules/.bin/` e fica disponível em qualquer pacote sem path relativo:

```json
// app/package.json
{
  "private": true,
  "bin": {
    "package-build": "../scripts/package-build.sh"
  },
  "workspaces": [...]
}
```

**Lambda** — `package.json` chama o bin diretamente:

```json
{
  "name": "@<project>/lambda-http-v1-checkout-post",
  "version": "0.0.0",
  "type": "module",
  "engines": { "node": "24" },
  "scripts": {
    "build":      "package-build",
    "type-check": "tsc --noEmit",
    "test":       "vitest run"
  },
  "dependencies": {
    "@<project>/checkout":      "*",
    "@<project>/product":       "*",
    "@<project>/order":         "*",
    "@<project>/infra-dynamo":  "*",
    "@aws-sdk/client-dynamodb": "...",
    "@aws-sdk/lib-dynamodb":    "..."
  }
}
```

**`scripts/package-build.sh`** — script de bundling exclusivo para Lambdas. Domínios e shared declaram `"build": "tsc --build"` diretamente no `package.json`:

```bash
#!/usr/bin/env bash
set -euo pipefail

npx esbuild src/handler.ts \
  --bundle \
  --minify \
  --sourcemap=linked \
  --platform=node \
  --target=node24 \
  --format=esm \
  --outfile=dist/index.js \
  --external:@aws-sdk/*
```

Resultado: `dist/index.js` + `dist/index.js.map`. Sem `node_modules` no pacote — esbuild bundla tudo exceto `@aws-sdk/*` (disponível no runtime).

`sourcemap=linked` gera o `.map` como arquivo separado e embute o comentário `//# sourceMappingURL=` no `.js`. O Node resolve o arquivo na mesma pasta, mostrando linha TypeScript original nos stack traces do CloudWatch.

> **Pré-requisito:** `tsc --build` deve rodar antes do `build` de cada Lambda — o esbuild lê os `.js` compilados dos pacotes de domínio via os `exports` (`./dist/service/*.js`). A resolução de workspace packages com wildcard subpath exports via esbuild deve ser validada no spike.

### scripts/build-all.sh

O script de build fica em `scripts/build-all.sh` na raiz do projeto. Ele declara a ordem explícita de build por tier, respeitando dependências — sem depender de inferência automática do npm workspaces. Aceita o argumento `ci` para modo CI (sem fund/audit, paralelismo máximo).

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ROOT_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"
APP_DIR="$ROOT_DIR/app"

# --- modo ---
CI_MODE=false
[[ "${1:-}" == "ci" ]] && CI_MODE=true

NPM_INSTALL_CMD="npm install"
$CI_MODE && NPM_INSTALL_CMD="npm install --no-fund --no-audit"

# --- paralelismo ---
CPUS=$(nproc 2>/dev/null || sysctl -n hw.logicalcpu 2>/dev/null || echo 4)
if $CI_MODE; then
  MAX_PARALLEL=$CPUS
else
  MAX_PARALLEL=$(( CPUS > 2 ? CPUS - 2 : 1 ))
  MAX_PARALLEL=$(( MAX_PARALLEL < 2 ? 2 : MAX_PARALLEL ))
fi
echo "[info]  CPUs: $CPUS — parallel jobs: $MAX_PARALLEL"

# --- build_module ---
build_module() {
  local dir="$1"
  local label="${dir#"$ROOT_DIR"/}"
  local output

  echo "[build] $label"

  if ! output=$(cd "$dir" && $NPM_INSTALL_CMD 2>&1); then
    printf '\n[FAILED] %s (npm install)\n%s\n' "$label" "$output" >&2
    return 1
  fi

  if ! output=$(cd "$dir" && npm run build 2>&1); then
    printf '\n[FAILED] %s (build)\n%s\n' "$label" "$output" >&2
    return 1
  fi

  echo "[done]  $label"
}

# --- build_tier ---
# Roda todos os módulos do tier em paralelo, respeitando MAX_PARALLEL.
# Aguarda todos antes de retornar — falha se qualquer um falhar.
build_tier() {
  local -a pids=() labels=() failed=()
  local active=0

  for dir in "$@"; do
    build_module "$dir" &
    pids+=($!)
    labels+=("${dir#"$ROOT_DIR"/}")
    (( ++active ))

    if (( active >= MAX_PARALLEL )); then
      wait "${pids[${#pids[@]} - $active]}" || true
      (( --active ))
    fi
  done

  local i=0
  for pid in "${pids[@]}"; do
    wait "$pid" || failed+=("${labels[$i]}")
    (( i++ ))
  done

  if (( ${#failed[@]} > 0 )); then
    printf '\n[ERROR] %d build(s) failed:\n' "${#failed[@]}" >&2
    printf '  - %s\n' "${failed[@]}" >&2
    exit 1
  fi
}

# --- helper ---
app() { for m in "$@"; do echo "$APP_DIR/$m"; done; }

# =============================================================
# Ordem de build — alterar ao adicionar pacotes ou dependências
# =============================================================

# Tier 1 — shared sem dependências internas
build_tier $(app shared/infra-dynamo shared/error)

# Tier 2 — shared que depende do Tier 1
build_tier $(app shared/util)

# Tier 3 — domínios (dependem de shared)
build_tier $(app product order)

# Tier 4 — orquestradores (dependem de domínios)
build_tier $(app checkout)

# Tier 5 — lambdas (esbuild a partir dos dist/ compilados)
# Auto-descoberta: qualquer package.json dentro de app/lambdas/ vira um build
mapfile -t LAMBDA_DIRS < <(
  find "$APP_DIR/lambdas" -name "package.json" \
    ! -path "*/node_modules/*" \
    -exec dirname {} \;
)
build_tier "${LAMBDA_DIRS[@]}"

echo ""
echo "[SUCCESS] All builds completed"
```

Ao adicionar um domínio ou orquestrador, incluir no tier correto. Lambdas são descobertas automaticamente via `find` — não precisam ser listadas manualmente. O `build-all.sh` não sabe o que cada pacote faz — chama `npm run build` e cada pacote se resolve.

### CDK

Usa `Function` com `Code.fromAsset()` — o CDK é só deploy, sem bundling:

```typescript
import { Function, Runtime, Code } from 'aws-cdk-lib/aws-lambda'
import * as path from 'path'

new Function(this, 'CheckoutPost', {
  runtime: Runtime.NODEJS_24_X,
  handler: 'index.handler',
  code: Code.fromAsset(
    path.join(__dirname, '../app/lambdas/http/v1/checkout/post/dist')
  ),
  environment: {
    NODE_OPTIONS: '--enable-source-maps',
  },
})
```

O artefato que sobe é previsível e inspecionável antes do deploy — `dist/index.js` pode ser lido diretamente para verificar o bundle.

---

## shared/

Pacotes independentes — um por responsabilidade. Ciclos de vida distintos: `infra-dynamo` muda com o SDK, `error` é estável, `util` muda com frequência.

```typescript
// shared/infra-dynamo/src/index.ts
import type { DynamoDBDocumentClient } from '@aws-sdk/lib-dynamodb'
import type { TransactWriteItem } from '@aws-sdk/lib-dynamodb'

export type DynamoConfig = { client: DynamoDBDocumentClient; tableName: string }

export type Transact = (items: TransactWriteItem[]) => Promise<void>
```

`DynamoConfig` é o motivo de existência deste pacote — usado por todos os repositories. `Transact` fica aqui como tipo nomeado para o service orquestrador depender.

`makeTransact` não pertence aqui — são 3 linhas sem lógica. Vai inline no handler que precisar.

---

## Tree-shaking

Com wildcard exports, cada arquivo de `repository/` e `service/` é um módulo independente. O handler importa só os que usa — esbuild inclui apenas esses arquivos e suas dependências transitivas no bundle final.

Funciona porque as factories são funções puras sem side effects no topo do módulo.

---

## Fluxo de Dependências

```
schema               ← port/repository, port/service
AWS SDK              ← port/repository, port/service (TransactWriteItem — tipo de infra sem abstração)
infra-dynamo         ← repository
infra-dynamo         ← service orquestrador (Transact — exceção intencional, ver seção abaixo)
port/repository      ← repository
port/repository      ← service de domínio (como input)
port/service         ← service de domínio (como output)
port/service         ← service orquestrador (como input)
schema + port + repository + service + shared/* ← lambdas/ (composition root)
```

Sem ciclos. `schema` não importa nada do projeto. Services orquestradores nunca importam repositories.

---

## Regras de Camada

| Camada | Pode importar | Nunca importa |
|---|---|---|
| `schema` | nada do projeto | qualquer outra camada |
| `port/repository` | schema (mesmo domínio), AWS SDK (TransactWriteItem) | repository, service, lambda |
| `port/service` | schema (mesmo domínio), AWS SDK (TransactWriteItem) | repository, service, lambda |
| `infra-dynamo` | AWS SDK | schema, port, repository, service, lambda |
| `error` | nada do projeto | qualquer outra camada |
| `util` | error | schema, port, repository, service |
| `repository` | port/repository, schema, infra-dynamo, AWS SDK | service, lambda |
| `service de domínio` | port/repository, port/service, schema, error | repository, infra-dynamo, AWS SDK, lambda |
| `service orquestrador` | port/service (qualquer domínio), error, infra-dynamo (Transact) | repository, AWS SDK, lambda |
| `lambdas/` | service/*, repository/*, infra-dynamo, util | lógica de negócio inline, instanciação dentro do handler |

**Regras invioláveis:**
- Service nunca importa repository — só port types
- Service orquestrador nunca importa repository — só service ports de outros domínios
- Nenhuma instanciação de factory dentro da função handler — apenas no nível do módulo

A regra "service nunca importa repository de outro domínio" é enforçada por **convenção e code review**, não por tooling. Com pacotes npm separados, o ESLint não consegue distinguir se um import de outro pacote é um repository ou um service — o path do pacote não carrega essa semântica.

**Por que ports importam `TransactWriteItem` direto do AWS SDK:**
`TransactWriteItem` é um tipo de infraestrutura que não é abstraído — o projeto o usa como está, sem transformação. Re-exportá-lo via `infra-dynamo` seria burocracia sem valor, criando indireção onde não há abstração real. Quando um tipo de infra não é encapsulado, importa-se da fonte diretamente.

---

## Operações Transacionais

Cada domínio encapsula a construção do seu próprio `TransactItem`. O orquestrador coleta os itens e executa via `makeTransact`.

```
repository → makeGet*TransactItem   (conhece tableName e estrutura DynamoDB)
service    → make*TransactItem      (envelopa o repository, expõe port limpo)
orquestrador → recebe port/service  (só chama, sem conhecer internos de cada domínio)
handler    → makeTransact inline    (3 linhas, sem justificar pacote compartilhado)
```

### Exceção intencional: orquestrador importa `infra-dynamo`

O service orquestrador importa `Transact` de `@<project>/infra-dynamo`. Isso viola a regra geral de que services não conhecem infra — e a violação é aceita conscientemente.

**Por quê — a restrição do DynamoDB:** o `TransactWriteCommand` do DynamoDB exige que **todos os itens da transação sejam enviados em uma única chamada de API**. Não existe `BEGIN TRANSACTION` seguido de operações independentes e depois `COMMIT`, como em bancos SQL. Quem coordena a transação precisa necessariamente: (1) coletar todos os itens, (2) conhecer como enviá-los juntos para o banco.

Isso torna impossível manter o service orquestrador completamente agnóstico de infraestrutura sem introduzir um transaction manager externo — equivalente ao JTA/XA de Java — que não existe no ecossistema Lambda/DynamoDB.

**Alternativas consideradas e descartadas:**
- *Functional core (return a plan):* o orquestrador retorna `TransactItem[]` e a Lambda executa. Elimina o import, mas a Lambda passa a ser um tradutor de domínio → infra, extrapolando o papel de composition root.
- *Tipo neutro em `@<project>/types`:* `TransactItem` e `Transact` em pacote sem AWS SDK. Funciona em teoria, mas o TypeScript exige compatibilidade estrutural precisa com o SDK na fronteira de `infra-dynamo`, na prática requer `as never` ou replicação fiel dos tipos do SDK.
- *Genérico `T` propagado:* elimina o import mas infesta todos os ports e services com `<T>`, aumentando complexidade sem ganho real de isolamento.

**Onde a regra continua valendo:** o orquestrador importa apenas `Transact` (o executor) — nunca `DynamoDBClient`, nunca `TransactWriteCommand`, nunca `tableName`. Os internos de cada domínio permanecem opacos. O acoplamento é ao *contrato de execução atômica*, não à implementação DynamoDB.

---

## Quando Não Usar Este Modelo

- Projetos majoritariamente CRUD simples sem regras cross-domain — o modelo anterior tem custo menor e funciona bem
- Times em projetos iniciais onde o domínio ainda está sendo explorado — mais fácil adotar quando o domínio já está estabilizado

---

## Origem

Decisão tomada após análise dos trade-offs do projeto GE Amizade (2026-06). Sessão de design cobriu: DI via construtor vs method injection, tree-shaking, functional core, hexagonal architecture, npm workspaces, wildcard exports, make* universal, e composition root na Lambda.
