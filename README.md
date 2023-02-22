# tracked-queue

:warning:**NOTA:** este é o README para a próxima versão v2! Para o README v1, veja [aqui](https://github.com/linkedin/tracked-queue/blob/e934485f04db56b9bd64ebc66eb0f21006d2d6ae/README.md). :warning:

<!--[![npm(https://img.shields.io/npm/v/tracked-queue.svg])](https://www.npmjs.com/package/tracked-queue)-->

[![CI](https://github.com/linkedin/tracked-queue/actions/workflows/CI.yml/badge.svg)](https://github.com/linkedin/tracked-queue/actions/workflows/CI.yml) [![Versões de TypeScript suportadas](https://img.shields.io/badge/TypeScript-4.4%20%7C%204.5%20%7C%204.6%20%7C%20next-3178c6)](https://github.com/linkedin/tracked-queue/blob/main/.github/workflows/CI.yml#L82) <!--[![Nightly TypeScript Run](https://github.com/linkedin/tracked-queue/actions/workflows/Nightly%20TypeScript%20Run.yml/badge.svg)](https://github.com/linkedin/tracked-queue/actions/workflows/Nightly%20TypeScript%20Run.yml)-->

Uma implementação [autotracked](https://v5.chriskrycho.com/journal/autotracking-elegant-dx-via-cutting-edge-cs/) de uma fila dupla, implementada como um buffer de anel apoiado por um nativo Matriz JavaScript, com desempenho ideal para todas as operações comuns:

- _O(1)_ empurrar e pop de qualquer extremidade da fila;
- _O(1)_ ler de qualquer elemento na fila;
- _O(N)_ iteração de toda a fila;
- _O(N)_ acesso a qualquer intervalo de tamanho _N_ dentro da fila;
- _O(N+X)_ armazenamento para uma fila de tamanho _N_, com _X_ uma sobrecarga fixa na ordem de algumas dezenas de bytes:
  - armazenamento de apoio de tamanho `N`;
  - um único valor de capacidade;
  - dois ponteiros para o armazenamento, com o custo adicional de uma "tag" Glimmer para cada um deles.

Isso é útil para casos em que você precisa ser capaz de empurrar e retirar itens de qualquer extremidade de uma fila de tamanho fixo, especialmente onde você deseja ter leituras rápidas de qualquer lugar na fila.

Por exemplo, se você tiver um fluxo de eventos vindo de uma conexão websocket, que você deseja exibir para um usuário, mas você deseja manter apenas 1.000 deles na memória a qualquer momento, isso permite que você simplesmente envie para o volta da fila à medida que novos itens chegam, sem a necessidade de gerenciar manualmente a frente da fila para manter o tamanho da fila.

<!-- omit in toc -->
## Conteúdo

- [Example](#example)
- [Compatibility](#compatibility)
  - [TypeScript](#typescript)
  - [Browser support](#browser-support)
- [Installation](#installation)
- [Docs](#docs)
- [Performance](#performance)
- [Contributing](#contributing)
- [License](#license)

## Exemplo

Crie uma fila de uma capacidade especificada e, em seguida, envie itens para ela a partir de matrizes existentes ou valores individuais:

```ts
//Crie uma fila com capacidade 5, a partir de uma matriz existente de elementos:
import TrackedQueue from 'tracked-queue';
let queue = new TrackedQueue<string>({ capacity: 5 });

queue.append(['alpha', 'bravo', 'charlie']);
console.log(queue.size); // 3
console.log([...queue]); // ["alpha", "bravo", "charlie"]

queue.pushBack('delta');
console.log(queue.size); // 4
console.log([...queue]); // ["alpha", "bravo", "charlie", "delta"]
```

Se você adicionar mais elementos no final da fila, excedendo sua capacidade, ele será descartado na frente da fila, com o 'tamanho' da fila nunca excedendo sua capacidade especificada, e quaisquer itens removidos para liberar espaço serão retornados. (O mesmo vale para `prepend`, na frente da fila.)

```ts
let pushedOut = queue.append(["echo", "foxtrot");
console.log(queue.size); // 5
console.log([...queue]); // ["bravo", "charlie", "delta", "echo", "foxtrot"]
console.log(pushedOut); // ["alpha"]
```

Você também pode adicionar e remover itens de qualquer extremidade da fila:

```ts
let poppedFromBack = queue.popBack();
console.log(poppedFromBack); // "foxtrot"
console.log([...queue]); // ["bravo", "charlie", "delta", "echo"]

let poppedFromFront = queue.popFront();
console.log(poppedFromFront); // "bravo"
console.log([...queue]); // ["charlie", "delta", "echo"]

queue.pushBack('golf');
queue.pushFront('hotel');
console.log([...queue]); // ["hotel", "charlie", "delta", "echo", "golf"]
```

Estes também retornam um valor que teve que ser removido para abrir espaço para eles, se houver:

```ts
let poppedByPush = queue.pushBack('india');
console.log([...queue]); // ["charlie", "delta", "echo", "golf", "india"]
console.log(poppedByPush); // "hotel"
// tornar a fila não vazia
queue.popFront();
let nothingPopped = queue.pushFront('juliet');
console.log([...queue]); // ["juliet", "delta", "echo", "golf", "india"]
console.log(poppedByPush); // undefined
```

## Compatibilidade

- Ember.js v3.16 or above
- Ember CLI v2.13 or above
- Node.js v12 or above

### TypeScript

Este projeto segue o rascunho atual da especificação [Semantic Versioning for TypeScript Types][semver].

- **Versões do TypeScript atualmente suportadas:** v4.4, v4.5, e v4.6
- **Compiler support policy:** [simple majors][sm]
- **Public API:** all published types not in a `-private` module are public

[semver]: https://www.semver-ts.org
[sm]: https://www.semver-ts.org/#simple-majors

### Browser support

Este projeto usa `Proxy` nativo (através de uma dependência) e, portanto, não é compatível com o IE11. Ele suporta N-1 para todos os outros navegadores.

## Installation

- With npm:

  ```sh
  npm install tracked-queue
  ```

- With yarn:

  ```sh
  yarn add tracked-queue
  ```

- With ember-cli:

  ```sh
  ember install tracked-queue
  ```

## Documentos

- Consulte [docs/API.md](./docs/API.md) para obter a documentação completa da API.
- Consulte [docs/under-the-hood.md](./docs/under-the-hood.md) para obter detalhes sobre como a fila funciona.

## Performance

O "aplicativo fictício" inclui duas demonstrações de desempenho, que você pode ver executando `ember serve` e navegando até `http://localhost:4200`:

- A render performance demo showing that the queue implementation itself is never the bottleneck for performance; the cost of rendering DOM elements is.

- Uma demonstração de desempenho operacional, que permite ver o comportamento de push e popping para frente e para trás da fila. (Esta é uma medição ingênua usando [a API `Performance`] [perf-api].) Algumas coisas a serem observadas:

  - Quando a capacidade da fila é muito maior do que o número de itens empurrados para dentro ou para fora dela, o desempenho de `pushBack` e `popBack` da parte de trás da fila é comparável às ações push e pop do array nativo, porque _é_ apenas essas operações mais o aumento do índice para o "back" da fila

  - Quando o número de itens empurrados excede a capacidade da fila, fazendo com que a fila “quebre”, os tempos de inserção `pushBack` e `popBack` caem para cerca de 100 × pior que nativos, mas o uso do espaço é limitado: `push` e `pop nativos ` são rápidos porque permitem que o buffer cresça de forma ilimitada. Esta é a compensação fundamental de um buffer de anel: ele fornece controle sobre o uso do espaço em troca de custos ligeiramente mais altos para push e pop no final da fila

  - O desempenho do `pushFront` e do `popFront` é cerca de 10 vezes pior do que o `unshift` e o `shift` nativos até um limite (que varia por navegador), ponto em que o desempenho dos métodos nativos se torna centenas de vezes pior e degrada rapidamente, a ponto de `pushFront` e `popFront` travar uma guia do navegador ao lidar com muito mais do que cerca de 100.000 operações, à medida que o redimensionamento do array e a movimentação de itens crescem de forma ilimitada. Enquanto isso, `pushFront` e `popFront` têm características de desempenho consistentes, não importa o tamanho da fila.
You can see these dynamics clearly by varying the number of operations to perform and the size of the queue.

[perf-api]: http://developer.mozilla.org/en-US/docs/Web/API/Performance

## Contribuindo

Consulte o guia [Contribuir](CONTRIBUTING.md) para obter detalhes.

## Licença

Este projeto está licenciado sob a [Licença BSD 2-Clause](LICENSE.md).
