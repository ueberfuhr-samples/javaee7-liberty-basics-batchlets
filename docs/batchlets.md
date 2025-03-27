# Batchlets

1. [Was sind Batchlets?](#was-sind-batchlets)
2. [Wie erstellt man ein Batchlet?](#wie-erstellt-man-ein-batchlet)
3. [Wie werden Batchlets ausgeführt?](#wie-werden-batchlets-ausgeführt)

## Was sind Batchlets?

Batchlets sind ein Bestandteil der Java Batch API (JSR 352) — das ist die Java-Standard-API für Batch-Verarbeitung.
Sie sind die einfachste Form eines Batch Steps - eine Batch-Komponente, die eine einzelne Aufgabe ausführt – also etwas, das keinen Chunk-orientierten Input/Output braucht.

Kurz gesagt:
- Ein ist eine _fire-and-forget_-Aufgabe.
- Es hat keine Items, keine Reader/Processor/Writer – es ist einfach ein einmaliges Stück Logik.

## Wie erstellt man ein Batchlet?

Die Implementierung erfolgt ähnlich zu Servlets:

```java
@Named("MySimpleBatchlet")
@Dependent
public class MySimpleBatchlet extends AbstractBatchlet {

  // Kontextinformationen
  @Inject
  private JobContext jobContext;

  @Inject
  @BatchProperty(name = "message")
  private String message;

  @Override
  public String process() throws Exception {
    System.out.printf(
      "Batchlet is running (Execution ID: %d)! Message: %s%n",
      jobContext.getExecutionId(),
      message
    );
    return "COMPLETED";
  }

}
```

Batch-Properties werden in der `job.xml` (`/META-INF/batch-jobs`) definiert:

```xml
<job id="simpleJob">
    <step id="step1">
        <batchlet ref="MySimpleBatchlet">
            <properties>
                <property name="message" value="Hello from Batchlet!" />
            </properties>
        </batchlet>
    </step>
</job>
```

## Wie werden Batchlets ausgeführt?

Batchlets werden im Rahmen eines Jobs ausgeführt, in dem sie als Step eingebettet sind. Insofern starten wir den Batch-Job wie folgt:

```java
  public long executeJob() { // returns execution id
    return BatchRuntime
        .getJobOperator()
        .start("simpleJob", new Properties());
  }
```
