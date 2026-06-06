---
name: cdk-domain-architecture
description: "Leia antes de criar ou modificar qualquer código CDK. Cobre estrutura de diretórios, organização por região e ambiente, components, stacks e config."
---

# CDK Domain Architecture

> Este documento define a arquitetura do projeto CDK — como organizar diretórios, stacks, components e configurações para suportar multi-região, múltiplos ambientes e reutilização. Para o padrão de deploy de Lambdas (Code.fromAsset, bundling), veja [09-lambda-domain-architecture.md](./09-lambda-domain-architecture.md).

---

## Index

| Seção | Quando consultar |
|---|---|
| [Estrutura de diretórios](#estrutura-de-diretórios) | Antes de criar qualquer arquivo no `cdk/` |
| [As três camadas](#as-três-camadas) | Antes de decidir onde colocar um novo arquivo |
| [Hierarquia de escopo](#hierarquia-de-escopo) | Ao entender o que cada nível cobre |
| [Nomenclatura de stacks](#nomenclatura-de-stacks) | Antes de criar uma nova stack |
| [Entrypoints e deploy](#entrypoints-e-deploy) | Antes de criar arquivos em `bin/` ou scripts npm |
| [Dependências entre stacks](#dependências-entre-stacks) | Ao compartilhar valores entre stacks |
| [Config nas stacks](#config-nas-stacks) | Ao passar configuração para uma stack |
| [Components e herança](#components-e-herança) | Ao criar ou estender um component |

---

## Estrutura de diretórios

```
cdk/
  bin/
    us-east-1-dev.ts              ← entrypoint completo de us-east-1/dev
    us-east-1-prd.ts              ← entrypoint completo de us-east-1/prd
    sa-east-1-dev.ts              ← entrypoint completo de sa-east-1/dev
    sa-east-1-prd.ts              ← entrypoint completo de sa-east-1/prd
  config/
    global/
      global-config.ts            ← valores que não dependem de região nem de ambiente
    region/
      us-east-1/
        shared-config.ts          ← valores compartilhados em us-east-1 (todos os envs)
        dev/
          dev-config.ts           ← valores de dev em us-east-1
        prd/
          prd-config.ts           ← valores de prd em us-east-1
      sa-east-1/
        shared-config.ts
        dev/
          dev-config.ts
        prd/
          prd-config.ts
  stacks/
    global/
      global-stack.ts             ← recursos AWS globais (Route53, IAM roles genéricas, policies reutilizáveis)
    region/
      components/                 ← components compartilhados entre todas as regiões
      us-east-1/
        components/               ← components específicos de us-east-1
        shared-stack.ts           ← recursos de us-east-1 que não são de dev nem prd
        dev/
          dev-stack.ts            ← stack principal de dev em us-east-1
          api-stack.ts            ← sub-stack de dev
          data-stack.ts           ← sub-stack de dev
        prd/
          prd-stack.ts
          api-stack.ts
          data-stack.ts
      sa-east-1/
        components/
        shared-stack.ts
        dev/
          dev-stack.ts
          api-stack.ts
          data-stack.ts
        prd/
          prd-stack.ts
          api-stack.ts
          data-stack.ts
  components/                     ← components globais, sem dependência de região
```

---

## As três camadas

### components/

O **como** — constrói um recurso AWS. Um component é um Construct CDK reutilizável: define como criar um API Gateway, uma tabela DynamoDB, uma Lambda com suas permissões.

Um component não sabe em qual região está nem para qual ambiente serve — recebe isso via parâmetros.

### stacks/

O **onde** — instancia e organiza components por região e ambiente. Uma stack é um composition root: importa components, passa a config correspondente, e define o que existe em cada contexto.

As stacks não contêm lógica de construção de recursos — delegam para os components.

### config/

O **quê** — nomes, valores e parâmetros que alimentam os components. A config não importa nem instancia nada do CDK — são objetos com valores.

---

## Hierarquia de escopo

Cada nível cobre um escopo distinto:

| Nível | Escopo | Exemplos |
|---|---|---|
| `config/global` | Recursos AWS sem região (globais por natureza) | Route53 hosted zones, IAM roles genéricas |
| `config/region/{região}/shared` | Recursos da região, independente de ambiente | VPC, certificados ACM regionais |
| `config/region/{região}/{env}` | Recursos de um ambiente específico dentro de uma região | Tabelas DynamoDB de dev, Lambdas de prd |
| `components/` (raiz) | Constructs sem dependência de região | Padrões genéricos reutilizáveis em qualquer região |
| `stacks/region/components/` | Constructs compartilhados entre todas as regiões | |
| `stacks/region/{região}/components/` | Constructs específicos de uma região | |

A hierarquia de config segue a mesma estrutura das stacks — cada stack consome a config do nível correspondente.

---

## Nomenclatura de stacks

O ID de cada stack é **PascalCase do caminho do diretório**, sem redundância:

| Arquivo | Stack ID |
|---|---|
| `stacks/global/global-stack.ts` | `Global` |
| `stacks/region/us-east-1/shared-stack.ts` | `UsEast1Shared` |
| `stacks/region/us-east-1/dev/dev-stack.ts` | `UsEast1Dev` |
| `stacks/region/us-east-1/dev/api-stack.ts` | `UsEast1DevApi` |
| `stacks/region/us-east-1/prd/data-stack.ts` | `UsEast1PrdData` |
| `stacks/region/sa-east-1/dev/api-stack.ts` | `SaEast1DevApi` |

---

## Entrypoints e deploy

### Por que um arquivo por região+ambiente (e não um único app com context parameters)

Um único `app.ts` com `-c region=us-east-1 -c env=dev` é o padrão mais comum, mas introduz lógica condicional no entrypoint para decidir quais stacks instanciar — e um `cdk synth` sem os contextos corretos quebra silenciosamente ou sintetiza tudo de uma vez.

Um arquivo por contexto é determinístico: zero lógica condicional, zero risco de sintetizar a stack errada. O custo é fixo (N arquivos); o ganho em previsibilidade e isolamento de deploy cresce com o número de regiões.

### Estrutura de bin/

Cada arquivo em `bin/` é um entrypoint CDK completo e independente — cria seu próprio `App`, instancia todas as stacks daquele contexto e chama `synth()`. As stacks de um mesmo ambiente (`dev-stack`, `api-stack`, `data-stack`) são independentes entre si — não há stack pai que instancia as demais. Nunca usar nested stacks.

```ts
// bin/us-east-1-dev.ts
const app = new cdk.App();
const env = { region: devConfig.region, account: devConfig.account };

new UsEast1Dev(app, "UsEast1Dev", { env, config: devConfig });
new UsEast1DevApi(app, "UsEast1DevApi", { env, config: devConfig });
new UsEast1DevData(app, "UsEast1DevData", { env, config: devConfig });

app.synth();
```

A `global-stack` tem entrypoint próprio (`bin/global.ts`) — recursos globais (Route53, IAM roles genéricas) não pertencem a nenhuma região e são deployados independentemente. Nunca instanciar a global-stack dentro de um entrypoint regional.

A `shared-stack` de cada região também tem entrypoint próprio (`bin/{região}-shared.ts`) — recursos regionais compartilhados (VPC, ACM) têm ciclo de vida independente dos ambientes e não são recriados a cada deploy de dev ou prd.

Adicionar uma nova região = criar o arquivo `bin/{região}-{env}.ts` e o script npm correspondente.

Não há `cdk.json` — cada script npm passa `--app` explicitamente e é autossuficiente. Rodar `cdk deploy` sem o script npm não é o fluxo esperado.

### Scripts npm

Um script por entrypoint (região + ambiente):

```json
"deploy:us-east-1:dev": "cdk deploy --app 'npx ts-node bin/us-east-1-dev.ts' --require-approval never --verbose",
"deploy:us-east-1:prd": "cdk deploy --app 'npx ts-node bin/us-east-1-prd.ts' --require-approval never --verbose",
"deploy:sa-east-1:dev": "cdk deploy --app 'npx ts-node bin/sa-east-1-dev.ts' --require-approval never --verbose",
"deploy:sa-east-1:prd": "cdk deploy --app 'npx ts-node bin/sa-east-1-prd.ts' --require-approval never --verbose"
```

### Deploy de stack individual

Feito via CLI diretamente — sem script npm dedicado:

```bash
cdk deploy --app 'npx ts-node bin/us-east-1-dev.ts' UsEast1DevApi
```

---

## Dependências entre stacks

Três mecanismos, cada um para um caso específico:

| Caso | Mecanismo |
|---|---|
| Valor previsível (nome que você define) | **config** |
| ID/ARN gerado pela AWS, stacks do mesmo ambiente | **CfnOutput + Fn.importValue** |
| Valor que cruza nível (global→regional, shared→env) | **SSM Parameter Store** |

**Árvore de decisão:**

```
O valor é previsível (você define o nome)?
  └─ sim → config
  └─ não (ID/ARN gerado pela AWS)
       └─ cruza fronteira de nível (entrypoints diferentes)?
            └─ sim → SSM Parameter Store
            └─ não (mesmo entrypoint / mesmo ambiente) → CfnOutput + Fn.importValue
```

**Exemplos canônicos:**

| Recurso | Criado por | Consumido por | Mecanismo | Motivo |
|---|---|---|---|---|
| Route53 Hosted Zone | `global-stack` | stack regional | SSM | ID gerado pela AWS, cruza global→regional |
| Certificado ACM | `us-east-1/shared-stack` | `us-east-1/dev-stack` | SSM | ARN gerado pela AWS, cruza shared→env |
| S3 Bucket | `us-east-1/dev-stack` | `us-east-1/dev/api-stack` | config | Nome definido por você, previsível |
| EC2 Instance | `us-east-1/dev-stack` | `us-east-1/dev/api-stack` | CfnOutput + importValue | ID gerado pela AWS, mesmo entrypoint |

### config

Quando o valor é previsível — você define o nome do bucket, da tabela, da função. A stack consumidora lê diretamente do config correspondente.

### CfnOutput + Fn.importValue

Quando o ID é gerado pela AWS mas as stacks têm vínculo direto (mesmo entrypoint, mesmo ambiente). Cada stack é explícita sobre o que exporta e o que importa:

```ts
// UsEast1Dev — exporta
new CfnOutput(this, "InstanceId", {
  value: this.instance.instanceId,
  exportName: "UsEast1Dev-InstancePrimaria-InstanceId",
});

// UsEast1DevApi — importa
const instanceId = Fn.importValue("UsEast1Dev-InstancePrimaria-InstanceId");
```

Convenção do `exportName`: `{StackId}-{nome-do-recurso}-{atributo}`

> **Limitação:** `Fn.importValue` só resolve dentro do mesmo CDK app. Stacks de entrypoints diferentes (ex: `global` e `us-east-1-dev`) não compartilham o app — use SSM nesses casos.

### SSM Parameter Store

Quando o valor cruza fronteiras de nível (global → regional, shared → env). Entrypoints diferentes não compartilham o mesmo CDK app, então `Fn.importValue` não funciona — SSM é o único mecanismo que atravessa essa fronteira em deploy time.

A stack criadora escreve no SSM; a consumidora lê com `ssm.StringParameter.valueForStringParameter()`.

**Convenção de path:** `/{escopo}/{tipo}/{nome-do-recurso}/{atributo}` em kebab-case:

```
/global/route53/example-com/hosted-zone-id
/us-east-1/shared/acm/api-example-com/cert-arn
/us-east-1/dev/ec2/ec2-dev-db-01/instance-id
```

O `{nome-do-recurso}` vem do config — quem cria e quem consome usam o mesmo nome para montar o path.

---

## Config nas stacks

A config é injetada na stack via `props.config` pelo entrypoint (`bin/`). A stack não importa o config diretamente. Isso vale para todos os níveis — global, shared e env seguem o mesmo padrão.

Quando uma stack de ambiente precisa de valores do shared (ex: nome da VPC), o `bin/` importa os dois configs e faz o merge antes de passar para as stacks — `dev-config.ts` não importa `shared-config.ts` diretamente.

```ts
// bin/us-east-1-dev.ts
import { sharedConfig } from "../config/region/us-east-1/shared-config";
import { devConfig } from "../config/region/us-east-1/dev/dev-config";

const config = { ...sharedConfig, ...devConfig };

new UsEast1Dev(app, "UsEast1Dev", { env, config });
```

```ts
// stacks/region/us-east-1/dev/db-stack.ts
import { UsEast1DevConfig } from "../../../../config/region/us-east-1/dev/dev-config";

interface DbStackProps extends StackProps {
  config: UsEast1DevConfig;
}

export class UsEast1DevDbStack extends Stack {
  constructor(scope: Construct, id: string, props: DbStackProps) {
    super(scope, id, props);

    new RdsAurora(this, "RdsAurora", {
      env: props.env,
      databaseName: props.config.databaseName,
      rootPassword: props.config.rootPassword,
    });
  }
}
```

O tipo do config é definido e exportado no próprio arquivo de config. A stack importa esse tipo para usar nas props — o tipo não é propagado além disso.

Todas as sub-stacks de um mesmo ambiente recebem o config completo via props — o entrypoint passa o mesmo objeto para cada uma. Não se criam tipos parciais por sub-stack.

Components nunca importam config diretamente — recebem **valores avulsos**, não o objeto de config:

```ts
new RdsAurora(this, "RdsAurora", {
  databaseName: props.config.databaseName,  // string
  rootPassword: props.config.rootPassword,  // string
});
```

---

## Components e herança

Os três níveis de components formam uma hierarquia de herança:

```
components/                          ← abstract base — define contrato e lógica comum
stacks/region/components/            ← abstract regional — estende com comportamento regional
stacks/region/{região}/components/   ← concreto — implementação específica da região
```

Components são instanciados diretamente nas stacks (`new UsEast1Lambda(this, "...", props)`) — sem factory ou helper. A stack é o composition root.

Use herança (não composição) porque o comportamento de cada nível é uma invariante estrutural — toda Lambda em `us-east-1` sempre tem VPC attachment e endpoints regionais, sem exceção. Não é comportamento opcional que se ativa por parâmetro; é parte da definição de "ser um component dessa região". A hierarquia de três níveis mapeia diretamente na hierarquia de diretórios.

A classe abstrata contém **lógica compartilhada + pontos de extensão** — não é apenas uma interface. Recursos comuns (IAM role base, log group, tags) são construídos no abstract; os filhos completam o que é específico.

```ts
// components/base-lambda.ts
abstract class BaseLambda extends Construct {
  protected readonly role: iam.Role;
  // constrói role base, log group comum
  // filhos implementam o handler específico
}

// stacks/region/components/regional-lambda.ts
abstract class RegionalLambda extends BaseLambda {
  // adiciona comportamento regional (VPC attachment, regional endpoints)
}

// stacks/region/us-east-1/components/us-east-1-lambda.ts
class UsEast1Lambda extends RegionalLambda {
  // implementação concreta para us-east-1
}
```
