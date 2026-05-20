---
name: react-native
description: "Read before writing or modifying any React Native / Expo code."
---

# React Native + Expo Architecture

> This document is specific to this project. It describes architectural decisions, conventions, and layer responsibilities that apply to the React Native / Expo codebase. Read it before writing or modifying any screen, component, service, store, or navigation file.

## Filosofia

As mesmas decisões que guiam o backend, adaptadas para o frontend:

**Produção primeiro.** Toda decisão é tomada pensando no comportamento em produção. Conforto de desenvolvimento (mocks, atalhos de navegação, dados fixos) é temporário e nunca deve chegar à versão final.

**Consistência elimina decisão.** Quando o custo de decidir caso a caso é maior do que aplicar uniformemente, a regra vira absoluta. Onde buscar dados, como tratar erros, onde guardar estado — essas respostas são sempre as mesmas.

**Explícito sobre implícito.** Tipos declarados em interfaces nomeadas. Nenhum campo inferido de `any`. Estado global declarado em stores com interface explícita.

**Mock é scaffolding, não arquitetura.** Mocks existem para permitir que o frontend avance antes do backend estar pronto. Todo mock tem uma data de validade: o momento em que o endpoint real for entregue. Nunca adicione lógica de negócio em um mock.

**Reutilizar antes de criar.** Antes de escrever qualquer coisa, verificar se já existe no projeto um componente, hook, service ou tipo que resolva o problema.

**Separação de responsabilidade.** Cada camada tem uma responsabilidade e não ultrapassa sua fronteira. A tela não faz chamadas HTTP. O service não conhece navegação. O store não conhece a API.

---

## Stack

| Pacote | Versão | Papel |
|---|---|---|
| `expo` | ~55 | Toolchain e runtime |
| `react-native` | 0.85 | Framework mobile |
| `react-navigation` | 7 | Navegação |
| `react-native-paper` | 5 | Componentes UI (Material Design 3) |
| `zustand` | 5 | Estado global |
| `@react-native-async-storage/async-storage` | 3 | Persistência local (token JWT) |

Antes de instalar um novo pacote, verificar se algum pacote já instalado resolve o problema.

---

## TypeScript

O projeto usa a configuração base do Expo (`expo/tsconfig.base`) com `strict: true`. Não alterar as opções do compilador sem justificativa documentada.

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

### Path alias

Use `@/` como alias para `src/`. Nunca use caminhos relativos com `../` para importar fora do diretório atual.

```typescript
// wrong
import { theme } from '../../../theme'

// correct
import { theme } from '@/theme'
```

---

## Estrutura de pastas

```
src/
├── components/      ← componentes reutilizáveis sem lógica de negócio
├── config/          ← configurações de ambiente (API base URL, etc.)
├── contexts/        ← React Contexts — apenas AuthContext
├── navigation/      ← AppNavigator e stacks/tabs de navegação
├── screens/         ← telas organizadas por domínio
│   ├── auth/        ← Login, Register, ConfirmEmail
│   ├── admin/       ← telas de administração
│   └── ...          ← outras áreas funcionais
├── services/        ← chamadas à API (real + mock)
├── store/           ← Zustand stores por entidade
│   └── u/           ← stores do usuário autenticado (authStore, accountStore, personStore)
├── theme/           ← tema global (react-native-paper)
└── types/           ← interfaces e tipos TypeScript
```

---

## Camadas e responsabilidades

### Tela (`screens/`)

A tela é responsável por:
- Renderizar a UI com base no estado local e nos stores
- Chamar services para buscar ou modificar dados
- Tratar estados de loading e erro locais
- Navegar entre telas

A tela **nunca**:
- Faz chamadas HTTP diretamente (não usa `fetch`, não usa `request`)
- Contém lógica de negócio ou transformação de dados
- Conhece os detalhes de implementação do service

```typescript
// wrong — tela chamando fetch diretamente
const data = await fetch('/v1/persons')

// correct — tela chamando o service
const persons = await PersonService.list()
```

### Service (`services/`)

O service é responsável por:
- Fazer chamadas HTTP via `request()` do `httpClient`
- Encapsular a URL, método e payload de cada endpoint
- Retornar os dados tipados para a tela

O service **nunca**:
- Conhece navegação ou estado da UI
- Contém lógica de negócio (validações, transformações)
- É chamado por outro service

O padrão de service tem dois arquivos:

```
services/
  xyzService.ts        ← service real — único ponto de entrada para telas
  xyzService.mock.ts   ← dados simulados — chamado apenas pelo service real
```

Telas e componentes importam **apenas** o service real. O mock é detalhe de implementação do service.

```typescript
// xyzService.ts
import { XyzServiceMock } from './xyzService.mock'
import { request } from './httpClient'

export const XyzService = {
  // enquanto o backend não existe:
  list: () => XyzServiceMock.list(),

  // quando o endpoint estiver pronto, trocar por:
  // list: () => request<Xyz[]>('/v1/xyz'),
}
```

A migração do mock para o endpoint real é feita **apenas** no service real. Telas não mudam.

### HTTP client (`services/httpClient.ts`)

A função `request<T>()` é o único ponto de acesso à API. Ela:
- Injeta o JWT no header `Authorization`
- Desembrulha o envelope `{ success, data, message, error }` da API
- Lança um erro com `message` e `code` quando `success: false`
- Chama o handler de 401 registrado pelo `AuthContext`

Nenhum arquivo fora de `services/` importa o `httpClient` diretamente.

### Store (`store/`)

Stores Zustand armazenam estado de sessão do usuário autenticado — dados que precisam estar disponíveis em qualquer tela sem prop drilling.

Stores atuais:
- `authStore` — token JWT, accountId, personId, email
- `accountStore` — dados da `Account` carregada após login
- `personStore` — dados da `Person` carregada após login

Regras:
- Cada store tem uma interface explícita com getters e setters nomeados
- Stores não fazem chamadas HTTP — apenas guardam e expõem dados
- Estado local de tela (loading, erro, valor de input) fica em `useState`, não no store

```typescript
interface AccountStore {
  account: Account | null;
  setAccount: (account: Account) => void;
  clearAccount: () => void;
}
```

### AuthContext (`contexts/AuthContext.tsx`)

O `AuthContext` é responsável pelo ciclo de vida da sessão:
- Restaurar o token do `AsyncStorage` na inicialização do app
- Executar o login (chamar `AuthService`, persistir token, popular stores)
- Executar o logout (limpar `AsyncStorage` e limpar todos os stores)
- Registrar o handler de 401 no `httpClient`

Regra: nenhuma tela ou serviço gerencia o token diretamente. Toda operação de sessão passa pelo `AuthContext`.

### Navegação (`navigation/`)

O `AppNavigator` é o guard de autenticação:
- Sem token → mostra o stack de autenticação (Login, Register, etc.)
- Com token mas stores vazias → mostra `PostLoginLoadingScreen` (carrega dados do usuário)
- Com token e stores populadas → mostra `MainTabs`

Cada área funcional tem seu próprio stack dentro de `MainTabs`. Novos grupos de telas ganham um novo stack navigator nomeado.

---

## Tema

O tema central fica em `src/theme/index.ts` e estende o `MD3LightTheme` do React Native Paper. É a **única fonte de verdade** para cores, tipografia e espaçamentos do sistema de design.

**Nunca hardcode cores em componentes ou telas.**

```typescript
// wrong
color: '#6750A4'
backgroundColor: '#F5F5F5'

// correct
color: theme.colors.primary
backgroundColor: theme.colors.background
```

Ao precisar de uma cor que não existe no tema, adicionar ao tema — não inline no componente.

---

## Type safety

### Sem `any`

Nunca use `any`. Se o tipo não é conhecido, use `unknown` e faça narrowing antes de usar o valor.

```typescript
// wrong
const err = e as any
console.log(err.code)

// correct
const err = e as Error & { code?: string }
console.log(err.code)
```

### Sem `as` para cast de tipos de domínio

Nunca use `as` para forçar um tipo de domínio em um valor desconhecido. Use type guards quando necessário.

```typescript
// wrong
const account = result as Account

// correct
function isAccount(value: unknown): value is Account {
  return typeof value === 'object' && value !== null && 'id' in value && 'email' in value
}
```

O operador `as` é permitido apenas em dois casos pontuais: props do React Navigation (`route.params as XParams`) e interop com bibliotecas sem tipos adequados — documentado com comentário inline explicando o motivo.

---

## Nomes de arquivos e convenções

**Arquivos:**

| Tipo | Convenção | Exemplo |
|---|---|---|
| Tela | `PascalCase` + `Screen.tsx` | `MeuPerfilScreen.tsx` |
| Componente | `PascalCase.tsx` | `ScreenBackHeader.tsx` |
| Service | `camelCase` + `Service.ts` | `accountService.ts` |
| Mock | `camelCase` + `Service.mock.ts` | `accountService.mock.ts` |
| Store | `camelCase` + `Store.ts` | `authStore.ts` |
| Tipo | `camelCase.ts` | `account.ts`, `organizacao.ts` |
| Context | `PascalCase` + `Context.tsx` | `AuthContext.tsx` |
| Navigator | `PascalCase.tsx` | `AppNavigator.tsx`, `MainTabs.tsx` |

**Identificadores no código:**

- `camelCase` — variáveis, funções, parâmetros, propriedades de objetos
- `PascalCase` — interfaces, type aliases, componentes, telas
- Nomes descritivos e autoexplicativos — nunca abreviações

---

## ESLint

O projeto usa ESLint com `typescript-eslint` e plugins para React e React Hooks. Rodar antes de commitar:

```
npm run lint
npm run lint:fix  ← auto-corrige o que for possível
```

Erros de ESLint não são warnings — são erros. O CI rejeita código com erros de lint.

---

## Sem console.log em código de produção

`console.log` é permitido apenas para depuração temporária durante desenvolvimento. Nunca commitar `console.log` em fluxos de negócio.

O `AppNavigator` mantém logs de guarda de autenticação (`[Guard] ...`) que são intencionais e documentados — são a exceção, não a regra.
