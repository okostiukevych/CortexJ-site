# Lab & Notebook

The Lab track (spec P1) gives JVM developers an interactive AI workflow without
switching to a Python notebook. Everything runs on the JVM itself — the whole
slice is built into the JDK, no Python kernel required.

## LabSession

`cortexj-lab-api` wraps the JDK's JShell in a managed, closable session that
inherits your process classpath — so `CortexJ` snippets work directly:

```java
try (LabSession lab = LabSession.create()) {
    SnippetResult result = lab.eval("io.cortexj.CortexJ.builder().build()");
    System.out.println(result.output());

    // the environment is captured at call time, reproducibly
    EnvironmentDescriptor env = lab.describeEnvironment();
    System.out.println(env.jdkVersion() + " on " + env.os());
}
```

- **Typed errors** — snippet failures come back typed, not as stack-trace soup.
- **Rich display contract** — `Displayable.table` renders headers and rows
    (chart/image renderers are tracked follow-ups, honestly not in 0.x).
- **Notebook → Maven export** — `lab.exportProjectSkeleton(path, projectId)`
    writes a starter project from your session work.

## Jupyter kernel

`cortexj-notebook` implements the Jupyter wire protocol (ZeroMQ, HMAC-signed
envelopes) as a Java kernel, with a live in-process contract test. Point Jupyter
at the kernel and evaluate JVM code — the session inherits the process
classpath, so Runtime model loads work interactively.

## Requirements

- Java 17+ (ADR-017; the Lab is a modern-JVM track, unlike the Java 8 Runtime).
- No Python anywhere in the loop.

## Next steps

- [Experiments](experiments.md) — capture what you did interactively as a
    reproducible run.
- [Data](data.md) — load and fingerprint datasets inside the session.
