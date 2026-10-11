# Tensor & Autograd

The Tensor track (spec P7), Autograd (P8) and NN modules (P9) give the JVM an
engine-neutral math stack: tensors with a backend SPI, a tape-based gradient
graph with differential tests, and composable neural modules.

## Tensors

`cortexj-tensor-api` defines `Tensor`, `Shape`, `DType`, `Device` and the
`TensorBackend` SPI. Backends ship as separate modules:

| Backend | Module | Notes |
|---|---|---|
| CPU reference | `cortexj-tensor-cpu` | the correctness baseline |
| Vector | `cortexj-tensor-vector` | Java Vector API path (JIT-vs-vector trade-offs documented in ADR-025) |
| LibTorch | `cortexj-tensor-libtorch` | optional native bridge (PyTorch interop, spec P5) |

```java
TensorBackend cpu = new CpuBackend();
Tensor a = cpu.create(new double[]{1, 2, 3, 4}, Shape.of(2, 2), DType.F32, Device.CPU);
Tensor b = cpu.create(new double[]{5, 6, 7, 8}, Shape.of(2, 2), DType.F32, Device.CPU);
Tensor c = cpu.matmul(a, b);
```

The conformance suite runs every backend against the same parity checks — a
backend that disagrees with the reference fails its tests, not your training.

## Autograd

`cortexj-autograd` is a tape-based graph: gradient tracking, `backward()`,
detach / no-grad scopes, custom gradient rules (`record`), gradient hooks and
checkpointing:

```java
Autograd autograd = new Autograd(BACKEND);
Variable w1 = autograd.variable(new double[]{0.3, -0.2}, Shape.of(2));
Variable x  = autograd.constant(new double[]{1.0, 1.0}, Shape.of(2));

Variable loss = autograd.mean(autograd.mul(w1, x));
autograd.backward(loss);
// w1.grad() now holds the gradient
```

Correctness is differential: every op's gradients are compared against
finite-difference references within tolerance (`AutogradDifferentialTest`).

## NN modules

`cortexj-nn` (spec P9) provides the module API: parameter registration, reusable
layers (`Linear`), pipelines, and sharded linear for distributed training. The
phase-C work added collectives-style sharding hooks used by
[multi-node training](training.md#distributed-training).

## Next steps

- [Training & PEFT](training.md) — what this math stack serves.
- [The Runtime](../index.md) — where trained adapters deploy.
