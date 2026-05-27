---
name: vuejs
description: "Read before writing or modifying any Vue.js code."
---

# Vue.js Architecture

> Documento base para projetos Vue.js. Descreve decisões de arquitetura, convenções e responsabilidades de cada camada. Todos os exemplos assumem Vue 3 + Composition API + Vite + TypeScript. Leia antes de escrever ou modificar qualquer view, componente, composable, store ou arquivo de roteamento.
>
> **Este documento prescreve como fazer — não descreve o que já está feito.** O que vale é a regra, não o exemplo. O estado atual do projeto pode diferir; o objetivo é convergir para estas diretrizes.
>
> **Valores de domínio nos exemplos são sempre ilustrativos.** Nomes de stores, namespaces de logger, códigos de erro, nomes de rotas, estruturas de pastas — quando aparecem em exemplos, mostram o padrão a seguir, não os valores obrigatórios. O projeto define os valores concretos; o documento define a estrutura.
>
> **Sobre diretrizes que parecem específicas de projeto:** algumas decisões deste documento (como o formato do envelope HTTP ou o comportamento do cliente de autenticação) são boas práticas consolidadas, não imposições de um projeto específico. Quando não houver um documento de projeto que substitua ou complemente uma dessas diretrizes, esta é a regra vigente. Documentos numerados `30+` podem sobrescrever ou estender qualquer parte deste documento.

---

## Índice

| Seção | Conteúdo | Quando ler |
|---|---|---|
| **S1 — Filosofia e Padrões** | Princípios gerais, Composition API, Atomic Design, type safety, convenções de nomes, linting, logs, testes, padrões visuais do projeto | Sempre — qualquer task |
| **S2 — Infraestrutura** | Estrutura de pastas, camadas (view / composable / service / store / httpClient), envelope de resposta, roteamento, config, types, i18n, barrel exports | Ao criar ou alterar estrutura, novos arquivos, novas camadas |
| **S3 — Implementação** | `authStore`, stores de domínio (bootstrap/clear), guards de rota (`getToken()`), views de erro | Somente ao implementar ou dar manutenção em: auth, sessão, bootstrap, fluxo de login, views de erro |
| **S4 — Padrões de Interação** | Loading (`useAsyncState` local + overlay global), erros (filosofia, tipos, `useErrorHandler`, `useAsyncAction`), paginação (full scan / sob demanda), ações em lote, formulários (`useFormAction`, `useDestructiveAction`), estados vazios | Ao implementar qualquer view com loading, erro, lista, formulário ou ação |

**Mapeamento task → seção:**

- Criar ou editar qualquer view, componente, composable, util → **S1 + S2 + S4**
- Dúvida sobre onde colocar um arquivo ou qual camada usar → **S2**
- Implementar login, logout, bootstrap, view de erro → **S1 + S2 + S3**
- Dar manutenção em `authStore`, router guards → **S3**
- Implementar formulário → **S4** (`useFormAction`, `useDestructiveAction`)
- Implementar lista com paginação ou ações por item → **S4** (Paginação, Ações em lote)
- Implementar loading, erro ou estado vazio → **S4** (`useAsyncState`, `useErrorHandler`, `useAsyncAction`)
- Implementar mock de service → **S2** (Service + regras de mock)

---

## S1 — Filosofia e Padrões

### Filosofia

As mesmas decisões que guiam o backend, adaptadas para o frontend:

**Produção primeiro.** Toda decisão é tomada pensando no comportamento em produção. Conforto de desenvolvimento (mocks, dados fixos, rotas de atalho) é temporário e nunca deve chegar à versão final.

**Consistência elimina decisão.** Onde buscar dados, como tratar erros, onde guardar estado, como estruturar a UI — essas respostas são sempre as mesmas.

**Explícito sobre implícito.** Tipos declarados em interfaces nomeadas. Nenhum campo inferido de `any`. Estado global declarado em stores com tipos explícitos. Props tipadas com `defineProps<T>()`.

**Mock é scaffolding, não arquitetura.** Mocks existem para permitir que o frontend avance antes do backend estar pronto. Todo mock tem uma data de validade: o momento em que o endpoint real for entregue. Nunca adicione lógica de negócio em um mock.

**Reutilizar antes de criar.** Antes de escrever qualquer coisa, verificar se já existe no projeto — ou nas libs instaladas — um componente, composable, util, service ou tipo que resolva o problema.

**Não existe? Pergunte antes de criar.** Se o componente necessário não existe no projeto nem nas libs instaladas, não crie por conta própria. Pergunte ao usuário: o que esse componente deve fazer, como deve se comportar, quais variações precisa suportar, onde será usado. Criar um componente sem essas respostas é prototipar — e este não é um projeto de prototipação.

**Projeto real, não protótipo.** A solução mais curta raramente é a correta. O critério não é "funciona agora" — é "funciona bem, é consistente e pode ser mantido". Atalhos que economizam minutos criam horas de retrabalho.

**Separação de responsabilidade.** Cada camada tem uma responsabilidade e não ultrapassa sua fronteira. A view não faz chamadas HTTP. O service não conhece roteamento. A store não conhece a API. O componente não conhece a store.

**UI construída em camadas.** A interface segue o modelo Atomic Design: atoms formam molecules, molecules formam organisms, organisms compõem layouts, layouts sustentam views. Nenhuma camada pula um nível. Essa hierarquia garante que mudar um atom propaga a mudança para todo o sistema de forma previsível.

---

### Composition API

Sempre use a Composition API com `<script setup lang="ts">`. Nunca use a Options API.

```vue
<!-- wrong -->
<script lang="ts">
export default {
  data() { return { name: '' } },
  methods: { save() { /* ... */ } }
}
</script>

<!-- correct -->
<script setup lang="ts">
import { ref } from 'vue'

const name = ref('')

function save(): void { /* ... */ }
</script>
```

`<script setup>` é a forma canônica do Vue 3. Oferece melhor inferência de tipos, menos boilerplate e melhor tree-shaking.

**Regras de reatividade:**

- Use `ref()` para primitivos e `reactive()` para objetos quando o objeto inteiro precisa de reatividade
- Prefira `ref()` na maioria dos casos — é mais consistente e evita o problema de desestruturação de `reactive()`
- Use `computed()` para valores derivados — nunca recalcule no template
- Use `watch()` e `watchEffect()` com parcimônia — na maioria dos casos, `computed()` resolve

```typescript
// correct — ref para primitivo, computed para valor derivado
const items  = ref<Item[]>([])
const isEmpty = computed(() => items.value.length === 0)
```

---

### TypeScript

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "jsx": "preserve",
    "lib": ["ESNext", "DOM"],
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts", "src/**/*.vue"]
}
```

#### Path alias

Use `@/` como alias para `src/`. Nunca use caminhos relativos com `../` para importar fora do diretório atual.

```typescript
// wrong
import { userService } from '../../../services/user/userService'

// correct
import { userService } from '@/services/user/userService'
```

Configure o alias no `vite.config.ts`:

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { resolve } from 'node:path'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': resolve(import.meta.dirname, 'src'),
    },
  },
})
```

---

### Atomic Design

A UI do projeto segue o modelo **Atomic Design**. Cada peça de interface pertence a exatamente um nível, e esse nível define seu contrato de responsabilidade.

| Nível        | Onde                   | Descrição                                                                                                                                              |
| ------------ | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Atom**     | `components/atom/`     | Peça mínima e indivisível. Sem composição de outros componentes do projeto. Exemplos: `Button.vue`, `Badge.vue`, `Input.vue`.                          |
| **Molecule** | `components/molecule/` | Composição de atoms com um propósito único. Exemplos: `SearchBar.vue` (input + botão), `ListItem.vue` (avatar + texto + badge).                        |
| **Organism** | `components/organism/` | Seção completa e autossuficiente, composta de atoms e molecules. Exemplos: `UserCard.vue`, tabela com header e paginação.                              |
| **Layout**   | `layouts/`             | Esqueleto estrutural da view — define onde ficam header, conteúdo e footer. Não contém dados. Exemplos: `ListLayout.vue`, `FormLayout.vue`.            |
| **View**     | `views/`               | A página em si. Usa um layout, compõe organisms/molecules/atoms, busca dados via services e lida com roteamento.                                       |

#### Regras

- Cada nível só pode compor elementos do seu nível ou de níveis inferiores. Um atom não usa molecules; uma molecule não usa organisms.
- Dentro de cada nível (`atom/`, `molecule/`, `organism/`), os componentes são organizados em subdiretórios por categoria funcional (ex: `atom/buttons/`, `atom/badges/`).
- Nenhum componente abaixo de View conhece o router, stores ou services diretamente.
- Antes de criar qualquer componente novo, verificar se já existe um no nível adequado que resolva o problema.
- **Componentes de libs podem ser usados em qualquer nível.** Atoms, molecules e organisms podem usar diretamente componentes das libs instaladas.
- **Só crie um componente se for uma especialização.** Se a lib já oferece um botão, use-o diretamente. Crie um `atom/buttons/BtnConfirm.vue` apenas se houver comportamento ou estilo específico do projeto que justifique a especialização. Wrappers sem propósito são proibidos.

---

### Type safety

#### Sem `any`

Nunca use `any`. Se o tipo não é conhecido, use `unknown` e faça narrowing antes de usar o valor.

```typescript
// wrong
const data = response as any

// correct
function isUser(value: unknown): value is User {
  return typeof value === 'object' && value !== null && 'id' in value
}
```

#### Props tipadas

Sempre tipar props com `defineProps<T>()`. Nunca use a forma de array ou objeto sem tipos.

```typescript
// wrong
const props = defineProps(['name', 'age'])

// correct
interface Props {
  name:  string
  age?:  number
}
const props = defineProps<Props>()
```

#### Emits tipados

Sempre tipar emits com `defineEmits<T>()`.

```typescript
// correct
const emit = defineEmits<{
  save:   [data: FormData]
  cancel: []
}>()
```

---

### Nomes de arquivos e convenções

**Componentes Vue usam `PascalCase.vue`.** O Vue exige que componentes com letra maiúscula sejam tratados como componentes customizados, não elementos HTML.

**Views encapsulam a hierarquia de acesso.** O nome começa com o nível de acesso (`Public` ou `Private`), seguido do domínio e da ação — formando uma cadeia autodocumentada que elimina colisões entre views de domínios diferentes.

```
Public  + Auth  + Login       → PublicAuthLoginView.vue
Public  + Error + Generic     → PublicErrorGenericView.vue
Private + Admin + Users       → PrivateAdminUsersView.vue
Private + Admin + Users + Add → PrivateAdminUsersAddView.vue
```

**Arquivos:**

| Tipo        | Convenção                                                                | Exemplo                                        |
| ----------- | ------------------------------------------------------------------------ | ---------------------------------------------- |
| View        | `PascalCase` + `View.vue` seguindo hierarquia de acesso                  | `PrivateAdminUsersView.vue`                    |
| Atom        | `PascalCase.vue` dentro de `atom/<categoria>/`                           | `atom/buttons/BtnBack.vue`                     |
| Molecule    | `PascalCase.vue` dentro de `molecule/<categoria>/`                       | `molecule/cards/EventCard.vue`                 |
| Organism    | `PascalCase.vue` dentro de `organism/<categoria>/`                       | `organism/lists/UserList.vue`                  |
| Layout      | `PascalCase` + `Layout.vue` dentro de `layouts/<categoria>/`             | `layouts/forms/FormLayout.vue`                 |
| Store       | `camelCase` + `Store.ts`                                                 | `authStore.ts`                                 |
| Service     | `camelCase` + `Service.ts`                                               | `userService.ts`                               |
| Mock        | `camelCase` + `Service.mock.ts`                                          | `userService.mock.ts`                          |
| Composable  | `camelCase` com prefixo `use` + `.ts` dentro de `composables/<domínio>/` | `composables/commons/useAsyncAction.ts`        |
| Util        | `camelCase` descrevendo a operação + `.ts` dentro de `utils/<domínio>/`  | `utils/dateTime/formatter.ts`                  |
| Tipo        | `camelCase.ts` dentro de `types/<domínio>/`                              | `types/user/user.model.ts`                     |
| Config      | `camelCase.ts`                                                           | `config/index.ts`                              |

**Identificadores no código:**

- `camelCase` — variáveis, funções, parâmetros, propriedades de objetos, exports de services e stores
- `PascalCase` — componentes Vue (obrigação do framework), interfaces e type aliases
- Nomes descritivos e autoexplicativos — evite abreviações

---

### Linting

Todo projeto deve ter ESLint configurado com `typescript-eslint` e `eslint-plugin-vue`:

```javascript
// eslint.config.js
import tseslint from 'typescript-eslint'
import pluginVue from 'eslint-plugin-vue'
import importPlugin from 'eslint-plugin-import'

export default tseslint.config(
  ...tseslint.configs.strictTypeChecked,
  ...pluginVue.configs['flat/recommended'],
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
        extraFileExtensions: ['.vue'],
      },
    },
    plugins: { import: importPlugin },
    rules: {
      '@typescript-eslint/consistent-type-imports':         ['error', { prefer: 'type-imports' }],
      '@typescript-eslint/no-import-type-side-effects':     'error',
      '@typescript-eslint/explicit-function-return-type':   ['error', { allowExpressions: true }],
      '@typescript-eslint/explicit-module-boundary-types':  'error',
      '@typescript-eslint/no-unused-vars': ['error', {
        vars: 'all',
        args: 'all',
        argsIgnorePattern: '^_',
        varsIgnorePattern: '^_',
      }],
      'vue/component-api-style':      ['error', ['script-setup']],
      'vue/block-lang':               ['error', { script: { lang: 'ts' } }],
      'vue/define-props-declaration': ['error', 'type-based'],
      'vue/define-emits-declaration': ['error', 'type-based'],
      'import/no-duplicates':         'error',
      'import/order': ['error', {
        groups: ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'],
        pathGroups: [{ pattern: '@/**', group: 'internal' }],
        'newlines-between': 'always',
      }],
    },
  },
  { ignores: ['dist/'] }
)
```

Install required plugins:

```
npm install -D typescript-eslint eslint-plugin-vue eslint-plugin-import
```

Erros de lint não são warnings — são erros. O CI rejeita código com erros de lint.

---

### Logs

Todo log do projeto passa pelo `logger` — nunca use `console.log`, `console.info` ou `console.warn` diretamente no código. Logs diretos no console não têm controle de nível, não têm namespace e não podem ser desligados em produção.

```typescript
// src/utils/logger/logger.ts
const base = createLogger({ level: import.meta.env.PROD ? 'warn' : 'debug' })

export const logger = {
  auth:    base.extend('AUTH'),
  service: base.extend('SERVICE'),
  store:   base.extend('STORE'),
}
```

```typescript
// uso
import { logger } from '@/utils/logger/logger'

logger.auth.debug('bootstrap started')
logger.service.error('failed to fetch users', error)
```

A biblioteca de logging é definida pelo projeto. O padrão acima mostra a interface esperada — namespaces por camada, níveis configuráveis por ambiente.

Regras:

- Nunca commitar `console.log` — o linter deve bloquear
- `logger.error` e `logger.warn` ativos em produção (enviados para serviço de monitoramento)
- `logger.debug` e `logger.info` silenciados em produção por padrão

---

### Testes

Use Vitest como test runner. Integra com Vite nativamente e entende TypeScript e ESM sem configuração adicional.

**Filosofia de testes:**

- Teste no nível mais alto possível. O teste correto para uma view é um teste E2E que simula o usuário usando o app — cobre view, store e service ao mesmo tempo.
- Testes de componente em isolamento raramente compensam o custo de manutenção.

| O que testar          | Nível | Justificativa                                                                 |
| --------------------- | ----- | ----------------------------------------------------------------------------- |
| `utils/`              | Unit  | Funções puras, fácil de testar e alto valor                                   |
| `composables/`        | Unit  | Lógica reutilizável, testável sem DOM completo                                |
| Fluxos de view        | E2E   | Cobre tudo de uma vez, é o "usuário usando"                                   |
| Componentes isolados  | —     | Evitar — E2E já cobre, e testes de componente tendem a testar implementação   |

A ferramenta de E2E (Playwright ou Cypress) é definida pelo projeto no arquivo `30+`.

---

### Padrões visuais do projeto (`XX-ui-patterns.md`)

Este documento cobre arquitetura e princípios — decisões que valem para qualquer projeto Vue.js. Decisões visuais específicas do projeto pertencem a um documento separado, numerado na faixa `30+`, seguindo o índice em `00-index.md`.

**Todo projeto que usa este documento base deve criar seu próprio arquivo de padrões visuais.** Sem ele, cada view inventa sua própria resposta para situações recorrentes — e a inconsistência é inevitável.

Esse arquivo cobre as decisões que se repetem em toda view mas que dependem do design do projeto. Exemplos do que precisa ser definido lá:

- Como exibir estado de carregamento
- Como exibir erros ao usuário
- Como exibir listas vazias
- Como estruturar formulários e exibir erros de validação
- Componentes visuais recorrentes (`ConfirmDialog`, `EmptyState`, `SkeletonCard`)

A lista não é fechada. Sempre que uma situação recorrente não tiver resposta definida, a decisão vai para esse arquivo — não fica implícita no código.

---

## S2 — Infraestrutura

### Instalação de pacotes

Sempre instale pacotes sem especificar versão:

```
npm install <package>
```

- Nunca edite o `package.json` manualmente para adicionar uma dependência com versão fixada
- Especificar versão (`npm install <package>@x.y.z`) só é permitido quando um humano solicitar explicitamente, ou quando outra dependência já instalada exigir aquela versão como peer dependency

Fixar uma versão instala um pacote desatualizado em vez da versão atual, acumulando vulnerabilidades de segurança ao longo do tempo. Todo agente de IA que escreve uma versão diretamente no `package.json` sem executar `npm install` introduz esse risco.

---

### Estrutura de pastas

```
src/
├── components/       ← peças de UI sem lógica de negócio (Atomic Design)
│   ├── atom/
│   │   ├── buttons/
│   │   ├── badges/
│   │   └── inputs/
│   ├── molecule/
│   │   ├── cards/
│   │   └── list-items/
│   └── organism/
│       └── ...
├── layouts/          ← templates estruturais de view
│   ├── forms/        ← FormLayout.vue
│   └── list/         ← ListLayout.vue
├── views/            ← views espelhando a hierarquia de rotas
│   ├── public/
│   │   ├── auth/     ← PublicAuthLoginView.vue
│   │   └── error/    ← PublicErrorGenericView.vue, PublicErrorBootstrapView.vue
│   └── private/
│       └── <domain>/ ← Private<Domain><Action>View.vue
├── router/           ← Vue Router organizado por hierarquia de acesso
│   ├── index.ts      ← createRouter, guards globais
│   ├── public.ts     ← rotas públicas
│   └── private.ts    ← rotas privadas (meta: { requiresAuth: true })
├── stores/           ← Pinia stores por entidade
├── services/         ← chamadas à API (real + mock)
│   └── api/          ← clientes HTTP
├── composables/      ← lógica com estado reutilizável (equivalente a hooks)
│   └── commons/      ← composables genéricos sem domínio específico
├── config/           ← variáveis de ambiente — um único index.ts
├── types/            ← interfaces e tipos TypeScript, subdiretório por domínio
│   └── router/       ← declaração de RouteMeta
├── utils/            ← funções puras sem reatividade Vue
│   ├── dateTime/
│   └── string/
└── locales/          ← arquivos de tradução i18n
    ├── pt-BR.json
    └── en.json
```

---

### Camadas e responsabilidades

#### View (`views/`)

A view é responsável por:

- Renderizar a UI com base no estado local e nas stores
- Chamar services para buscar ou modificar dados
- Tratar estados de loading e erro locais
- Navegar entre rotas

A view **nunca**:

- Faz chamadas HTTP diretamente
- Contém lógica de negócio ou transformação de dados
- Conhece os detalhes de implementação do service

```typescript
// wrong — view chamando fetch diretamente
const data = await fetch('/v1/users')

// correct — view chamando o service
const users = await userService.list()
```

**Tratamento de erros na view.** Toda chamada a um service deve estar dentro de um `try/catch`. A view decide o que fazer com o erro:

- Erro de negócio (validação falhou, recurso não encontrado) → `ref` local, feedback inline ao usuário
- Erro crítico (falha irrecuperável, estado inválido) → `useReportCriticalError()` → `errorStore.setError(err)` + `router.push({ name: 'public-error-generic' })`

```typescript
const reportCriticalError = useReportCriticalError()

try {
  users.value = await userService.list()
} catch (err) {
  if (err instanceof ApiError && err.code === 'NOT_FOUND') {
    errorMessage.value = t('user.notFound')  // erro esperado — inline
  } else {
    reportCriticalError(err)  // erro inesperado — errorStore + redirect
  }
}
```

#### Service (`services/`)

O service é responsável por:

- Fazer chamadas HTTP via `request()` do `httpClient`
- Encapsular a URL, método e payload de cada endpoint
- Retornar os dados tipados para a view

O service **nunca**:

- Conhece o router ou estado da UI
- Conhece stores — não importa nem acessa nenhuma store
- Contém lógica de negócio de domínio
- É chamado por outro service

O padrão de service tem dois arquivos, organizados em subdiretórios por domínio:

```
services/
├── api/
│   ├── backend.httpClient.ts     ← backend principal
│   └── erp.httpClient.ts         ← backend ERP (quando necessário)
├── auth/
│   ├── authService.ts
│   └── authService.mock.ts
└── user/
    ├── userService.ts
    └── userService.mock.ts
```

Views importam **apenas** o service real. O mock é detalhe de implementação do service:

```typescript
// userService.ts — enquanto o backend não existe
import { userServiceMock } from './userService.mock'

export const userService = {
  // TODO: backend não implementado — substituir quando endpoint estiver pronto
  list: () => userServiceMock.list(),
}
```

Quando o endpoint estiver pronto, o mock é removido e o arquivo `.mock.ts` é deletado:

```typescript
// userService.ts — backend implementado
import { backendHttpClient } from '@/services/api/backend.httpClient'

export const userService = {
  list: () => backendHttpClient.request<User[]>('/v1/users'),
}
```

**Regras de mock:**

- **Separação obrigatória:** lógica de mock nunca entra no arquivo do service real. O arquivo `.mock.ts` é o único lugar onde mocks existem. O service real só delega para ele temporariamente via `import`.
- Todo uso de mock tem um comentário `// TODO: backend não implementado`
- Se o endpoint existe, não há justificativa para manter o mock — é débito técnico
- Quando todos os endpoints de um service estiverem implementados, o arquivo `.mock.ts` é deletado
- A migração é feita **apenas** no service real — views não mudam

Como complemento, a variável `VITE_USE_MOCKS=true` pode ser usada para tornar o uso de mocks explícito por ambiente — útil para forçar erro em runtime quando um service ainda delega para o mock em um ambiente onde não deveria.

#### HTTP clients (`services/api/`)

Cada arquivo em `services/api/` é um cliente HTTP dedicado a um backend. Um cliente:

- Injeta o JWT no header `Authorization` lendo o token diretamente da `authStore` (em memória) — nunca do `localStorage`. O `localStorage` é acessado apenas na `authStore.bootstrap()`.
- Desembrulha o envelope de resposta da API e lança erro quando a resposta indica falha
- Chama o handler de 401 registrado pelo router guard (quando aplicável)

**Envelope padrão de resposta** — este é o modelo base; o documento `30+` do projeto define a estrutura real caso o backend divirja:

```typescript
// sucesso (lista)
{ data: T[], pagination: { pageSize: number, nextCursor: string | null, hasNext: boolean, ... } }

// sucesso (item único)
{ data: T }

// erro
{ "error": "ENTITY_NOT_FOUND", "message": "Entity abc-123 not found" }
```

- `error` — código `UPPER_SNAKE_CASE`, estável e comparável programaticamente. Quando vindo do backend, passa por `t()` para exibir ao usuário.
- `message` — texto legível por humanos, pode mudar — não usar para lógica.

O `httpClient` lê a `authStore` diretamente — isso é um acoplamento tolerável sem alternativa viável. Passar o token como parâmetro em cada chamada polui todos os services. O cliente é infraestrutura, não serviço de domínio, e essa é a única exceção à regra de que camadas de infraestrutura não conhecem stores.

Nenhum arquivo fora de `services/` importa um cliente HTTP diretamente.

#### Store (`stores/`)

Stores Pinia armazenam estado de sessão — dados que precisam estar disponíveis em qualquer view sem prop drilling.

Sempre use o estilo **setup store** (Composition API) no Pinia — nunca o estilo options:

```typescript
// stores/authStore.ts
import { defineStore } from 'pinia'
import { ref } from 'vue'

export const useAuthStore = defineStore('auth', () => {
  const token = ref<string | null>(null)

  function setToken(newToken: string): void {
    token.value = newToken
  }

  function clearToken(): void {
    token.value = null
  }

  return { token, setToken, clearToken }
})
```

Exemplos de stores típicas:

- `authStore` — token JWT e identificadores da sessão
- `accountStore` — dados da conta carregada após login
- `errorStore` — erro crítico atual, usado para redirecionar o usuário a uma view de erro
- `loadingStore` — estado do overlay global de loading

> **`errorStore` e `loadingStore` são opcionais.** Existem para projetos que adotam tratativas globais de erro e loading — overlay que bloqueia toda a interface, view de erro centralizada. Nem todo projeto precisa disso. O documento `30+` do projeto deve avaliar se essas stores são necessárias, como são estruturadas e quais componentes as consomem. Sem essa decisão documentada, não as crie por conta própria.

**Quando usar store:**

- Dado compartilhado entre views com um único ponto de modificação (ex: nome do usuário — qualquer view lê, só o login altera)
- Dado de sessão que precisa estar disponível em qualquer ponto do app

**Quando não usar store:**

- Dado local da view (loading, erro, valores de formulário) → `ref` local
- Dado que a view busca para si mesma → `ref` + service
- Dado passado entre views via navegação → params de rota. Params identificam e orientam — não transportam estado. Passar `id`, `type` ou campos que definem o comportamento da próxima view é correto. Passar um objeto completo de domínio via router state é proibido: some com F5 e cria acoplamento implícito entre views.

Regras gerais:

- Cada store usa setup store com `ref` tipados explicitamente
- Stores não chamam o `httpClient` diretamente — toda comunicação com a API é delegada aos services
- `localStorage` e `sessionStorage` são acessados exclusivamente por stores
- Erros que a store não sabe resolver são sempre propagados ao chamador

#### Composables (`composables/`)

Composables são funções que usam APIs de reatividade do Vue internamente (`ref`, `reactive`, `watch`, etc.). São o lugar para lógica com estado reutilizável entre views ou componentes.

```
composables/
├── commons/          ← composables genéricos sem domínio específico
│   ├── useAsyncAction.ts
│   ├── useErrorHandler.ts
│   └── useReportCriticalError.ts
└── <domain>/         ← composables específicos de um domínio
    └── useUserFilter.ts
```

**Quando criar um composable:** quando a mesma lógica com estado aparecer em mais de um lugar. Se a lógica existe em uma view só, fica na view.

**Composables não são `utils/`.** Se a função não usa nenhuma API de reatividade do Vue, é uma função pura e vai para `utils/`.

#### Utils (`utils/`)

Utils são funções puras sem reatividade Vue — formatação, validação, parsing, cálculos. Podem ser usadas em qualquer contexto.

```
utils/
├── dateTime/
│   ├── formatter.ts
│   └── validator.ts
└── string/
    ├── formatter.ts
    └── parser.ts
```

O subdiretório define o domínio. O arquivo define a operação (`formatter`, `validator`, `parser`, `calculator`). Juntos são auto-explicativos sem repetição.

**Se precisar de reatividade Vue dentro, não é util — é composable.**

---

### Roteamento (`router/`)

Use Vue Router. Organize as rotas em dois arquivos separados por nível de acesso:

```typescript
// router/public.ts
import type { RouteRecordRaw } from 'vue-router'

export const publicRoutes: RouteRecordRaw[] = [
  {
    path: '/login',
    name: 'public-auth-login',
    component: () => import('@/views/public/auth/PublicAuthLoginView.vue'),
  },
  {
    path: '/error',
    name: 'public-error-generic',
    component: () => import('@/views/public/error/PublicErrorGenericView.vue'),
  },
]
```

```typescript
// router/private.ts
import type { RouteRecordRaw } from 'vue-router'

export const privateRoutes: RouteRecordRaw[] = [
  {
    path: '/',
    meta: { requiresAuth: true },
    component: () => import('@/layouts/PrivateLayout.vue'),
    children: [
      {
        path: 'users',
        name: 'private-admin-users',
        component: () => import('@/views/private/admin/PrivateAdminUsersView.vue'),
      },
    ],
  },
]
```

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import { useAuthStore } from '@/stores/authStore'
import { publicRoutes } from './public'
import { privateRoutes } from './private'

const router = createRouter({
  history: createWebHistory(),
  routes: [...publicRoutes, ...privateRoutes],
})

router.beforeEach((to) => {
  const authStore = useAuthStore()
  if (to.meta.requiresAuth && !authStore.getToken()) {
    return { name: 'public-auth-login' }
  }
})

export default router
```

Regras:

- **Nomes de rota em `kebab-case`** — encapsulam a hierarquia de acesso: `private-admin-users`
- **Lazy loading obrigatório** — toda view usa `() => import(...)`. Nunca import estático em rotas.
- **Views de erro dentro das rotas públicas** — erros críticos não requerem autenticação
- O guard global é o único responsável por redirecionar rotas protegidas — nunca verificar autenticação dentro de uma view
- O guard usa `authStore.getToken()`, não `authStore.token`. Em recarga de página, o token ainda não foi carregado em memória — `getToken()` cai no `localStorage` como fallback. Usar `authStore.token` diretamente causaria redirecionamento incorreto para login antes do bootstrap terminar.
- Declare meta de rota em `types/router/meta.ts` para que `to.meta.requiresAuth` seja tipado

```typescript
// types/router/meta.ts
import 'vue-router'

declare module 'vue-router' {
  interface RouteMeta {
    requiresAuth?: boolean
  }
}
```

---

### Config (`config/index.ts`)

Variáveis de ambiente que mudam entre builds ficam em um único arquivo. Nenhum outro arquivo acessa `import.meta.env` diretamente.

```typescript
// src/config/index.ts
export const API_BASE_URL       = import.meta.env.VITE_API_URL ?? 'http://localhost:3000'
export const API_TIMEOUT        = 10_000
export const PAGE_SIZE          = Number(import.meta.env.VITE_PAGE_SIZE ?? 20)
export const SKELETON_COUNT     = Number(import.meta.env.VITE_SKELETON_COUNT ?? 5)
export const TOAST_VISIBILITY_MS = Number(import.meta.env.VITE_TOAST_VISIBILITY_MS ?? 30_000)
```

Prefixo `VITE_` é obrigatório para que o Vite exponha a variável no cliente.

---

### Types (`types/`)

O padrão é sempre criar tipos em `types/`, com subdiretório por domínio.

**Proibido usar `index.ts` como barrel export em qualquer parte do `src/`.** Essa regra é global — vale para `types/`, `components/`, `composables/`, `services/`, `utils/` e demais diretórios. O caminho do arquivo é informação semântica: `import { User } from '@/types/user/user.model'` diz que é o modelo de domínio. `import { User } from '@/types/user/user.api'` diz que é um tipo de contrato com a API. Um `index.ts` que re-exporta tudo apaga essa informação, dificulta tree-shaking e aumenta o risco de imports circulares.

```
types/
├── router/
│   └── meta.ts            ← declaração de RouteMeta
├── user/
│   ├── user.model.ts      ← modelo de domínio (o que o app usa internamente)
│   └── user.api.ts        ← tipos de request/response da API
└── event/
    ├── event.model.ts
    └── event.api.ts
```

**Tipo local é exceção, não regra.** Use tipo local apenas quando for verdadeiramente interno a um único arquivo e sem utilidade fora dele. Se existe qualquer chance do tipo ser usado em outro arquivo — view, store, service, componente — ele vai para `types/`.

---

### Internacionalização (i18n)

Use `vue-i18n` para suporte a múltiplos idiomas. O idioma atual é armazenado em uma Pinia store — nunca gerenciado apenas localmente.

```
npm install vue-i18n
```

**Bootstrap do i18n:**

```
authStore.bootstrap()
  → languageStore.bootstrap()  ← carrega idioma salvo do localStorage + aplica via i18n
  → accountStore.bootstrap()
  → ...
```

Organização das chaves por domínio:

```json
{
  "validation": {
    "required": "Campo obrigatório.",
    "email":    "Email inválido.",
    "min":      "Mínimo {{count}} caracteres."
  },
  "action": {
    "save":   "Salvar",
    "cancel": "Cancelar",
    "delete": "Excluir",
    "add":    "Adicionar"
  }
}
```

**Regra obrigatória:** nenhum texto visível ao usuário é hardcoded. Todo string definido no frontend passa por `t()`. Isso inclui: labels, placeholders, títulos de view, textos de botão, mensagens de estado vazio, toasts de feedback.

**Conteúdo vindo do backend — caso a caso:**

- Backend enviou um **código de erro** (ex: `"USER_NOT_FOUND"`) → passar por `t()`, traduzindo o código para uma mensagem legível
- Backend enviou **dados de domínio** (nome de usuário, título de produto, descrição) → exibir diretamente, sem `t()`. São valores inseridos por usuários, não constantes do sistema — não faz sentido traduzi-los.

```vue
<script setup lang="ts">
import { useI18n } from 'vue-i18n'
const { t } = useI18n()
</script>

<template>
  <input :placeholder="t('user.name')" />
  <button>{{ t('action.save') }}</button>
</template>
```

---

## S3 — Implementação

> Leia esta seção apenas ao implementar ou dar manutenção em: auth, sessão, bootstrap, fluxo de login, views de erro crítico.

### authStore (`stores/authStore.ts`)

O `authStore` é o repositório da sessão e o orquestrador da autenticação e inicialização das stores.

Responsabilidades:

- Guardar e expor o token JWT em memória
- Implementar `getToken()`: verifica memória primeiro — se não encontrar, busca no `localStorage`, salva em memória e retorna. Se não encontrar em nenhum dos dois, retorna `null`.
- Implementar `login(credentials)`: chama `authService.login()`, persiste o token no `localStorage` e em memória, retorna sucesso ou propaga a exceção para a view tratar
- Implementar `bootstrap()`: chama `bootstrap()` em cada store que precisa ser inicializada — chamado pela `PrivatePostLoginLoadingView`
- Implementar `clearAuth()`: limpa token do `localStorage` e da memória, e aciona `clear()` nas demais stores

**Fluxo de login:**

```
PublicAuthLoginView — usuário submete formulário
  → authStore.login(credentials)
    → authService.login(credentials)
      → sucesso → token salvo no localStorage e no authStore
        → authStore retorna sucesso para PublicAuthLoginView
          → router.push({ name: 'private-post-login-loading' })
            → PrivatePostLoginLoadingView chama authStore.bootstrap()
              → sucesso → router.push({ name: 'private-home' })
              → falha UNAUTHORIZED → guard de rota do 401 já limpou a sessão
                                     e redirecionou para public-auth-login
              → outra falha → reportCriticalError(err)
                               → PublicErrorBootstrapView
      → falha → exceção propagada → PublicAuthLoginView trata (feedback inline)
```

---

### Stores de domínio

Cada store de domínio (ex: `accountStore`, `personStore`) é autossuficiente na sua inicialização.

Responsabilidades:

- Guardar e expor os dados do seu domínio
- Implementar `bootstrap()`: chama seu próprio service para buscar e popular os dados
- Implementar `clear()`: limpa os dados ao fazer logout

```typescript
// padrão de interface para stores inicializáveis
interface DomainStore {
  bootstrap(): Promise<void>
  clear(): void
}
```

O `authStore.bootstrap()` conhece quais stores precisam ser inicializadas:

```typescript
async function bootstrap(): Promise<void> {
  await languageStore.bootstrap()
  await accountStore.bootstrap()
  await personStore.bootstrap()
}
```

**Regra:** toda store com `bootstrap()` **deve** ser adicionada explicitamente ao `authStore.bootstrap()`. Essa lista é a fonte de verdade das dependências de inicialização do app. Criar uma store com `bootstrap()` sem adicioná-la aqui é um erro — a store começará vazia e a view quebrará silenciosamente.

O `authStore` conhecer as stores de domínio é um acoplamento intencional e defensivo — torna as dependências de inicialização explícitas e visíveis em um único lugar. A lista é pequena por design: apenas as stores estritamente necessárias para o app funcionar são inicializadas no bootstrap. Stores de domínio que não são críticas na carga inicial não entram aqui.

---

### Guards de rota e sessão

O guard global em `router/index.ts` é o único responsável por redirecionar rotas protegidas.

**`authStore.clearAuth()` é chamado em dois contextos:**

- **`httpClient`** — ao receber 401, o cliente chama `authStore.clearAuth()` e o guard redireciona para login
- **View de logout** — em resposta ao logout explícito do usuário

Nenhum outro service, composable ou view chama `clearAuth()` diretamente.

---

### Views de erro

**`PublicErrorBootstrapView`** — exclusiva para falhas no bootstrap da sessão. Exibe o erro e oferece um botão de retry que navega para `PrivatePostLoginLoadingView`.

```
PrivatePostLoginLoadingView chama authStore.bootstrap()
  → falha → reportCriticalError(err)
              → errorStore.setError(err)
              → router.push({ name: 'public-error-bootstrap' })
                → usuário aciona retry → router.push({ name: 'private-post-login-loading' })
                  → PrivatePostLoginLoadingView chama authStore.bootstrap() novamente
```

**`PublicErrorGenericView`** — usada por qualquer view do app para erros inesperados não cobertos por uma view específica. Lê o erro do `errorStore`.

**O que toda view de erro deve fazer:**

- Informar que o sistema apresentou um erro e orientar o usuário (tentar novamente ou contactar o suporte)
- Oferecer a ação de recuperação adequada ao contexto
- Exibir detalhes técnicos do erro (código, timestamp) para apoiar o suporte
- **Nunca expor dados sensíveis** — token JWT, senhas e dados pessoais nunca aparecem

---

## S4 — Padrões de Interação

> Leia esta seção ao implementar qualquer view com loading, erro, lista vazia ou formulário.

---

### Loading

O projeto tem dois contextos de loading com comportamentos distintos.

O projeto tem dois contextos de loading com ferramentas distintas.

#### Loading local — busca inicial de dados

Use `useAsyncState` do VueUse. Ele expõe `isLoading`, `state` e `error` reativos, elimina o boilerplate de `ref<boolean>` + try/catch e é o padrão para busca de dados na montagem da view.

```
npm install @vueuse/core
```

**Views com formulário ou conteúdo único** — exibe spinner centralizado enquanto aguarda:

```vue
<script setup lang="ts">
import { useAsyncState } from '@vueuse/core'

const { state: user, isLoading } = useAsyncState(() => userService.get(id), null)
</script>

<template>
  <div v-if="isLoading" class="loading-center">
    <Spinner />
  </div>
  <div v-else><!-- conteúdo --></div>
</template>
```

**Views com lista** — exibe `SKELETON_COUNT` repetições do `SkeletonCard`:

```vue
<template>
  <template v-if="isLoading">
    <SkeletonCard v-for="i in SKELETON_COUNT" :key="i" />
  </template>
  <template v-else><!-- lista --></template>
</template>
```

`SKELETON_COUNT` vem de `config/index.ts`. `SkeletonCard` fica em `components/atom/feedback/SkeletonCard.vue`.

#### Overlay global — ações do usuário

Use `useAsyncAction` (composable interno do projeto) quando o usuário executa uma ação que chama a API (salvar, deletar, login, logout, etc.). Bloqueia a interface inteira via `loadingStore`, impedindo duplo-clique e navegação acidental durante a operação.

`useAsyncState` e `useAsyncAction` **não são intercambiáveis:**

| | `useAsyncState` (VueUse) | `useAsyncAction` (interno) |
|---|---|---|
| Quando usar | Busca inicial de dados na montagem | Ação disparada pelo usuário |
| Overlay global | Não | Sim |
| Classificação de erro | Não | Sim (rede / negócio / crítico) |

**Regra:** toda operação `async` disparada pelo usuário usa `useAsyncAction`. Sem exceção.

Operações síncronas de store (setters, clearers) **não** usam o overlay — são instantâneas.

---

### Erros

#### Filosofia

**A view é quem decide o que fazer com cada erro.** Erros sobem da camada de service até a store, e da store até a view — nenhuma camada intermediária silencia erros que não são de sua responsabilidade.

Na view, a decisão segue uma hierarquia simples:

1. **Erro de domínio reconhecido** — a view conhece o código do erro e sabe exatamente como reagir: mostrar mensagem inline em um campo, exibir toast específico, redirecionar para outra rota. A view trata e sinaliza que o erro foi tratado.
2. **Erro genérico** — rede, timeout, erro desconhecido, erro de API sem tratamento específico → `useErrorHandler` aplica a tratativa padrão (toast de rede ou redirect crítico).

**Nunca silenciar um erro com `catch (err) {}` vazio.**

---

#### Tipos de erro e como aparecem

| Tipo | Origem | Apresentação |
|---|---|---|
| Validação de formulário | Frontend (zod/vee-validate) | Inline abaixo do campo |
| Erro de domínio da API | Backend (`error` code conhecido) | Tratado pela view (inline ou toast específico) |
| Erro de negócio genérico | Backend (`error` code desconhecido) | Toast via `useErrorHandler` |
| Sem conexão / timeout | Rede | Toast de rede via `useErrorHandler` |
| Sessão expirada (401) | `httpClient` | Interceptado pelo cliente — nenhuma view trata 401 diretamente |
| Erro crítico / inesperado | Qualquer camada | `useReportCriticalError` → `errorStore` + redirect |

O 401 é interceptado pelo `httpClient`:
```
httpClient recebe 401 → authStore.clearAuth() → router guard redireciona para login
```

---

#### `showToast` — wrapper obrigatório

Toda chamada ao toast passa por `showToast` em `utils/toast/toast.ts`. Nunca chamar a lib de toast diretamente.

```typescript
// utils/toast/toast.ts
import { TOAST_VISIBILITY_MS } from '@/config'

export function showToast(params: ToastParams): void {
  Toast.show({ duration: TOAST_VISIBILITY_MS, ...params })
}
```

---

#### `useReportCriticalError`

Salva o erro no `errorStore` e redireciona para a view de erro genérico. Usado como último recurso — quando a view não reconhece o erro.

```typescript
// composables/commons/useReportCriticalError.ts
export function useReportCriticalError() {
  const router     = useRouter()
  const errorStore = useErrorStore()

  return (err: unknown): void => {
    errorStore.setError(err)
    void router.push({ name: 'public-error-generic' })
  }
}
```

---

#### `useErrorHandler`

Classifica e apresenta erros genéricos. Fica em `composables/commons/useErrorHandler.ts`. É o handler padrão para tudo que a view não tratou explicitamente.

```typescript
// composables/commons/useErrorHandler.ts
export function useErrorHandler() {
  const reportCritical = useReportCriticalError()
  const { t } = useI18n()

  return (err: unknown): void => {
    if (isNetworkError(err)) {
      showToast({ type: 'networkError', text: t('error.network.message') })
    } else if (err instanceof ApiError) {
      showToast({ type: 'error', text: err.message })
    } else {
      reportCritical(err)
    }
  }
}
```

`isNetworkError` fica em `utils/error/classifier.ts` e é a única responsável por distinguir erros de rede de erros de negócio.

---

#### `useAsyncAction`

Envolve ações disparadas pelo usuário com overlay de loading e tratamento de erro. O `onError` é onde a view trata erros de domínio — retornar `true` sinaliza que o erro foi tratado e o handler genérico não é chamado.

```typescript
// composables/commons/useAsyncAction.ts
interface AsyncActionOptions {
  onSuccess?: () => void
  onError?:   (err: unknown) => boolean | void
  overlay?:   boolean  // default: true — false para ações em lote
}

export function useAsyncAction() {
  const handleError  = useErrorHandler()
  const loadingStore = useLoadingStore()

  return async (action: () => Promise<void>, options?: AsyncActionOptions): Promise<void> => {
    const showOverlay = options?.overlay ?? true
    if (showOverlay) loadingStore.show()
    try {
      await action()
      options?.onSuccess?.()
    } catch (err) {
      const handled = options?.onError?.(err)
      if (!handled) handleError(err)
    } finally {
      if (showOverlay) loadingStore.hide()
    }
  }
}
```

**Exemplo com erro de domínio:**

```typescript
const runAction = useAsyncAction()

const onSubmit = handleSubmit(async (values) => {
  await runAction(() => userService.create(values), {
    onError: (err) => {
      if (err instanceof ApiError && err.code === 'EMAIL_IN_USE') {
        emailError.value = t('user.emailInUse') // inline no campo
        return true // tratado — não cai no handler genérico
      }
      // sem retorno → cai em useErrorHandler automaticamente
    },
    onSuccess: () => { showToast({ type: 'success', ... }); router.back() }
  })
})
```

---

#### Erros na busca inicial de dados (`useAsyncState`)

Para a busca inicial de dados, o tratamento de erro vai no callback `onError` do `useAsyncState`:

```typescript
const handleError = useErrorHandler()

const { state: users, isLoading } = useAsyncState(
  () => userService.list(),
  [],
  {
    onError: (err) => {
      if (err instanceof ApiError && err.code === 'FORBIDDEN') {
        noAccess.value = true // erro de domínio — view trata diretamente
        return
      }
      handleError(err) // genérico — delega ao handler padrão
    }
  }
)
```

---

### Estados Vazios

Quando uma lista não tem itens, o componente `EmptyState` substitui o conteúdo.

```typescript
// components/atom/feedback/EmptyState.vue
interface Props {
  icon:     string
  message:  string
  action?:  { label: string; onClick: () => void }
}
```

- Ícone + mensagem contextual definidos pela view — nunca genérico (`"Nenhum item"`)
- Centralizado na área de conteúdo
- Ação opcional — passada apenas quando faz sentido pelo negócio

---

### Listas paginadas

O modelo de paginação é sempre por cursor. Qual dos dois casos abaixo usar é ditado pela regra de negócio — não por preferência técnica.

---

#### Caso 1 — Full scan: carregar tudo, tratar no frontend

Usado quando a operação exige acesso ao conjunto completo de dados — ordenação arbitrária, filtros cruzados, seleção múltipla, exportação. O frontend busca todos os registros em lotes via cursor até o fim e aplica ordenação, filtragem e exibição localmente.

Exemplos: administrar todos os usuários, gerenciar catálogo de produtos.

```typescript
const items     = ref<Item[]>([])
const isLoading = ref(false)

async function loadAll(): Promise<void> {
  isLoading.value = true
  items.value = []
  let cursor: string | null = null

  try {
    do {
      const res = await itemService.list({ cursor: cursor ?? undefined, pageSize: PAGE_SIZE })
      items.value = [...items.value, ...res.data]
      cursor = res.pagination.nextCursor
    } while (cursor)
  } finally {
    isLoading.value = false
  }
}
```

Filtro, ordenação e busca são feitos sobre `items` com `computed()` — sem nova chamada à API.

> O limite de volume viável para o Caso 1 depende do projeto, da página e da regra de negócio. Quando o volume comprometer a performance do frontend, o filtro deve migrar para o backend — decisão documentada no arquivo `30+` do domínio.

---

#### Caso 2 — Continuação: carregar sob demanda

Usado quando os dados têm uma ordem natural (mais recentes, por data de criação) e o usuário avança conforme necessita. O frontend busca o primeiro lote e exibe um botão "Carregar mais" enquanto houver cursor.

Exemplos: listar os 50 eventos mais recentes, mostrar produtos por ordem de criação.

```typescript
const items       = ref<Item[]>([])
const nextCursor  = ref<string | null>(null)
const loadingMore = ref(false)

async function loadInitial(): Promise<void> {
  const res = await itemService.list({ pageSize: PAGE_SIZE })
  items.value     = res.data
  nextCursor.value = res.pagination.nextCursor
}

async function loadMore(): Promise<void> {
  loadingMore.value = true
  try {
    const res = await itemService.list({ cursor: nextCursor.value ?? undefined, pageSize: PAGE_SIZE })
    items.value      = [...items.value, ...res.data]
    nextCursor.value = res.pagination.nextCursor
  } finally {
    loadingMore.value = false
  }
}
```

```vue
<template>
  <div v-for="item in items" :key="item.id">
    <ItemCard :item="item" />
  </div>
  <EmptyState v-if="!isLoading && items.length === 0" ... />
  <button
    v-if="nextCursor"
    :disabled="loadingMore"
    @click="loadMore"
  >
    {{ t('action.loadMore') }}
  </button>
</template>
```

Scroll infinito não é usado — o usuário controla explicitamente quando buscar mais itens.

---

O tamanho do lote vem de `config/index.ts` (`PAGE_SIZE`).

> Este documento não cobre WebSockets, SSE ou qualquer mecanismo de atualização em tempo real. Projetos que necessitem dessas funcionalidades devem documentá-las em um arquivo `30+`.

---

### Formulários

#### Stack de validação

- **`vee-validate`** — gerencia valores, erros e estado do formulário
- **`zod`** — define o schema de validação com tipos TypeScript gerados automaticamente
- **`@vee-validate/zod`** — integra vee-validate com zod via `toTypedSchema`

```
npm install vee-validate zod @vee-validate/zod
```

#### Comportamento de validação

- **onBlur** — ao sair de um campo, valida aquele campo. Campos não tocados não exibem erro.
- **No submit** — valida todos os campos, incluindo os não tocados.
- **Botão de submit** — habilitado no estado inicial. Desabilitado quando há erros visíveis.

#### `useFormAction`

Composable que encapsula o ciclo completo de um formulário: validação, check de dirty, overlay de loading e tratamento de erro. Recebe a instância do `useForm` e retorna uma função que cria o handler de submit. Usado em **todos os formulários** — criação e edição.

```typescript
// composables/commons/useFormAction.ts
export function useFormAction<T extends Record<string, unknown>>(
  form: ReturnType<typeof useForm<T>>
) {
  const runAction  = useAsyncAction()
  const router     = useRouter()
  const { t }      = useI18n()
  const errorCount = ref(0)

  return (action: (data: T) => Promise<void>, options?: AsyncActionOptions) =>
    form.handleSubmit(
      async (data) => {
        errorCount.value = 0
        if (!form.meta.value.dirty) {
          void router.back()  // nada mudou — comportamento intencional, sem feedback
          return
        }
        await runAction(() => action(data), options)
      },
      () => {
        errorCount.value++
        if (errorCount.value >= 3) {
          errorCount.value = 0
          showToast({ type: 'error', text: t('form.hasErrors.message') })
        }
      }
    )
}
```

**Fluxo interno:**
1. Valida todos os campos ao submit — marca os não tocados com erro se inválidos
2. Se nada mudou (`dirty = false`) — navega de volta sem chamar a API. Comportamento intencional: nada a salvar, nada a comunicar.
3. Se inválido 3 vezes seguidas — exibe toast orientando o usuário a verificar os campos (resolve campos fora da área visível)
4. Executa a ação via `useAsyncAction` — overlay + tratamento de erro

#### Formulário de criação

```vue
<script setup lang="ts">
import { useForm } from 'vee-validate'
import { toTypedSchema } from '@vee-validate/zod'
import { z } from 'zod'

const schema = z.object({
  name:  z.string().min(1),
  email: z.string().email(),
})

const form = useForm({ validationSchema: toTypedSchema(schema) })

const [name,  nameAttrs]  = form.defineField('name',  { validateOnBlur: true })
const [email, emailAttrs] = form.defineField('email', { validateOnBlur: true })

const runFormAction = useFormAction(form)

const handleSave = runFormAction(async (values) => {
  await userService.create(values)
  showToast({ type: 'success', ... })
  router.back()
})
</script>

<template>
  <p class="required-note">{{ t('form.requiredFieldsNote') }}</p>
  <input v-model="name" v-bind="nameAttrs" :placeholder="`${t('user.name')} *`" />
  <span v-if="form.errors.value.name">{{ form.errors.value.name }}</span>
  <input v-model="email" v-bind="emailAttrs" :placeholder="t('user.email')" />
  <button :disabled="Object.keys(form.errors.value).length > 0" @click="handleSave">
    {{ t('action.save') }}
  </button>
</template>
```

#### Formulário de edição — pré-população

Em formulários de edição, os dados são buscados via `useAsyncState` e o formulário é populado com `resetForm` quando chegam. O `dirty` do vee-validate compara com os `initialValues` passados no `resetForm` — se o usuário retornar ao valor original, `dirty` volta a `false`.

```vue
<script setup lang="ts">
import { watch } from 'vue'
import { useAsyncState } from '@vueuse/core'

const props = defineProps<{ id: string }>()

const schema = z.object({ name: z.string().min(1), email: z.string().email() })
const form   = useForm({ validationSchema: toTypedSchema(schema) })

const [name,  nameAttrs]  = form.defineField('name',  { validateOnBlur: true })
const [email, emailAttrs] = form.defineField('email', { validateOnBlur: true })

const handleError = useErrorHandler()

const { state: user, isLoading } = useAsyncState(
  () => userService.get(props.id),
  null,
  { onError: (err) => handleError(err) }
)

// Popula o formulário quando o dado chega — resetForm define os initialValues
// que o vee-validate usa para calcular dirty
watch(user, (val) => {
  if (val) form.resetForm({ values: { name: val.name, email: val.email } })
}, { immediate: true })

const runFormAction = useFormAction(form)

const handleSave = runFormAction(async (values) => {
  await userService.update(props.id, values)
  showToast({ type: 'success', ... })
  router.back()
})
</script>
```

#### `useAsyncAction` direto na view

Para ações simples disparadas pelo usuário sem formulário e sem confirmação — arquivar, ativar/desativar, reenviar email, sincronizar. O `onError` trata erros de domínio específicos; o handler genérico cobre o restante.

#### `useDestructiveAction`

Para ações irreversíveis. Exibe dialog de confirmação, aguarda resposta, executa apenas no OK.

**Toda ação irreversível usa `useDestructiveAction`.** A exceção é quando um humano expressa que o dialog deve ser removido — documentar com comentário no código.

```typescript
// composables/commons/useDestructiveAction.ts
export function useDestructiveAction() {
  const runAction    = useAsyncAction()
  const { t }        = useI18n()
  const visible      = ref(false)
  const dialogConfig = ref({ title: '', message: '' })
  const pendingAction = ref<(() => void) | null>(null)

  function destructiveAction(
    action: () => Promise<void>,
    options?: AsyncActionOptions & { title?: string; message?: string }
  ): void {
    dialogConfig.value = {
      title:   options?.title   ?? t('dialog.destructive.title'),
      message: options?.message ?? t('dialog.destructive.message'),
    }
    pendingAction.value = () => runAction(action, options)
    visible.value = true
  }

  function confirm(): void { visible.value = false; pendingAction.value?.() }
  function cancel():  void { visible.value = false }

  return { destructiveAction, visible, dialogConfig, confirm, cancel }
}
```

```vue
<script setup lang="ts">
const { destructiveAction, visible, dialogConfig, confirm, cancel } = useDestructiveAction()

const handleDelete = (): void => destructiveAction(
  async () => { await userService.delete(id); router.back() },
  { message: t('user.deleteConfirm', { name: user.value.name }) }
)
</script>

<template>
  <button @click="handleDelete">{{ t('action.delete') }}</button>
  <ConfirmDialog
    :visible="visible"
    :title="dialogConfig.title"
    :message="dialogConfig.message"
    @confirm="confirm"
    @cancel="cancel"
  />
</template>
```

> O `ConfirmDialog` referenciado aqui é um componente obrigatório do projeto. Sua interface de props/emits, posicionamento no Atomic Design e relação com a lib de UI devem ser detalhados em um documento complementar numerado `30+`.

#### Separação de handlers e template

Handlers são definidos como `const` no `<script setup>` — nunca como funções inline no template.

```typescript
// correct — handler definido antes do template
const handleSave = runFormAction(async (data) => {
  await userService.update(id, data)
  showToast({ type: 'success', ... })
  router.back()
})
```

```html
<!-- correct -->
<button @click="handleSave">{{ t('action.save') }}</button>

<!-- wrong — lógica inline no template -->
<button @click="async () => { await service.update(id, data); router.back() }">Salvar</button>
```

Funções inline são permitidas apenas para casos triviais: `@click="router.back()"`.

#### Feedback de sucesso

Toda ação que conclui com sucesso exibe um toast `success` — sem exceção.

**Regra de navegação após sucesso:**

- Formulários (criação e edição) → toast + `router.back()`
- Ações de propósito único (deletar, confirmar) → toast + `router.back()`
- Ações em lote (toggles em lista) → toast por ação + debounce para recarregar lista

#### Ações por item em lista

Quando cada item de uma lista tem uma ação própria (ex: botão "alterar status"), cada botão precisa de loading e disabled independentes — o overlay global não é usado.

O loading por item é rastreado por um `Set` de IDs reativos:

```typescript
const loadingIds = ref(new Set<string>())
const runAction  = useAsyncAction()

function handleToggleStatus(item: Item): void {
  void runAction(
    async () => {
      loadingIds.value.add(item.id)
      const updated = await itemService.toggleStatus(item.id)
      const index = items.value.findIndex(i => i.id === item.id)
      if (index !== -1) items.value[index] = updated  // atualiza in-place
    },
    {
      overlay: false,
      onSuccess: () => { loadingIds.value.delete(item.id) },
      onError:   () => { loadingIds.value.delete(item.id); return false },
    }
  )
}
```

```vue
<template>
  <div v-for="item in items" :key="item.id">
    <ItemCard :item="item" />
    <button :disabled="loadingIds.has(item.id)" @click="handleToggleStatus(item)">
      <Spinner v-if="loadingIds.has(item.id)" />
      <span v-else>{{ t('action.toggleStatus') }}</span>
    </button>
  </div>
</template>
```

**Duas estratégias para atualizar a lista após a ação:**

- **Atualização in-place** — o service retorna o item atualizado e a view substitui diretamente no array. UX mais fluida, sem re-fetch.
- **Refresh com debounce** — após cada ação, agenda um reload da lista com `useDebounceFn` do VueUse. Mais simples, garante consistência com o backend. Preferível quando o servidor não retorna o item atualizado.

A escolha depende do que o endpoint retorna e da regra de negócio — definida no `30+` do domínio.

#### Quando usar cada composable de ação

| Composable | Quando usar |
|---|---|
| `useFormAction` | Todo formulário — criação e edição |
| `useAsyncAction` | Ações sem formulário e sem confirmação (arquivar, ativar/desativar, reenviar) |
| `useDestructiveAction` | Ações irreversíveis que precisam de confirmação (deletar, cancelar) |

#### Campos obrigatórios

Indicados com asterisco após o label: `Nome *`. Uma legenda discreta no topo do formulário explica o asterisco — uma vez, não em cada campo.

```vue
<p class="required-note">{{ t('form.requiredFieldsNote') }}</p>
<input :placeholder="`${t('user.name')} *`" v-bind="nameAttrs" v-model="name" />
<input :placeholder="t('user.phone')"        v-bind="phoneAttrs" v-model="phone" />
```
