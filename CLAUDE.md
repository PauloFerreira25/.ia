# Regras do Projeto

## JavaScript / TypeScript

- **Nunca** adicionar comentários `eslint-disable`, `eslint-disable-next-line` ou qualquer variante de supressão de ESLint em arquivos `.js`, `.ts`, `.jsx` ou `.tsx`.
- Se encontrar um `eslint-disable` existente no código, tentar resolver o problema subjacente que causou o erro de ESLint (corrigir o código para estar em conformidade com a regra).
- Se não souber como resolver o erro de ESLint sem desabilitar a regra, **parar e chamar um humano** — não contornar com disable.
