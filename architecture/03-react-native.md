---
name: react-native
description: "Read before writing or modifying any React Native / Expo code."
---

# React Native + Expo Architecture

> Documento base para projetos React Native. Descreve decisões de arquitetura, convenções e responsabilidades de cada camada. Exemplos e configurações usam Expo por ser o toolchain mais adotado — adapte onde necessário se o projeto não usar Expo. Leia antes de escrever ou modificar qualquer tela, componente, service, store ou arquivo de navegação.
>
> **Este documento prescreve como fazer — não descreve o que já está feito.** O que vale é a regra, não o exemplo. O estado atual do projeto pode diferir; o objetivo é convergir para estas diretrizes.
>
> **Valores de domínio nos exemplos são sempre ilustrativos.** Nomes de stores, namespaces de logger, códigos de erro, nomes de telas, estruturas de pastas — quando aparecem em exemplos, mostram o padrão a seguir, não os valores obrigatórios. O projeto define os valores concretos; o documento define a estrutura.
>
> **Sobre diretrizes que parecem específicas de projeto:** algumas decisões deste documento (como o formato do envelope HTTP ou o comportamento do cliente de autenticação) são boas práticas consolidadas, não imposições de um projeto específico. Quando não houver um documento de projeto que substitua ou complemente uma dessas diretrizes, esta é a regra vigente. Documentos numerados `30+` podem sobrescrever ou estender qualquer parte deste documento.

---

## Índice

| Seção                           | Conteúdo                                                                                                                           | Quando ler                                                                                          |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **S1 — Filosofia e Padrões**    | Princípios gerais, TypeScript, Atomic Design, type safety, convenções de nomes, linting, logs, performance, testes                 | Sempre — qualquer task                                                                              |
| **S2 — Infraestrutura**         | Dependências, estrutura de pastas, camadas (tela / service / HTTP / store / context), navegação, tema, config, types, hooks, utils | Ao criar ou alterar estrutura, novos arquivos, novas camadas                                        |
| **S3 — Implementação**          | `authStore`, stores de domínio (bootstrap/clear), erros críticos e telas de erro                                                   | Somente ao implementar ou dar manutenção em: auth, sessão, bootstrap, fluxo de login, telas de erro |
| **S4 — Padrões de Interação**   | Loading (local + overlay global), erros (tipos, hooks, toast), estados vazios, formulários (validação, hooks de ação)              | Ao implementar qualquer tela com loading, erro, lista vazia ou formulário                           |

**Mapeamento task → seção:**

- Criar ou editar qualquer tela, componente, hook, util → **S1 + S2 + S4**
- Dúvida sobre onde colocar um arquivo ou qual camada usar → **S2**
- Implementar login, logout, bootstrap, tela de erro → **S1 + S2 + S3**
- Dar manutenção em `authStore`, `AuthContext`, `errorStore` → **S3**
- Implementar formulário, loading, feedback de erro ou estado vazio → **S4**

---

## S1 — Filosofia e Padrões

### Filosofia

As mesmas decisões que guiam o backend, adaptadas para o frontend:

**Produção primeiro.** Toda decisão é tomada pensando no comportamento em produção. Conforto de desenvolvimento (mocks, atalhos de navegação, dados fixos) é temporário e nunca deve chegar à versão final.

**Consistência elimina decisão.** Quando o custo de decidir caso a caso é maior do que aplicar uniformemente, a regra vira absoluta. Onde buscar dados, como tratar erros, onde guardar estado, como estruturar a UI — essas respostas são sempre as mesmas.

**Explícito sobre implícito.** Tipos declarados em interfaces nomeadas. Nenhum campo inferido de `any`. Estado global declarado em stores com interface explícita.

**Mock é scaffolding, não arquitetura.** Mocks existem para permitir que o frontend avance antes do backend estar pronto. Todo mock tem uma data de validade: o momento em que o endpoint real for entregue. Nunca adicione lógica de negócio em um mock.

**Reutilizar antes de criar.** Antes de escrever qualquer coisa, verificar se já existe no projeto — ou nas libs instaladas — um componente, hook, util, service ou tipo que resolva o problema. A busca começa em `utils/` e `hooks/` para lógica pura e com estado, nos níveis do Atomic Design para UI, e nas APIs das bibliotecas instaladas.

**Não existe? Pergunte antes de criar.** Se o componente necessário não existe no projeto nem nas libs instaladas, não crie por conta própria. Pergunte ao usuário: o que esse componente deve fazer, como deve se comportar, quais variações precisa suportar, onde será usado. Criar um componente sem essas respostas é prototipar — e este não é um projeto de prototipação.

**Projeto real, não protótipo.** A solução mais curta raramente é a correta. O critério não é "funciona agora" — é "funciona bem, é consistente e pode ser mantido". Atalhos que economizam minutos criam horas de retrabalho.

**Subdiretórios são preferíveis a arquivos acumulados.** Um diretório com poucos arquivos bem nomeados é mais navegável do que um diretório com dezenas de arquivos planos. Organize por domínio desde o início — é mais fácil expandir uma estrutura já segmentada do que refatorar um diretório caótico depois.

**Separação de responsabilidade.** Cada camada tem uma responsabilidade e não ultrapassa sua fronteira. A tela não faz chamadas HTTP. O service não conhece navegação. O store não conhece a API. O componente não conhece o store.

**UI construída em camadas.** A interface segue o modelo Atomic Design: atoms formam molecules, molecules formam organisms, organisms compõem layouts, layouts sustentam screens. Nenhuma camada pula um nível. Essa hierarquia garante que mudar um atom propaga a mudança para todo o sistema de forma previsível.

---

### TypeScript

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

#### Path alias

Use `@/` como alias para `src/`. Nunca use caminhos relativos com `../` para importar fora do diretório atual.

```typescript
// wrong
import { theme } from "../../../theme";

// correct
import { theme } from "@/theme";
```

---

### Atomic Design

A UI do projeto segue o modelo **Atomic Design**. Cada peça de interface pertence a exatamente um nível, e esse nível define seu contrato de responsabilidade.

| Nível        | Onde                   | Descrição                                                                                                                                       |
| ------------ | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **Atom**     | `components/atom/`     | Peça mínima e indivisível. Sem composição de outros componentes do projeto. Exemplos: `Button`, `Badge`, `Avatar`, `Input`.                     |
| **Molecule** | `components/molecule/` | Composição de atoms com um propósito único. Exemplos: `SearchBar` (input + botão), `ListItem` (avatar + texto + badge).                         |
| **Organism** | `components/organism/` | Seção completa e autossuficiente, composta de atoms e molecules. Exemplos: `UserCard`, tabela com header e paginação.                           |
| **Layout**   | `layouts/`             | Esqueleto estrutural da tela — define onde ficam header, conteúdo e footer. Não contém dados. Exemplos: `ListScreenLayout`, `FormScreenLayout`. |
| **Screen**   | `screens/`             | A tela em si. Usa um layout, compõe organisms/molecules/atoms, busca dados via services e lida com navegação.                                   |

#### Regras

- Cada nível só pode compor elementos do seu nível ou de níveis inferiores. Um atom não usa molecules; uma molecule não usa organisms.
- Dentro de cada nível (`atom/`, `molecule/`, `organism/`), os componentes são organizados em subdiretórios por categoria funcional (ex: `atom/buttons/`, `atom/badges/`).
- Nenhum componente abaixo de Screen conhece navegação, stores ou services.
- Antes de criar qualquer componente novo, verificar se já existe um no nível adequado que resolva o problema.
- **Componentes de libs podem ser usados em qualquer nível.** Atoms, molecules e organisms podem usar diretamente componentes das libs instaladas.
- **Só crie um componente se for uma especialização.** Se a lib já oferece um `Button`, use-o diretamente. Crie um `atom/buttons/BtnConfirm` apenas se houver comportamento ou estilo específico do projeto que justifique a especialização. Wrappers sem propósito são proibidos.

---

### Type safety

#### Sem `any`

Nunca use `any`. Se o tipo não é conhecido, use `unknown` e faça narrowing antes de usar o valor.

```typescript
// wrong
const err = e as any;
console.log(err.code);

// correct
const err = e as Error & { code?: string };
console.log(err.code);
```

#### Sem `as` para cast de tipos de domínio

Nunca use `as` para forçar um tipo de domínio em um valor desconhecido. Use type guards quando necessário.

```typescript
// wrong
const account = result as Account;

// correct
function isAccount(value: unknown): value is Account {
  return (
    typeof value === "object" &&
    value !== null &&
    "id" in value &&
    "email" in value
  );
}
```

O operador `as` é permitido apenas em um caso pontual: interop com bibliotecas sem tipos adequados — documentado com comentário inline explicando o motivo. Para props do React Navigation, o tipo correto vem do ParamList via `StackScreenProps` ou `useRoute<RouteProp<...>>` — sem necessidade de `as`.

---

### Nomes de arquivos e convenções

**Regra base: tudo é `camelCase`.** A única exceção são arquivos que exportam componentes React (telas, atoms, molecules, organisms, layouts, navigators) — esses usam `PascalCase` porque o React exige que componentes comecem com letra maiúscula. Componentes com letra minúscula são interpretados como elementos HTML e não funcionam.

**Nomes de telas e stacks encapsulam a hierarquia de acesso.** O nome começa com o nível de acesso (`Public` ou `Private`), seguido do domínio e da ação — formando uma cadeia que é autodocumentada e elimina colisões entre telas de domínios diferentes.

```
Public  + Auth  + Login        → PublicAuthLoginScreen
Public  + Error + Generic      → PublicErrorGenericScreen
Private + Admin + Users        → PrivateAdminUsersScreen
Private + Admin + Users + Add  → PrivateAdminUsersAddScreen
```

`PrivateAdminUsersAddScreen` é diferente de `PrivateUsersAddScreen` — o domínio `Admin` está no nome, não apenas no diretório.

**Arquivos:**

| Tipo      | Convenção                                                               | Exemplo                                   |
| --------- | ----------------------------------------------------------------------- | ----------------------------------------- |
| Tela      | `PascalCase` + `Screen.tsx` seguindo hierarquia de acesso               | `PrivateAdminUsersScreen.tsx`             |
| Stack     | `PascalCase` + `Stack.tsx` seguindo hierarquia de acesso                | `PrivateAdminStack.tsx`                   |
| Atom      | `PascalCase.tsx` dentro de `atom/<categoria>/`                          | `atom/buttons/BtnBack.tsx`                |
| Molecule  | `PascalCase.tsx` dentro de `molecule/<categoria>/`                      | `molecule/cards/EventoCard.tsx`           |
| Organism  | `PascalCase.tsx` dentro de `organism/<categoria>/`                      | `organism/lists/PessoasList.tsx`          |
| Layout    | `PascalCase` + `Layout.tsx` dentro de `layouts/<categoria>/`            | `layouts/forms/FormScreenLayout.tsx`      |
| Navigator | `PascalCase.tsx`                                                        | `AppNavigator.tsx`, `PrivateMainTabs.tsx` |
| Context   | `PascalCase` + `Context.tsx`                                            | `AuthContext.tsx`                         |
| Service   | `camelCase` + `Service.ts`                                              | `userService.ts`                          |
| Mock      | `camelCase` + `Service.mock.ts`                                         | `userService.mock.ts`                     |
| Store     | `camelCase` + `Store.ts`                                                | `authStore.ts`                            |
| Tipo      | `camelCase.ts` dentro de `types/<domínio>/`                             | `types/user/user.model.ts`                |
| Config    | `camelCase.ts`                                                          | `config/index.ts`                         |
| Hook      | `camelCase` com prefixo `use` + `.ts` dentro de `hooks/<domínio>/`      | `hooks/commons/modal/useGenericModal.ts`  |
| Util      | `camelCase` descrevendo a operação + `.ts` dentro de `utils/<domínio>/` | `utils/dateTime/formatter.ts`             |

**Identificadores no código:**

- `camelCase` — variáveis, funções, parâmetros, propriedades de objetos, exports de services e stores
- `PascalCase` — componentes React (obrigação do React), interfaces e type aliases
- Nomes descritivos e autoexplicativos — evite abreviações. Use apenas as amplamente reconhecidas pela comunidade. Nunca crie abreviações novas.

---

### Linting

Todo projeto deve ter um linter configurado. Rodar antes de commitar — o comando depende do projeto, consulte o `package.json`.

Erros de lint não são warnings — são erros. O CI rejeita código com erros de lint. Configuração específica do linter (regras, plugins, extends) é definida pelo projeto.

---

### Logs

Todo log do projeto passa pelo `logger` — nunca use `console.log`, `console.info` ou `console.warn` diretamente no código. Logs diretos no console não têm controle de nível, não têm namespace e não podem ser desligados em produção.

A biblioteca padrão recomendada é `react-native-logs` por suportar níveis, transportes configuráveis por ambiente e loggers com namespace. Se o projeto usar outra solução, documente no arquivo de padrões do projeto (`30+`).

```typescript
// utils/logger/logger.ts
import { logger as createLogger } from "react-native-logs";

const log = createLogger({
  levels: { debug: 0, info: 1, warn: 2, error: 3 },
  transport: __DEV__ ? consoleTransport : silentTransport,
});

export const logger = {
  auth: log.extend("AUTH"),
  service: log.extend("SERVICE"),
  store: log.extend("STORE"),
};
```

```typescript
// uso — em qualquer arquivo do projeto
logger.auth.debug("bootstrap iniciado");
logger.service.error("falha ao buscar produtos", error);
```

`__DEV__` é uma variável global do React Native — `true` em desenvolvimento, `false` em produção. Use-a para configurar o transporte do logger por ambiente.

Regras:

- Nunca commitar `console.log` — o linter deve bloquear
- `logger.error` e `logger.warn` podem ser ativos em produção (enviados para serviço de monitoramento)
- `logger.debug` e `logger.info` são silenciados em produção por padrão

---

### Performance

#### ScrollView vs FlatList

Esta é a armadilha de performance mais comum no React Native. A escolha errada trava o app com listas grandes.

| Componente    | Renderiza                                | Use quando                                                             |
| ------------- | ---------------------------------------- | ---------------------------------------------------------------------- |
| `ScrollView`  | **Todos os filhos de uma vez**           | Conteúdo fixo e pequeno — formulários, telas de detalhe, configurações |
| `FlatList`    | **Só o que está visível** (virtualizado) | Listas de dados dinâmicos — qualquer coisa que vem de uma API          |
| `SectionList` | **Só o que está visível**, com seções    | Listas agrupadas por categoria                                         |

**Regra:** se o conteúdo vem de uma API ou pode crescer, use `FlatList`. `ScrollView` com muitos itens renderiza tudo na memória e trava a UI.

```typescript
// wrong — trava com listas grandes
<ScrollView>
  {produtos.map(p => <ProdutoCard key={p.id} produto={p} />)}
</ScrollView>

// correct — renderiza só o visível
<FlatList
  data={produtos}
  keyExtractor={p => p.id}
  renderItem={({ item }) => <ProdutoCard produto={item} />}
/>
```

#### Otimizações de renderização (`React.memo`, `useCallback`, `useMemo`)

O React re-renderiza componentes filhos quando o pai re-renderiza. Em alguns cenários isso é desnecessário e causa lentidão — `React.memo`, `useCallback` e `useMemo` existem para evitar isso.

**Não aplique essas otimizações por padrão.** Elas adicionam complexidade e na maioria dos casos o React é rápido o suficiente sem elas. Usadas sem necessidade real, complicam o código sem benefício.

**Se identificar um cenário onde parecem necessárias, não decida sozinho — apresente o caso ao usuário e valide antes de aplicar.** O critério é: existe um problema de performance medido ou claramente visível? Se não, não use.

---

### Testes

Teste que não encontra bugs reais não tem valor. Não crie testes para atingir cobertura — crie testes para garantir comportamento.

**Nível mais alto possível.** O teste certo para uma tela é um teste E2E que simula o usuário usando o app — ele cobre tela, store e service ao mesmo tempo, como o handler test cobre handler, service e repository no backend. Testes de componente em isolamento raramente compensam o custo de manutenção.

| O que testar         | Nível | Justificativa                                                                                  |
| -------------------- | ----- | ---------------------------------------------------------------------------------------------- |
| `utils/`             | Unit  | Funções puras, sem UI, fácil de testar e alto valor                                            |
| Fluxos de tela       | E2E   | Cobre tudo de uma vez, é o "usuário usando"                                                    |
| Componentes isolados | —     | Evitar — E2E já cobre, e testes de componente tendem a testar implementação, não comportamento |

A configuração de ferramentas de teste (Jest, Maestro, Detox) é definida pelo projeto no arquivo `30+`.

---

### Padrões visuais do projeto (`XX-ui-patterns.md`)

Este documento cobre arquitetura e princípios — decisões que valem para qualquer projeto React Native. Decisões visuais específicas do projeto pertencem a um documento separado, numerado na faixa `30+`, seguindo o índice em `00-index.md`.

**Todo projeto que usa este documento base deve criar seu próprio arquivo de padrões visuais.** Sem ele, cada tela inventa sua própria resposta para situações recorrentes — e a inconsistência é inevitável.

Esse arquivo cobre as decisões que se repetem em toda tela mas que dependem do design do projeto. Exemplos do que precisa ser definido lá:

- Como exibir estado de carregamento
- Como exibir erros ao usuário
- Como exibir listas vazias
- Como estruturar formulários e exibir erros de validação

A lista não é fechada. Sempre que uma situação recorrente não tiver resposta definida, a decisão vai para esse arquivo — não fica implícita no código.

---

## S2 — Infraestrutura

### Dependências e bibliotecas

Este é um projeto real e de longo prazo — não uma prototipação. Toda decisão deve considerar manutenibilidade, consistência e escala.

**Leia o `package.json` antes de escrever qualquer coisa.** As bibliotecas instaladas definem o vocabulário do projeto. Elas já resolvem navegação, estado, UI, persistência, requisições HTTP. O trabalho é compor essas soluções, não reinventá-las.

**Use os conceitos que as libs já fornecem.** Se a biblioteca de UI tem variantes de tipografia, use essas variantes — não crie `fontSize` manual. Se a biblioteca de navegação tem um padrão para stack, use esse padrão — não crie navegação própria. Adotar o modelo da lib é sempre preferível a criar um paralelo.

**Antes de instalar uma nova dependência**, verificar se alguma lib já instalada resolve o problema. A adição de dependência deve ser a última opção, não a primeira.

---

### Estrutura de pastas

```
src/
├── components/      ← peças de UI sem lógica de negócio (Atomic Design)
│   ├── atom/        ← peças mínimas e indivisíveis
│   │   ├── buttons/
│   │   ├── badges/
│   │   └── text/
│   ├── molecule/    ← composições de atoms com um propósito
│   │   ├── cards/
│   │   └── list-items/
│   └── organism/    ← seções completas e autossuficientes
│       └── ...
├── layouts/         ← templates estruturais de tela
│   ├── forms/       ← FormScreenLayout, FormTabScreenLayout
│   └── list/        ← ListScreenLayout
├── config/          ← configurações de ambiente — um único index.ts (veja seção Config)
├── contexts/        ← React Contexts criados pelo projeto (veja regra abaixo)
├── navigation/      ← navegação organizada por hierarquia de acesso
│   ├── AppNavigator.tsx
│   ├── public/
│   │   ├── PublicStack.tsx
│   │   ├── PublicAuthStack.tsx
│   │   └── PublicErrorStack.tsx
│   └── private/
│       ├── PrivateStack.tsx
│       ├── PrivateMainTabs.tsx
│       └── <domain>/     ← um subdiretório por área funcional
│           └── Private<Domain>Stack.tsx
├── screens/         ← telas espelhando a hierarquia de navegação
│   ├── public/
│   │   ├── auth/    ← PublicAuthLoginScreen, PublicAuthRegisterScreen
│   │   └── error/   ← PublicErrorGenericScreen, PublicErrorBootstrapScreen
│   └── private/
│       ├── <domain>/  ← Private<Domain><Action>Screen
│       └── ...        ← um subdiretório por área funcional
├── services/        ← chamadas à API (real + mock)
├── store/           ← Zustand stores por entidade
├── theme/           ← tema global (react-native-paper)
├── types/           ← interfaces e tipos TypeScript, subdiretório por domínio
│   └── navigation/  ← ParamList de cada stack — um arquivo por stack
├── hooks/           ← custom hooks reutilizáveis, subdiretório por domínio
│   ├── commons/     ← hooks genéricos sem domínio específico
│   └── <domain>/    ← hooks específicos de um domínio
└── utils/           ← funções puras sem hooks, subdiretório por domínio
    ├── dateTime/    ← formatter.ts, validator.ts
    └── string/      ← formatter.ts, parser.ts
```

---

### Camadas e responsabilidades

#### Tela (`screens/`)

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
const data = await fetch("/v1/persons");

// correct — tela chamando o service
const persons = await personService.list();
```

**Tratamento de erros na tela.** Toda chamada a um service ou store deve estar dentro de um `try/catch`. A tela é a única camada que decide o que fazer com um erro:

- Erro de negócio (validação falhou, recurso não encontrado, sem permissão) → `useState` local, feedback inline ao usuário
- Erro crítico (falha irrecuperável, estado inválido) → `useReportCriticalError()` — encapsula `errorStore.setError(erro)` e `navigate('PublicErrorGenericScreen')`

```typescript
// correct
const reportCriticalError = useReportCriticalError();

try {
  const persons = await personService.list();
  setPersons(persons);
} catch (err) {
  if (err instanceof ApiError && err.code === "NOT_FOUND") {
    setError("Nenhuma pessoa encontrada."); // erro de negócio — feedback inline
  } else {
    reportCriticalError(err); // erro inesperado — errorStore + navigate
  }
}
```

#### Service (`services/`)

O service é responsável por:

- Fazer chamadas HTTP via `request()` do `httpClient`
- Encapsular a URL, método e payload de cada endpoint
- Retornar os dados tipados para a tela

O service **nunca**:

- Conhece navegação ou estado da UI
- Conhece stores — não importa nem acessa nenhuma store
- Contém lógica de negócio de domínio (validações de negócio, transformações)
- É chamado por outro service

**Tratamento de erros no service.** O service pode e deve tratar erros que são de sua responsabilidade — retry em timeout, normalização de códigos de erro HTTP, etc. O que o service nunca faz é capturar um erro fora do seu domínio e descartá-lo silenciosamente. Erros que o service não sabe resolver são sempre propagados ao chamador.

O padrão de service tem dois arquivos, organizados em subdiretórios por domínio. A infraestrutura HTTP fica em `services/api/`, separada dos services de domínio:

```
services/
├── api/                          ← infraestrutura HTTP — um arquivo por backend
│   ├── backend.httpClient.ts      ← backend principal
│   ├── erp.httpClient.ts         ← backend ERP
│   └── facebook.httpClient.ts    ← API externa
├── auth/
│   ├── authService.ts
│   └── authService.mock.ts
├── evento/
│   ├── eventoService.ts
│   └── eventoService.mock.ts
└── user/
    ├── userService.ts
    └── userService.mock.ts
```

Cada service de domínio importa o cliente HTTP adequado de `@/services/api/`. Se um service precisar de dois backends, importa dois clientes.

Telas e componentes importam **apenas** o service real. O mock é detalhe de implementação do service.

```typescript
// xyzService.ts — enquanto o backend não existe
import { xyzServiceMock } from "./xyzService.mock";

export const xyzService = {
  // TODO: backend não implementado — substituir quando endpoint estiver pronto
  list: () => xyzServiceMock.list(),
};
```

Quando o endpoint estiver pronto, o mock é removido e o arquivo `.mock.ts` é deletado:

```typescript
// xyzService.ts — backend implementado, sem mock
import { backendHttpClient } from "@/services/api/backend.httpClient";

export const xyzService = {
  list: () => backendHttpClient.request<Xyz[]>("/v1/xyz"),
};
```

**Regras de mock:**

- Todo uso de mock tem um comentário `// TODO: backend não implementado` — facilita localizar o que ainda falta
- Se o endpoint existe, não há justificativa para manter o mock — usar mock com backend disponível é débito técnico
- Quando todos os endpoints de um service estiverem implementados, o arquivo `.mock.ts` é deletado
- A migração é feita **apenas** no service real — telas não mudam

#### HTTP clients (`services/api/`)

Cada arquivo em `services/api/` é um cliente HTTP dedicado a um backend. Um cliente:

- Injeta o JWT no header `Authorization` lendo o token diretamente do `authStore` (em memória) — nunca do `AsyncStorage`. O `AsyncStorage` é acessado apenas no `authStore.bootstrap()` durante a inicialização do app.
- Desembrulha o envelope de resposta da API e lança erro quando a resposta indica falha
- Chama o handler de 401 registrado pelo `AuthContext` (quando aplicável)

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

Nenhum arquivo fora de `services/` importa um cliente HTTP diretamente.

#### Store (`store/`)

Stores Zustand armazenam estado de sessão do usuário autenticado — dados que precisam estar disponíveis em qualquer tela sem prop drilling.

Stores ficam diretamente em `store/`, sem subpastas. Exemplos de stores típicas:

- `authStore` — token JWT e identificadores da sessão
- `accountStore` — dados da conta carregada após login
- `personStore` — dados da pessoa carregada após login
- `errorStore` — erro crítico atual, usado para redirecionar o usuário a uma tela de erro

**Quando usar store:**

- Dado compartilhado entre telas que tem um único ponto de modificação (ex: nome do usuário — qualquer tela lê, só o login altera)
- Dado de sessão que precisa estar disponível em qualquer ponto do app sem prop drilling

**Quando não usar store:**

- Dado local da tela (loading, erro, valores de formulário) → `useState`
- Dado que a tela busca para si mesma → `useState` + service
- Dado passado entre telas via navegação → params do navigator. Params identificam e orientam — não transportam estado. Passar `id`, `type` ou campos que definem o comportamento da próxima tela é correto. Passar um objeto completo de domínio via params é proibido: some com reload e cria acoplamento implícito entre telas.

**Stores de cache:** quando uma tela identifica, por regra de negócio, que buscar os dados a cada acesso é custoso demais, ela pode requerer uma store de cache dedicada. Essa store vive em `store/` como qualquer outra e é responsável por guardar os dados e controlar quando invalidar (por tempo ou ação do usuário). Não é uma regra criar store de cache para toda lista — é uma decisão caso a caso da tela com base em necessidade real.

Regras gerais:

- Cada store tem uma interface explícita com getters e setters nomeados
- Stores não chamam o `httpClient` diretamente — toda comunicação com a API é delegada aos services
- `AsyncStorage` é acessado exclusivamente por stores — nunca por telas, services ou contexts. Cada store é responsável por suas próprias chaves de persistência.
- Stores podem tratar erros dentro do seu domínio de responsabilidade — por exemplo, validar dados antes de armazená-los e lançar se inválido. O que a store nunca faz é capturar um erro fora do seu domínio e descartá-lo silenciosamente. Erros que a store não sabe resolver são sempre propagados ao chamador. A decisão final sobre o que fazer com um erro pertence à tela ou ao `AuthContext`.

```typescript
interface AccountStore {
  account: Account | null;
  setAccount: (account: Account) => void;
  clearAccount: () => void;
}
```

#### AuthContext (`contexts/AuthContext.tsx`)

O `AuthContext` é o guarda da sessão — não contém lógica de negócio. Sua única responsabilidade é observar se há uma sessão válida e agir quando não há.

Fluxo na inicialização do app:

```
AuthContext chama authStore.getToken()
  → authStore verifica memória → não tem → verifica AsyncStorage
    → não encontrou → retorna null → AppNavigator renderiza PublicStack
    → encontrou     → salva em memória → retorna token
                      → AppNavigator renderiza PrivateStack
                        → PrivatePostLoginLoadingScreen chama authStore.bootstrap()
                          → sucesso → navega para PrivateHomeScreen
                          → falha   → reportCriticalError(err)
                                      → PublicErrorBootstrapScreen
```

Responsabilidades do `AuthContext`:

- Chamar `authStore.getToken()` na inicialização — o token é resolvido da memória ou do `AsyncStorage` pelo próprio store, sem o `AuthContext` conhecer os detalhes de persistência
- Registrar o handler de 401 no `httpClient` — ao receber 401, o `AuthContext` chama `authStore.clearAuth()` e navega para `PublicAuthLoginScreen`. O `clearAuth()` é responsável por limpar o token e acionar o `clear()` de cada store de domínio internamente.
- Observar o token no `authStore` — o `AppNavigator` reage e renderiza `PublicStack` ou `PrivateStack` conforme o estado

**`authStore.clearAuth()` é chamado em dois contextos:**

- **`AuthContext`** — em resposta a um 401 (sessão expirada pelo servidor)
- **`PublicAuthLogoutScreen`** — em resposta ao logout explícito do usuário

Nenhum outro service, hook ou tela chama `clearAuth()` diretamente.

O `AuthContext` **nunca**:

- Chama serviços diretamente
- Contém regras de negócio de autenticação
- Gerencia token ou dados de usuário — isso é responsabilidade das stores

#### Internacionalização (i18n)

Suporte a múltiplos idiomas é implementado como uma store — nunca como um Context. Uma `languageStore` guarda o idioma atual e as traduções carregadas. Qualquer componente lê da store. Trocar idioma atualiza todos os assinantes via Zustand.

**Stack:**

- **`i18next`** — motor de i18n, gerencia idioma atual e carregamento de traduções
- **`react-i18next`** — hook `useTranslation()` para componentes React Native
- **`zod-i18n-map`** — integra zod com i18next, traduz mensagens de validação automaticamente

**Estrutura de arquivos:**

```
src/
└── locales/
    ├── pt-BR.json   ← idioma padrão
    └── es.json      ← adicionar quando necessário
```

Organização das chaves por domínio:

```json
{
  "validation": {
    "required": "Campo obrigatório.",
    "email": "Email inválido.",
    "min": "Mínimo {{count}} caracteres."
  },
  "action": {
    "save": "Salvar",
    "cancel": "Cancelar",
    "delete": "Excluir",
    "add": "Adicionar"
  }
}
```

**Uso nos componentes:**

```typescript
const { t } = useTranslation()

<TextInput label={t('entity.name')} />
<Button>{t('action.save')}</Button>
```

**Regra obrigatória:** nenhum texto visível ao usuário é hardcoded. Todo string definido no frontend passa por `t()`. Isso inclui: labels, placeholders, títulos de tela, textos de botão, mensagens de estado vazio, toasts de feedback.

**Conteúdo vindo do backend — caso a caso:**

- Backend enviou um **código de erro** (ex: `"USER_NOT_FOUND"`) → passar por `t()`, traduzindo o código para uma mensagem legível
- Backend enviou **dados de domínio** (nome de usuário, título de produto, descrição) → exibir diretamente, sem `t()`. São valores inseridos por usuários, não constantes do sistema — não faz sentido traduzi-los.

**Integração com o `languageStore`:**

```
authStore.bootstrap()
  → languageStore.bootstrap()  ← carrega idioma salvo + chama i18next.changeLanguage()
  → accountStore.bootstrap()
  → ...
```

**i18n é estado global — vai em store, não em Context.**

#### Sobre Contexts em geral

Context é o mecanismo do React para executar efeitos reativos na raiz da árvore de componentes — coisas que precisam rodar antes de qualquer tela ser renderizada. **Não é um substituto para estado global** — para isso existe o Zustand.

Na prática, num projeto com Zustand, você quase nunca precisa criar um segundo Context. Tudo que parece precisar de Context — idioma, tema, permissões, dados do usuário — resolve com uma store.

Os Contexts que aparecem na árvore do app mas **não são escritos pelo projeto** são Providers de bibliotecas (`PaperProvider`, `NavigationContainer`, etc.) — eles não contam como Contexts do projeto.

**Antes de criar um novo Context, responda:** isso precisa de efeitos reativos integrados ao ciclo de vida do React, ou é só estado global? Se for estado global, é uma store.

---

### Navegação (`navigation/`)

A árvore de navegação usa um **único `NavigationContainer`**. O `AppNavigator` é o navigator raiz dentro dele e decide qual área renderizar com base no token. A separação entre `Public` e `Private` é feita por stacks condicionais — não por containers separados.

```
NavigationContainer
└── AppNavigator
    ├── PublicStack              ← sem token (navigation/public/)
    │   ├── PublicAuthStack
    │   │   ├── PublicAuthLoginScreen
    │   │   ├── PublicAuthRegisterScreen
    │   │   ├── PublicAuthLogoutScreen
    │   │   └── ...
    │   └── PublicErrorStack     ← acessível sem autenticação
    │       ├── PublicErrorBootstrapScreen
    │       └── PublicErrorGenericScreen
    └── PrivateStack             ← com token (navigation/private/)
        ├── PrivatePostLoginLoadingScreen
        └── PrivateMainTabs
            ├── Private<Domain>Stack  ← um por área funcional
            └── ...
```

**Telas de erro dentro do `PublicStack`.** Erros críticos não requerem autenticação para serem exibidos — vivem no `PublicErrorStack`, dentro do `PublicStack`. Qualquer tela (pública ou privada) navega para elas com `navigate('PublicErrorGenericScreen')`. O React Navigation encontra a tela subindo e descendo a hierarquia automaticamente. A localização em `public/error/` força a pergunta: "esse dado pode ser exposto sem autenticação?"

**Histórico de navegação unificado.** O histórico é único e linear — `goBack()` funciona livremente entre stacks de tabs diferentes. Se o usuário estava em `PrivateAdminUsersScreen`, navegou para Home e apertou voltar, retorna para `PrivateAdminUsersScreen`. Esse é o comportamento correto e esperado.

**Double-tap na tab reseta o stack.** Tocar em uma tab já ativa desempilha todas as telas daquele stack e retorna para a tela raiz. É a ação explícita do usuário para "começar do zero" naquela área.

```typescript
// em cada Screen dentro do tab navigator:
listeners={({ navigation }) => ({
  tabPress: () => {
    if (navigation.isFocused()) {
      navigation.dispatch(StackActions.popToTop())
    }
  },
})}
```

Regras:

- `AppNavigator` não contém lógica — apenas lê o token do `authStore` e renderiza `PublicStack` ou `PrivateStack`
- Toda tela nova é declarada em exatamente um stack, espelhando sua pasta em `screens/`
- Com token mas stores ainda vazias → `PrivatePostLoginLoadingScreen` executa o bootstrap antes de mostrar o app
- Cada área funcional tem seu próprio stack dentro de `PrivateMainTabs`
- Telas raiz de cada tab nunca têm botão voltar
- **Nomes de telas e stacks encapsulam a hierarquia de acesso** — o nome é autodocumentado e elimina ambiguidade entre telas de domínios diferentes (ver seção Convenções)

---

### Tema

O tema central fica em `src/theme/` e é a **única fonte de verdade** para cores, tipografia e espaçamentos do sistema de design. Ele estende o tema da biblioteca de UI instalada — leia o `package.json` para identificar qual é.

**Nunca hardcode valores visuais em componentes ou telas.** Cores, tamanhos de fonte e espaçamentos sempre vêm do tema.

```typescript
// wrong
color: "#6750A4";
backgroundColor: "#F5F5F5";
fontSize: 16;
padding: 12;

// correct
color: theme.colors.primary;
backgroundColor: theme.colors.background;
fontSize: theme.fonts.bodyMedium.fontSize; // ou equivalente da lib instalada
padding: Spacing.md;
```

Ao precisar de um valor que não existe no tema, adicionar ao tema — não inline no componente.

---

### Config (`config/index.ts`)

Valores de ambiente que mudam entre builds (URLs, timeouts, feature flags) ficam em um único arquivo `src/config/index.ts`. Nenhum outro arquivo acessa variáveis de ambiente diretamente.

```typescript
// src/config/index.ts
export const API_BASE_URL =
  process.env.EXPO_PUBLIC_API_URL ?? "http://localhost:3000";
export const API_TIMEOUT = 10_000;
```

```typescript
// uso
import { API_BASE_URL } from "@/config";
```

Se o arquivo crescer muito, quebre por domínio (`config/api.ts`, `config/features.ts`). Comece com um arquivo só.

---

### Types (`types/`)

O padrão é sempre criar tipos em `types/`, com subdiretório por domínio.

**Proibido usar `index.ts` como barrel export em qualquer parte do `src/`.** Essa regra é global — vale para `types/`, `components/`, `hooks/`, `services/`, `utils/` e demais diretórios. O caminho do arquivo é informação semântica: `import { User } from '@/types/user/user.model'` diz que é o modelo de domínio. `import { User } from '@/types/user/user.api'` diz que é um tipo de contrato com a API. Um `index.ts` que re-exporta tudo apaga essa informação, dificulta tree-shaking e aumenta o risco de imports circulares.

```
types/
├── navigation/
│   ├── publicStack.ts        ← ParamList do PublicStack
│   ├── privateStack.ts       ← ParamList do PrivateStack
│   ├── mainTabs.ts           ← ParamList do MainTabs
│   └── adminStack.ts         ← ParamList de cada stack de domínio
├── user/
│   ├── user.model.ts         ← modelo de domínio (o que o app usa internamente)
│   └── user.api.ts           ← tipos de request/response da API
├── evento/
│   ├── evento.model.ts
│   └── evento.api.ts
└── organizacao/
    └── organizacao.model.ts
```

Tipos de navegação ficam em `types/navigation/`, um arquivo por stack navigator. Nunca dentro de `navigation/` — tipos pertencem a `types/`.

Se o domínio for simples e tiver poucos tipos, um único arquivo por domínio é suficiente — quebre em arquivos conforme o domínio crescer.

**Tipo local é exceção, não regra.** Use tipo local apenas quando for verdadeiramente interno a um único arquivo e sem nenhuma utilidade fora dele — por exemplo, um tipo auxiliar para troca de parâmetros entre duas funções privadas dentro do mesmo service.

```typescript
// local — correto, nunca vai ser usado fora deste arquivo
type ParsedResponse = { id: string; raw: unknown }
function parse(raw: unknown): ParsedResponse { ... }
function transform(parsed: ParsedResponse) { ... }
```

Se existe qualquer chance do tipo ser usado em outro arquivo — tela, store, service, componente — ele vai para `types/`.

---

### Hooks (`hooks/`)

Hooks são funções que usam hooks do React internamente (`useState`, `useEffect`, `useRef`, etc.). São o lugar para lógica com estado reutilizável entre telas ou componentes.

```
hooks/
├── commons/          ← hooks genéricos sem domínio específico
│   └── modal/
│       └── useGenericModal.ts
└── <domain>/         ← hooks específicos de um domínio
    └── useEventosFilter.ts
```

**Quando criar um hook:** quando a mesma lógica com estado aparecer em mais de um lugar. Se a lógica existe em uma tela só, fica na tela.

**Hooks podem receber parâmetros** — inclusive stores — tornando-os adaptadores reutilizáveis entre estado e componente.

**Hooks não são `utils/`.** Se a função não usa nenhum hook do React, é uma função pura e vai para `utils/`.

---

### Utils (`utils/`)

Utils são funções puras sem hooks do React — formatação, validação, parsing, cálculos. Podem ser usadas em qualquer contexto, inclusive fora do React.

```
utils/
├── dateTime/
│   ├── formatter.ts   ← funções que formatam datas
│   └── validator.ts   ← funções que validam datas
└── string/
    ├── formatter.ts   ← funções que formatam strings
    └── parser.ts      ← funções que parseiam strings
```

O subdiretório define o domínio. O arquivo define a operação (`formatter`, `validator`, `parser`, `calculator`). Juntos são auto-explicativos sem repetição.

**Se precisar de um hook dentro, não é util — é hook.**

---

## S3 — Implementação

> Leia esta seção apenas ao implementar ou dar manutenção em: auth, sessão, bootstrap, fluxo de login, telas de erro crítico.

### authStore (`store/authStore.ts`)

O `authStore` é o repositório da sessão e o orquestrador da autenticação e inicialização das stores.

Responsabilidades:

- Guardar e expor o token JWT em memória
- Implementar `getToken()`: verifica memória primeiro — se não encontrar, busca no `AsyncStorage`, salva em memória e retorna. Se não encontrar em nenhum dos dois, retorna `null`.
- Implementar `login(credentials)`: chama `authService.login()`, persiste o token no `AsyncStorage` e em memória, retorna sucesso ou propaga a exceção para a tela tratar
- Implementar `bootstrap()`: chama `bootstrap()` em cada store que precisa ser inicializada — chamado pela `PrivatePostLoginLoadingScreen`
- Limpar o token do `AsyncStorage` e da memória, e acionar `clear()` nas demais stores no logout

**Fluxo de login:**

```
PublicAuthLoginScreen — usuário submete formulário
  → authStore.login(credentials)
    → authService.login(credentials)
      → sucesso → token salvo no AsyncStorage e no authStore
        → authStore retorna sucesso para PublicAuthLoginScreen
          → PublicAuthLoginScreen navega para PrivatePostLoginLoadingScreen
            → PrivatePostLoginLoadingScreen chama authStore.bootstrap()
              → sucesso → navega para PrivateHomeScreen
              → falha UNAUTHORIZED → ignora — handler de 401 do AuthContext
                                     já limpou a sessão e navegou para
                                     PublicAuthLoginScreen
              → outra falha → reportCriticalError(err) → PublicErrorBootstrapScreen
      → falha → exceção propagada → PublicAuthLoginScreen trata (feedback inline)
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
  bootstrap(): Promise<void>;
  clear(): void;
}
```

O `authStore.bootstrap()` conhece quais stores precisam ser inicializadas e chama o `bootstrap()` de cada uma em sequência. Cada store decide como se popular — o `authStore` não conhece os detalhes.

```typescript
// exemplo — a lista exata depende do projeto
async bootstrap() {
  await languageStore.bootstrap()
  await accountStore.bootstrap()
  await personStore.bootstrap()
}
```

**Regra:** toda store que precisa ser inicializada no login **deve** ser adicionada explicitamente ao `authStore.bootstrap()`. Essa lista é a fonte de verdade das dependências de inicialização do app. Criar uma store com `bootstrap()` sem adicioná-la aqui é um erro — a store começará vazia e a tela quebrará silenciosamente.

---

### Erros críticos

A tela sempre tem o erro em mãos — ela captura a exceção no `try/catch` e decide o que fazer com ela:

| Tipo                | Quando                                                                                           | Onde tratar                                                                                       |
| ------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| **Erro esperado**   | A tela reconhece o código do erro e sabe como reagir — 404, 403, falha de validação, lista vazia | `useState` na própria tela, feedback inline. O usuário continua navegando.                        |
| **Erro inesperado** | A tela não reconhece o erro — exceção de runtime, estado inválido, erro sem código conhecido     | `useReportCriticalError()` → `errorStore.setError(erro)` + `navigate('PublicErrorGenericScreen')` |

A lista de códigos de erro que cada tela trata por conta própria é decisão de projeto — definida no arquivo `30+`. O que este documento estabelece é o princípio: erros reconhecidos ficam na tela; erros não reconhecidos vão para o `errorStore`.

O projeto tem telas de erro pré-definidas em `screens/public/error/`, cada uma com responsabilidade e ação de recuperação específicas. O projeto pode adicionar telas de erro próprias neste diretório conforme necessário.

**`PublicErrorBootstrapScreen`** — exclusiva para falhas no bootstrap da sessão. Exibe o erro e oferece um botão de retry que navega para `PrivatePostLoginLoadingScreen` — sem chamar nenhuma store diretamente.

```
PrivatePostLoginLoadingScreen chama authStore.bootstrap()
  → falha → reportCriticalError(err)
              → errorStore.setError(err)
              → navigate('PublicErrorBootstrapScreen')
                → usuário aciona retry → navigate('PrivatePostLoginLoadingScreen')
                  → PrivatePostLoginLoadingScreen chama authStore.bootstrap() novamente
                    → sucesso → navega para PrivateHomeScreen
                    → falha   → reportCriticalError(err) → PublicErrorBootstrapScreen novamente
```

**`PublicErrorGenericScreen`** — tela de erro geral, usada por qualquer tela do app para erros inesperados não cobertos por uma tela específica. A ação de recuperação é definida pelo contexto — tipicamente voltar ao início ou tentar novamente.

O padrão para erros inesperados em telas é sempre o mesmo:

```
tela captura exceção não reconhecida no catch
  → useReportCriticalError(err)
    → errorStore.setError(err)
    → navigation.navigate('PublicErrorGenericScreen')
      → tela lê o erro do errorStore
      → exibe mensagem e ação de recuperação
      → ação de recuperação: errorStore.clearError() → navega de volta
```

Para evitar repetição, um hook encapsula as duas chamadas:

```typescript
// hooks/commons/useReportCriticalError.ts
export function useReportCriticalError() {
  const navigation = useNavigation();
  return (err: unknown) => {
    errorStore.setError(err);
    navigation.navigate("PublicErrorGenericScreen");
  };
}
```

**`errorStore`** é o canal de comunicação entre quem detecta o erro e a tela de erro. Não passa o erro via params de navegação — isso violaria a regra de params como identificadores.

**O que toda tela de erro deve fazer:**

- Informar que o sistema apresentou um erro e orientar o usuário (tentar novamente ou contactar o suporte)
- Oferecer a ação de recuperação adequada ao contexto
- Exibir detalhes técnicos do erro (código, serviço, timestamp) para apoiar o suporte
- **Nunca expor dados sensíveis** — token JWT, senhas e dados pessoais (nome, email, CPF) nunca aparecem

---

## S4 — Padrões de Interação

> Leia esta seção ao implementar qualquer tela com loading, erro, lista vazia ou formulário.

---

### Loading

O projeto tem dois contextos de loading com comportamentos distintos.

#### Loading local — busca inicial de dados

**Telas com formulário ou conteúdo único** — exibe `ActivityIndicator` centralizado enquanto aguarda.

```typescript
return loading ? <ActivityIndicator style={{ flex: 1 }} /> : <conteúdo />
```

**Telas com lista** — exibe `SKELETON_COUNT` repetições do `SkeletonCard` enquanto aguarda. O skeleton é um card cinza claro com animação de shimmer — único componente, independente do tipo de item da lista.

```typescript
return loading ? (
  <>
    {Array.from({ length: SKELETON_COUNT }).map((_, i) => (
      <SkeletonCard key={i} />
    ))}
  </>
) : <FlatList ... />
```

`SKELETON_COUNT` vem de `config`:

```typescript
export const SKELETON_COUNT = Number(process.env.EXPO_PUBLIC_SKELETON_COUNT ?? 5)
```

`SkeletonCard` fica em `atom/feedback/SkeletonCard.tsx`.

#### Overlay global — ações do usuário

Usado quando o usuário executa uma ação que chama a API (salvar, inscrever, deletar, login, logout, etc.). Bloqueia a tela inteira via `loadingStore`, impedindo duplo-clique e navegação acidental durante a operação.

**Regra:** toda operação `async` disparada pelo usuário — via `service` ou via `store action async` — usa o overlay global. Sem exceção.

Operações síncronas de store (`setUser`, `clearError`, `setError`) **não** usam o overlay — são instantâneas.

O `LoadingOverlay` é um `Modal` transparente com `ActivityIndicator` renderizado na raiz do app, acima de tudo (tabs, header, navegação).

---

### Erros

O projeto tem cinco tipos de erro com apresentações distintas.

#### `showToast` — wrapper obrigatório

Toda chamada ao toast passa por `showToast` em `utils/toast/toast.ts`. Nunca chamar a lib de toast diretamente. Os defaults ficam em um único lugar.

```typescript
// utils/toast/toast.ts
import { TOAST_VISIBILITY_MS } from '@/config'

export function showToast(params: ToastShowParams) {
  Toast.show({
    position: 'top',
    visibilityTime: TOAST_VISIBILITY_MS,
    ...params,
  })
}
```

O tempo de exibição é configurável em `src/config`:

```typescript
// src/config/index.ts
export const TOAST_VISIBILITY_MS = Number(
  process.env.EXPO_PUBLIC_TOAST_VISIBILITY_MS ?? 30_000
)
```

#### Toast types padrão

Três types são definidos uma vez na configuração do `react-native-toast-message`:

| Type | Ícone | Quando usar |
|---|---|---|
| `error` | ícone de erro/alerta | Erro de negócio da API |
| `networkError` | ícone de rede/wifi | Sem conexão, timeout |
| `success` | ícone de check | Ação concluída com sucesso |

Todos seguem o mesmo comportamento: topo da tela, `TOAST_VISIBILITY_MS` para sumir, swipe para fechar, toque pausa o contador.

#### Tipo 1 — Validação de formulário

Aparece inline, diretamente abaixo do campo com problema.

- Ao sair do campo (`onBlur`) — valida apenas o campo que perdeu foco
- Campo não tocado nunca exibe erro, mesmo se obrigatório
- No submit — valida todos os campos, incluindo os não tocados

```typescript
<TextInput
  error={touched.nome && !!errors.nome}
  onBlur={() => setTouched(t => ({ ...t, nome: true }))}
/>
{touched.nome && errors.nome && (
  <HelperText type="error">{errors.nome}</HelperText>
)}
```

Se o erro puder ser mapeado para um campo específico (ex: "email já cadastrado" vindo da API), trata como validação de formulário — aparece inline no campo correspondente.

#### Tipo 2 — Erro de negócio da API

A API processou a requisição e retornou um erro de regra de negócio que não pode ser atribuído a um campo específico.

**Como aparece:** banner flutuante no topo da tela via toast, sobreposto ao conteúdo sem deslocar o layout. Some automaticamente, mas pode ser fechado por swipe. Tocar pausa o contador.

```typescript
showToast({ type: 'error', text1: t('error.title'), text2: err.message })
```

#### Tipo 3 — Sem conexão / timeout

A requisição não chegou ao servidor. Usa toast com type específico de rede (definido no `30+`).

A função `isNetworkError` fica em `utils/error/classifier.ts` e é a única responsável por distinguir erros de rede de erros de negócio.

#### Tipo 4 — Sessão expirada (401)

Interceptado pelo `httpClient` e tratado exclusivamente pelo `AuthContext` — nenhuma tela trata 401 diretamente.

```
httpClient recebe 401 → handler do AuthContext → showToast → authStore.clearAuth() → AppNavigator redireciona para login
```

#### Tipo 5 — Erro crítico / inesperado

Último recurso do `catch` — não é de rede, não é de negócio, não foi previsto. Vai para `errorStore` + `PublicErrorGenericScreen` via `useReportCriticalError()`.

**Hierarquia de decisão no `catch`:**

```typescript
} catch (err) {
  if (isNetworkError(err)) {
    showToast({ type: 'networkError', ... })
  } else if (err instanceof ApiError) {
    showToast({ type: 'error', text2: err.message, ... })
  } else {
    reportCriticalError(err)
  }
}
```

**Regra:** todo `catch` de tela segue essa hierarquia. Nunca silenciar um erro com `catch (err) {}` vazio.

#### Hooks de ação — `useErrorHandler` e `useAsyncAction`

Para evitar repetição do mesmo bloco de loading + erro em toda tela, dois hooks encapsulam essas responsabilidades.

**`useErrorHandler`** — classifica e apresenta o erro automaticamente. Fica em `hooks/commons/useErrorHandler.ts`.

```typescript
export function useErrorHandler() {
  const reportCritical = useReportCriticalError()
  const { t } = useTranslation()

  return (err: unknown) => {
    if (isNetworkError(err)) {
      showToast({ type: 'networkError', text1: t('error.network.title'), text2: t('error.network.message') })
    } else if (err instanceof ApiError) {
      showToast({ type: 'error', text1: t('error.title'), text2: err.message })
    } else {
      reportCritical(err)
    }
  }
}
```

**`useAsyncAction`** — envolve qualquer ação assíncrona com overlay de loading e tratamento de erro. Fica em `hooks/commons/useAsyncAction.ts`.

```typescript
interface AsyncActionOptions {
  onSuccess?: () => void
  onError?: (err: unknown) => boolean | void
  overlay?: boolean  // default: true — false para telas de ação em lote
}

export function useAsyncAction() {
  const handleError = useErrorHandler()

  return async (action: () => Promise<void>, options?: AsyncActionOptions) => {
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

`onError` retorna `true` se tratou o erro — o handler padrão não é chamado. Sem retorno, cai no handler automaticamente.

**Ação em lote com `overlay: false`** — para telas com múltiplas ações em sequência (ex: marcar presenças), desabilitar o overlay e gerenciar loading localmente por item:

```typescript
const [loadingId, setLoadingId] = useState<string | null>(null)

const handleMark = (id: string) => runAction(
  async () => { setLoadingId(id); await service.mark(id) },
  {
    overlay: false,
    onSuccess: () => { setLoadingId(null); scheduleRefresh() },
    onError: () => { setLoadingId(null); return false },
  }
)
```

**Debounce para recarregar lista** — após ações em lote, a lista recarrega quando o usuário para de interagir. Tempo configurável em `src/config`:

```typescript
export const BATCH_ACTION_DEBOUNCE_MS = Number(
  process.env.EXPO_PUBLIC_BATCH_DEBOUNCE_MS ?? 5_000
)
```

```typescript
const refreshTimer = useRef<ReturnType<typeof setTimeout> | null>(null)

const scheduleRefresh = () => {
  if (refreshTimer.current) clearTimeout(refreshTimer.current)
  refreshTimer.current = setTimeout(loadList, BATCH_ACTION_DEBOUNCE_MS)
}
```

---

### Listas paginadas

O modelo de paginação é sempre por cursor. Qual dos dois casos abaixo usar é ditado pela regra de negócio — não por preferência técnica.

---

#### Caso 1 — Full scan: carregar tudo, tratar no frontend

Usado quando a operação exige acesso ao conjunto completo de dados — ordenação arbitrária, filtros cruzados, seleção múltipla, exportação. O frontend busca todos os registros em lotes via cursor até o fim e aplica ordenação, filtragem e exibição localmente.

Exemplos: administrar todos os usuários, gerenciar catálogo de produtos.

```typescript
const [items, setItems] = useState<Item[]>([])
const [isLoading, setIsLoading] = useState(false)

async function loadAll(): Promise<void> {
  setIsLoading(true)
  const all: Item[] = []
  let cursor: string | null = null

  try {
    do {
      const res = await itemService.list({ cursor: cursor ?? undefined, pageSize: PAGE_SIZE })
      all.push(...res.data)
      cursor = res.pagination.nextCursor
    } while (cursor)
    setItems(all)
  } finally {
    setIsLoading(false)
  }
}
```

Filtro, ordenação e busca são feitos sobre `items` com `useMemo` — sem nova chamada à API.

> O limite de volume viável para o Caso 1 depende do projeto, da tela e da regra de negócio. Quando o volume comprometer a performance do frontend, o filtro deve migrar para o backend — decisão documentada no arquivo `30+` do domínio.

---

#### Caso 2 — Continuação: carregar sob demanda

Usado quando os dados têm uma ordem natural (mais recentes, por data de criação) e o usuário avança conforme necessita. O frontend busca o primeiro lote e exibe um botão "Carregar mais" enquanto houver cursor. Scroll infinito não é usado — o usuário controla explicitamente quando buscar mais itens.

Exemplos: listar os 50 eventos mais recentes, mostrar produtos por ordem de criação.

```typescript
const [items, setItems] = useState<Item[]>([])
const [nextCursor, setNextCursor] = useState<string | null>(null)
const [loadingMore, setLoadingMore] = useState(false)
const runAction = useAsyncAction()

const loadMore = () => runAction(
  async () => {
    setLoadingMore(true)
    const res = await itemService.list({ cursor: nextCursor ?? undefined, pageSize: PAGE_SIZE })
    setItems(prev => [...prev, ...res.data])
    setNextCursor(res.pagination.nextCursor)
  },
  { overlay: false, onSuccess: () => setLoadingMore(false), onError: () => { setLoadingMore(false); return false } }
)

const handleRefresh = () => {
  setItems([])
  setNextCursor(null)
  loadInitial()
}

return (
  <FlatList
    data={items}
    keyExtractor={item => item.id}
    renderItem={({ item }) => <ItemCard item={item} />}
    onRefresh={handleRefresh}
    refreshing={refreshing}
    ListFooterComponent={
      nextCursor ? (
        <Button loading={loadingMore} onPress={loadMore}>
          {t('action.loadMore')}
        </Button>
      ) : null
    }
    ListEmptyComponent={!isLoading ? <EmptyState ... /> : null}
  />
)
```

---

O tamanho do lote vem de `config`:

```typescript
// src/config/index.ts
export const PAGE_SIZE = Number(process.env.EXPO_PUBLIC_PAGE_SIZE ?? 20)
```

> Este documento não cobre WebSockets, SSE ou qualquer mecanismo de atualização em tempo real. Projetos que necessitem dessas funcionalidades devem documentá-las em um arquivo `30+`.

---

### Estados Vazios

Quando uma lista não tem itens, o componente `EmptyState` substitui o conteúdo.

- Ícone + mensagem contextual definidos pela tela — nunca genérico (`"Nenhum item"`)
- Centralizado vertical e horizontalmente na tela
- Ícone discreto — pequeno, acima da mensagem
- Ação opcional — a tela passa `action` apenas quando faz sentido pelo negócio

```typescript
// atom/feedback/EmptyState.tsx
interface EmptyStateProps {
  icon: string     // nome do ícone (ex: MaterialCommunityIcons)
  message: string  // mensagem contextual — sempre via t()
  action?: {
    label: string
    onPress: () => void
  }
}
```

**FAB** é um padrão separado do `EmptyState`. Aparece nas telas que permitem adição, independente de haver itens ou não.

Regras:
- O ícone varia conforme a ação da tela — não é sempre `plus`
- FAB e botão no header são ambos válidos — a tela decide conforme o contexto
- O FAB nunca cobre conteúdo: o `FlatList` recebe `contentContainerStyle={{ paddingBottom: FAB_HEIGHT }}` para o último item respirar acima do botão
- Quantidade de FABs por tela: perguntar ao humano — decisão específica de cada tela

---

### Modais vs Navegação

**Regra: preferir navegação a modal.** Toda interação que exigiria um modal — formulário, detalhe, confirmação de dados — vira uma tela nova com `navigate`.

Modal só é justificado quando for impossível resolver com navegação — quando o bloqueio da tela de origem é parte essencial da experiência, não uma conveniência. Se houver dúvida, é uma tela nova.

---

### Layouts de tela

Toda tela usa um dos layouts abaixo como esqueleto estrutural. Nunca montar a estrutura de scroll, teclado e rodapé diretamente na tela — o layout encapsula isso.

#### `ListScreenLayout`

Para telas com lista de dados. Gerencia o `FlatList` com pull-to-refresh, skeleton, "carregar mais" e FAB opcional.

```
SafeAreaView
└── FlatList (pull-to-refresh, skeleton, ListFooterComponent)
FAB (absolute, canto inferior direito — opcional)
```

```typescript
// layouts/list/ListScreenLayout.tsx
interface ListScreenLayoutProps<T> {
  data: T[]
  loading: boolean
  nextCursor: string | null
  loadingMore: boolean
  onRefresh: () => void
  onLoadMore: () => void
  renderItem: (item: T) => React.ReactElement
  keyExtractor: (item: T) => string
  emptyState: React.ReactElement
  fab?: { icon: string; onPress: () => void }
}
```

#### `FormScreenLayout`

Para telas com formulário. Gerencia `KeyboardAvoidingView`, scroll dos campos e rodapé fixo com o botão de salvar.

```
KeyboardAvoidingView
└── ScrollView (campos do formulário)
Footer fixo (botão salvar — fora do scroll)
```

```typescript
// layouts/forms/FormScreenLayout.tsx
interface FormScreenLayoutProps {
  children: React.ReactNode        // campos do formulário
  onSave: () => void
  saveLabel?: string               // default: t('action.save')
  saving?: boolean                 // loading no botão
  saveDisabled?: boolean           // desabilita o botão
}
```

```typescript
// uso na tela
return (
  <FormScreenLayout
    onSave={handleSave}
    saveDisabled={Object.keys(errors).length > 0}
  >
    <TextInput label={t('person.name')} ... />
    <TextInput label={t('person.email')} ... />
  </FormScreenLayout>
)
```

**`KeyboardAvoidingView`:** `behavior="padding"` no iOS, sem behavior no Android — o Android ajusta via `windowSoftInputMode` no `app.json`.

---

### Formulários

#### Stack de validação

- **`react-hook-form`** — gerencia valores, erros e estado do formulário
- **`zod`** — define o schema de validação com tipos TypeScript gerados automaticamente
- **`zod-i18n-map`** — integra zod com o sistema de i18n do projeto para traduzir mensagens de validação

#### Comportamento de validação

- **onBlur** — ao sair de um campo, valida aquele campo. Campos não tocados não exibem erro.
- **No submit** — valida todos os campos, incluindo os não tocados.
- **Botão de submit** — habilitado no estado inicial. Desabilitado quando há erros visíveis: `disabled={Object.keys(errors).length > 0}`.

#### `useFormAction`

Hook que encapsula todo o ciclo de um formulário: validação, check de dirty, overlay de loading e tratamento de erro. Usado em **todos os formulários** — criação e edição.

```typescript
// hooks/commons/useFormAction.ts
export function useFormAction<T>(form: UseFormReturn<T>) {
  const runAction = useAsyncAction()
  const navigation = useNavigation()
  const { t } = useTranslation()
  const errorCount = useRef(0)

  return (action: (data: T) => Promise<void>, options?: AsyncActionOptions) => {
    return form.handleSubmit(
      async (data) => {
        errorCount.current = 0
        if (!form.formState.isDirty) {
          navigation.goBack()  // nada mudou — finge que salvou, sem chamar a API
          return
        }
        await runAction(() => action(data), options)
      },
      () => {
        errorCount.current += 1
        if (errorCount.current >= 3) {
          errorCount.current = 0
          showToast({ type: 'error', text1: t('form.hasErrors.title'), text2: t('form.hasErrors.message') })
        }
      }
    )
  }
}
```

**Fluxo interno:**
1. Valida todos os campos — marca os não tocados com erro se inválidos
2. Se inválido 3 vezes seguidas — exibe toast orientando o usuário a verificar os campos (resolve campos fora da área visível em telas pequenas)
3. Se nada mudou (`isDirty = false`) — navega de volta sem chamar a API
4. Executa a ação via `useAsyncAction` — overlay + tratamento de erro

#### `useDestructiveAction`

Hook para ações irreversíveis. Exibe dialog de confirmação, aguarda resposta, executa apenas no OK.

**Regra padrão: toda ação irreversível usa `useDestructiveAction`.** A exceção é quando um humano expressa que o dialog deve ser removido — nesse caso, documentar com comentário no código.

```typescript
// hooks/commons/useDestructiveAction.ts
export function useDestructiveAction() {
  const runAction = useAsyncAction()
  const { t } = useTranslation()
  const [visible, setVisible] = useState(false)
  const [dialog, setDialog] = useState<{ title: string; message: string } | null>(null)
  const pendingAction = useRef<(() => void) | null>(null)

  const destructiveAction = (
    action: () => Promise<void>,
    options?: AsyncActionOptions & { title?: string; message?: string }
  ) => {
    setDialog({
      title: options?.title ?? t('dialog.destructive.title'),
      message: options?.message ?? t('dialog.destructive.message'),
    })
    pendingAction.current = () => runAction(action, options)
    setVisible(true)
  }

  const handleConfirm = () => { setVisible(false); pendingAction.current?.() }

  const ConfirmDialog = () => (
    <Dialog visible={visible} onDismiss={() => setVisible(false)}>
      <Dialog.Title>{dialog?.title}</Dialog.Title>
      <Dialog.Content><Text>{dialog?.message}</Text></Dialog.Content>
      <Dialog.Actions>
        <Button onPress={() => setVisible(false)}>{t('action.cancel')}</Button>
        <Button onPress={handleConfirm}>{t('action.confirm')}</Button>
      </Dialog.Actions>
    </Dialog>
  )

  return { destructiveAction, ConfirmDialog }
}
```

A mensagem do dialog é passada pela tela — contextual e específica, nunca genérica:

```typescript
const { destructiveAction, ConfirmDialog } = useDestructiveAction()

const handleDelete = () => destructiveAction(
  async () => { await service.delete(id); showToast(...); navigation.goBack() },
  { message: t('entity.deleteConfirm', { name: entity.name }) }
)

return (
  <>
    <Button onPress={handleDelete}>{t('action.delete')}</Button>
    <ConfirmDialog />
  </>
)
```

#### Separação de handlers e JSX

Handlers são definidos como `const` antes do `return` — nunca como funções inline no JSX.

```typescript
export function SomeScreen() {
  const form = useForm<FormData>({ defaultValues: entity })
  const runFormAction = useFormAction(form)
  const runAction = useAsyncAction()

  const handleSave = runFormAction(async (data) => {
    await service.update(id, data)
    showToast({ type: 'success', ... })
    navigation.goBack()
  })

  const handleDelete = () => runAction(async () => {
    await service.delete(id)
    navigation.goBack()
  })

  return (
    <>
      <TextInput label={t('entity.name')} {...form.register('name')} />
      <Button onPress={handleSave}>{t('action.save')}</Button>
      <Button onPress={handleDelete}>{t('action.delete')}</Button>
    </>
  )
}
```

Funções inline no `onPress` são permitidas apenas para casos triviais: `onPress={() => navigation.goBack()}`.

#### Feedback de sucesso

Toda ação que conclui com sucesso exibe um toast `success` — sem exceção. Mesmo quando a tela navega de volta, o toast aparece sobre a tela de destino.

**Regra de navegação após sucesso:**
- Formulários (criação e edição) → toast + `goBack()`
- Ações de propósito único (deletar, confirmar) → toast + `goBack()`
- Ações em lote (marcar presenças, toggles em lista) → toast por ação + debounce para recarregar lista

#### Campos obrigatórios

Indicados com asterisco após o label: `Nome *`. Uma legenda discreta no topo do formulário explica o asterisco — uma vez, não em cada campo.

```typescript
<Text variant="bodySmall">{t('form.requiredFieldsNote')}</Text>
<TextInput label={`${t('entity.name')} *`} ... />  // obrigatório
<TextInput label={t('entity.phone')} ... />          // opcional
```

**Quando usar cada hook:**

| Hook | Quando usar |
|---|---|
| `useFormAction` | Todo formulário — criação e edição |
| `useAsyncAction` | Ações sem formulário e sem confirmação |
| `useDestructiveAction` | Ações irreversíveis que precisam de confirmação |
