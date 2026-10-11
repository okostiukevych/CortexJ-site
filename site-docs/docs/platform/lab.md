# Lab & Notebook

The Lab track (spec P1) gives JVM developers an interactive AI workflow without
switching to a Python notebook. Everything runs on the JVM itself — no Python
kernel, no sidecar. Modules: `cortexj-lab-api` (JShell session),
`cortexj-notebook` (Jupyter kernel). Requires Java 17+ (ADR-017).

## LabSession

`LabSession` wraps the JDK's JShell in a managed, closable session. It inherits
your **process classpath**, so your application's classes — including `CortexJ`
itself — work directly in evaluated snippets:

```java
try (LabSession lab = LabSession.create()) {
    SnippetResult result = lab.eval("1 + 2");
    System.out.println(result.value());      // "3"

    // your code and CortexJ are right there:
    lab.eval("var platform = io.cortexj.CortexJ.builder().build();");
    lab.eval("platform.models()");
}
```

### SnippetResult — typed, never a stack-trace soup

```java
SnippetResult result = lab.eval("intentionallyBad()");
if (result.hasErrors()) {
    result.errors();     // compile/diagnostic messages, one per problem
} else {
    result.value();      // the evaluated value as a string
}
```

Compile errors and runtime failures come back as **structured messages** on the
result — a notebook UI renders them per-line instead of dumping a stack trace.

### Environment descriptor — reproducibly captured

```java
EnvironmentDescriptor env = lab.describeEnvironment();
env.javaVersion();       // e.g. "25.0.4"
env.jvmVendor();         // e.g. "Oracle Corporation"
env.osArch();            // e.g. "macos-aarch64"
env.availableCores();
env.cortexjVersion();
env.timestampMillis();   // captured at call time
```

Call it at the start of an interactive session and the snapshot describes the
environment that produced every number after it — the same shape
[Experiments](experiments.md) embed into runs.

### Display contract

`Displayable` renders values richly — `Displayable.table(headers, rows)`
produces the table representation a notebook front-end shows. Chart and image
renderers are tracked follow-ups, honestly not in 0.x.

### Notebook → Maven export

```java
lab.exportProjectSkeleton(Path.of("my-project"), "com.example.myproject");
```

Writes a runnable starter project (starter-java17 based) from the session's
work — the bridge from "played with it in a notebook" to "checked into the
build".

## Jupyter kernel

`cortexj-notebook` implements the Jupyter wire protocol (ZeroMQ, HMAC-signed
envelopes) as a Java kernel, with a live in-process contract test. Point
Jupyter at the kernel and evaluate JVM code; the kernel's session inherits the
process classpath, so Runtime model loads work interactively.

Requirements: Java 17+, a Jupyter front-end that speaks the wire protocol.
No Python in the execution path — the kernel is pure JVM.

## Interaction with the rest of the platform

- [Data](data.md) frames load and print in the session;
- Runtime model loads and generations run live (the session inherits the
    classpath);
- what you settle on becomes an [Experiment](experiments.md) or a Maven
    skeleton.

## Next steps

- [Experiments](experiments.md) — capture what you did interactively.
- [Data](data.md) — load and fingerprint datasets inside the session.
