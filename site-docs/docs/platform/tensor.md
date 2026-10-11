# Tensor & Autograd

The Tensor track (spec P7), Autograd (P8) and NN modules (P9) give the JVM an
engine-neutral math stack: tensors with a backend SPI, a tape-based gradient
graph with differential tests, and composable neural modules. Modules:
`cortexj-tensor-api` + backends, `cortexj-autograd`, `cortexj-nn`.

## Tensors

`Tensor`, `Shape`, `DType`, `Device`, `Layout` and the `TensorBackend` SPI.
Tensors are `AutoCloseable` — close what you create.

```java
TensorBackend cpu = new CpuBackend();
Tensor a = cpu.create(new double[]{1, 2, 3, 4}, Shape.of(2, 2), DType.F32, Device.CPU);
Tensor b = cpu.create(new double[]{5, 6, 7, 8}, Shape.of(2, 2), DType.F32, Device.CPU);

Tensor c = cpu.matmul(a, b);
c.get(0, 1);          // scalar read by index
c.toArray();          // copy out (never the internal storage)
c.isView();           // views alias parent storage (ADR-024: closing a view never frees the parent)
```

### Backend operations

Every backend implements the same surface (parity-checked):

- creation: `create`, `zeros`, `fill`, `random` (seeded, reproducible);
- math: `matmul`, `batchMatmul`, `add`, `mul`, `sum`, `mean`;
- activations: `relu`, `softmax`, `sigmoid`, `tanh`, `gelu`;
- normalization: `layerNorm`;
- indexing: `gather`, `scatter`;
- shape: `reshape`, `transpose`.

### Backends

| Backend | Module | Role |
|---|---|---|
| CPU reference | `cortexj-tensor-cpu` | the correctness baseline — plain Java, runs everywhere |
| Vector | `cortexj-tensor-vector` | Java Vector API path (JIT-vs-vector trade-offs documented in ADR-025; raw backend numbers are never attributed to the platform) |
| LibTorch | `cortexj-tensor-libtorch` | optional native bridge (PyTorch interop, spec P5); skips loudly without the native engine |

The **conformance suite** runs every backend against the same parity checks
(`TensorConformanceTest`): a backend that disagrees with the reference fails its
tests, not your training.

## Autograd

`cortexj-autograd` is a tape-based graph: operations record nodes, `backward()`
walks the tape and populates `Variable.grad()`.

```java
Autograd autograd = new Autograd(BACKEND);
Variable w1 = autograd.variable(new double[]{0.3, -0.2}, Shape.of(2));
Variable b1 = autograd.variable(new double[]{0.1, 0.1}, Shape.of(2));
Variable w2 = autograd.variable(new double[]{0.5, -0.4}, Shape.of(2));
Variable x  = autograd.constant(new double[]{1.0, 1.0}, Shape.of(2));

// a tiny MLP: x -> linear(w1, b1) -> relu -> linear(w2)
Variable hidden = autograd.relu(autograd.add(autograd.mul(w1, x), b1));
Variable output = autograd.mul(w2, hidden);
Variable loss   = autograd.mean(output);

autograd.backward(loss);
// w1.grad(), w2.grad(), b1.grad() now hold the gradients
```

Operations: `variable` / `constant` (graph entry), `add`, `mul`, `matmul`,
`sum`, `mean`, `softmax`, `relu`, plus:

- **custom gradient rules** — `record(output, inputs, backwardFn)` registers
    your own backward for any composite op;
- **detach** — cut a subgraph out of gradient tracking;
- **no-grad scopes and gradient hooks** — `AutogradPolicies`
    (`AutogradPoliciesTest` pins the contract);
- **tape lifecycle** — `clearTape()` between iterations.

Correctness is **differential**: every op's gradients are compared against
finite-difference references within tolerance (`AutogradDifferentialTest`) —
the suite that lets you trust the numbers.

## NN modules (spec P9)

`cortexj-nn` provides the module API — parameter registration, reusable layers,
state-dict round-trip, train/eval mode:

- `Linear` — the dense layer with weight/bias parameters;
- **sharded linear** — the column/row sharding used by distributed training;
- **pipeline** — composing modules into a chain;
- **authoring custom modules in user code** — `ModuleAuthoringTest` implements a
    custom attention-style scoring module with parameters, state-dict
    round-trip and train/eval mode *without any framework changes*.

## The stack in one picture

```text
nn modules  →  autograd (tape, gradients)  →  tensor backends (CPU/Vector/LibTorch)
                     ↓                                   ↓
             training (LoRA, plans)              parity conformance
```

## Next steps

- [Training & PEFT](training.md) — what this math stack serves.
- [Data](data.md) — batching before the math.
- [The Runtime](../index.md) — where trained adapters deploy.
