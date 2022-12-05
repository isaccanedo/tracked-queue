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
// Create a queue with capacity 5, from an existing array of elements:
import TrackedQueue from 'tracked-queue';
let queue = new TrackedQueue<string>({ capacity: 5 });

queue.append(['alpha', 'bravo', 'charlie']);
console.log(queue.size); // 3
console.log([...queue]); // ["alpha", "bravo", "charlie"]

queue.pushBack('delta');
console.log(queue.size); // 4
console.log([...queue]); // ["alpha", "bravo", "charlie", "delta"]
```

If you append more elements to the back of the queue, exceeding its capacity, it will drop the front of the queue, with the `size` of the queue never exceeding its specified capacity, and any items removed to make room are returned. (The same goes for `prepend`, on the front of the queue.)

```ts
let pushedOut = queue.append(["echo", "foxtrot");
console.log(queue.size); // 5
console.log([...queue]); // ["bravo", "charlie", "delta", "echo", "foxtrot"]
console.log(pushedOut); // ["alpha"]
```

You can also add and remove items to and from either end of the queue:

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

These also return a value which had to be popped to make room for them, if any:

```ts
let poppedByPush = queue.pushBack('india');
console.log([...queue]); // ["charlie", "delta", "echo", "golf", "india"]
console.log(poppedByPush); // "hotel"
// make the queue non-empty
queue.popFront();
let nothingPopped = queue.pushFront('juliet');
console.log([...queue]); // ["juliet", "delta", "echo", "golf", "india"]
console.log(poppedByPush); // undefined
```

## Compatibility

- Ember.js v3.16 or above
- Ember CLI v2.13 or above
- Node.js v12 or above

### TypeScript

This project follows the current draft of [the Semantic Versioning for TypeScript Types][semver] specification.

- **Currently supported TypeScript versions:** v4.4, v4.5, and v4.6
- **Compiler support policy:** [simple majors][sm]
- **Public API:** all published types not in a `-private` module are public

[semver]: https://www.semver-ts.org
[sm]: https://www.semver-ts.org/#simple-majors

### Browser support

This project uses native `Proxy` (via a dependency), and so is not compatible with IE11. It supports N-1 for all other browsers.

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

## Docs

- See [docs/API.md](./docs/API.md) for full API documentation.
- See [docs/under-the-hood.md](./docs/under-the-hood.md) for details on how the queue works.

## Performance

The "dummy app" includes two performance demos, which you can see by running `ember serve` and navigating to `http://localhost:4200`:

- A render performance demo showing that the queue implementation itself is never the bottleneck for performance; the cost of rendering DOM elements is.

- An ops performance demo, which allows you to see the behavior of pushing and popping to and from the front and back of the queue. (This is a naive measurement using [the `Performance` API][perf-api].) A few things to observe:

  - When the queue capacity is much larger than the number of items pushed into or popped out of it, the performance of `pushBack` and `popBack` from the back of the queue is comparable to native array push and pop actions, because it _is_ just those operations plus bumping the index for the "back" of the queue

  - When the number of items pushed exceeds the queue capacity, causing the queue to “wrap”, `pushBack` and `popBack` insertion times drops to around 100× worse than native, but space usage is bounded: native `push` and `pop` are fast because they allow the buffer to grow in an unbounded fashion. This is the fundamental tradeoff of a ring-buffer: it provides control over space usage in exchange for slightly higher costs for push and pop on the back of the queue

  - O desempenho do `pushFront` e do `popFront` é cerca de 10 vezes pior do que o `unshift` e o `shift` nativos até um limite (que varia por navegador), ponto em que o desempenho dos métodos nativos se torna centenas de vezes pior e degrada rapidamente, a ponto de `pushFront` e `popFront` travar uma guia do navegador ao lidar com muito mais do que cerca de 100.000 operações, à medida que o redimensionamento do array e a movimentação de itens crescem de forma ilimitada. Enquanto isso, `pushFront` e `popFront` têm características de desempenho consistentes, não importa o tamanho da fila.
You can see these dynamics clearly by varying the number of operations to perform and the size of the queue.

[perf-api]: http://developer.mozilla.org/en-US/docs/Web/API/Performance

## Contributing

See the [Contributing](CONTRIBUTING.md) guide for details.

## License

This project is licensed under the [BSD 2-Clause License](LICENSE.md).
