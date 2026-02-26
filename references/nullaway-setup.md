# NullAway Build Configuration

Configure NullAway as an Error Prone plugin to enforce null-safety at compile time. The build must fail on any violation.

**Maven** -- add Error Prone + NullAway to the compiler plugin:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <compilerArgs>
            <arg>-XDcompilePolicy=simple</arg>
            <arg>-Xplugin:ErrorProne -XepOpt:NullAway:AnnotatedPackages=com.example -XepOpt:NullAway:JSpecifyMode=true</arg>
        </compilerArgs>
        <annotationProcessorPaths>
            <path>
                <groupId>com.google.errorprone</groupId>
                <artifactId>error_prone_core</artifactId>
                <version>2.36.0</version>
            </path>
            <path>
                <groupId>com.uber.nullaway</groupId>
                <artifactId>nullaway</artifactId>
                <version>0.12.6</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

Also set the Java compiler to use Error Prone by adding to `<properties>`:

```xml
<maven.compiler.source>21</maven.compiler.source>
<maven.compiler.target>21</maven.compiler.target>
```

And configure the `maven-compiler-plugin` to fork with Error Prone's `javac`:

```xml
<fork>true</fork>
<compilerArgs>
    <arg>-J--add-exports=jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED</arg>
    <arg>-J--add-exports=jdk.compiler/com.sun.tools.javac.file=ALL-UNNAMED</arg>
    <arg>-J--add-exports=jdk.compiler/com.sun.tools.javac.main=ALL-UNNAMED</arg>
    <arg>-J--add-exports=jdk.compiler/com.sun.tools.javac.model=ALL-UNNAMED</arg>
    <arg>-J--add-exports=jdk.compiler/com.sun.tools.javac.parser=ALL-UNNAMED</arg>
    <arg>-J--add-exports=jdk.compiler/com.sun.tools.javac.processing=ALL-UNNAMED</arg>
    <arg>-J--add-exports=jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED</arg>
    <arg>-J--add-exports=jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED</arg>
    <arg>-J--add-opens=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED</arg>
    <arg>-J--add-opens=jdk.compiler/com.sun.tools.javac.comp=ALL-UNNAMED</arg>
</compilerArgs>
```

**Gradle** -- apply the Error Prone and NullAway plugins:

```groovy
plugins {
    id 'net.ltgt.errorprone' version '4.1.0'
}

dependencies {
    errorprone 'com.google.errorprone:error_prone_core:2.36.0'
    errorprone 'com.uber.nullaway:nullaway:0.12.6'
}

tasks.withType(JavaCompile).configureEach {
    options.errorprone {
        error('NullAway')
        option('NullAway:AnnotatedPackages', 'com.example')
        option('NullAway:JSpecifyMode', 'true')
    }
}
```

**Gradle Kotlin DSL:**

```kotlin
plugins {
    id("net.ltgt.errorprone") version "4.1.0"
}

dependencies {
    errorprone("com.google.errorprone:error_prone_core:2.36.0")
    errorprone("com.uber.nullaway:nullaway:0.12.6")
}

tasks.withType<JavaCompile>().configureEach {
    options.errorprone {
        error("NullAway")
        option("NullAway:AnnotatedPackages", "com.example")
        option("NullAway:JSpecifyMode", "true")
    }
}
```

## Key Settings

| Option | Purpose |
|--------|---------|
| `AnnotatedPackages` | **Required.** Set to your project's root package (e.g., `com.example`). NullAway only checks these packages. |
| `JSpecifyMode=true` | **Required.** Enables JSpecify annotation support (`@NullMarked`, `@Nullable`). |
| `ExcludedFieldAnnotations` | Optional. Exclude fields annotated with specific annotations (e.g., `@Mock`) from null checks. |
| `ExternalInitAnnotations` | Optional. Annotations indicating fields are initialized externally (e.g., `@Autowired`). |

For Spring Boot projects, you may need to exclude framework-managed initialization:

```
-XepOpt:NullAway:ExternalInitAnnotations=org.springframework.beans.factory.annotation.Autowired
-XepOpt:NullAway:ExcludedFieldAnnotations=org.mockito.Mock
```

## Important

- Set `AnnotatedPackages` to the actual root package of the project, not `com.example`.
- NullAway runs as a compile-time check -- null violations are **build errors**, not warnings. The build will fail if any violation exists.
- When the build fails due to NullAway, fix the null-safety issue in the source code. Do not suppress warnings unless dealing with a confirmed false positive, and document the reason.

## IDE & Tooling Support

- IntelliJ IDEA 2025.3+ and Eclipse with Spring Tools provide compile-time warnings for JSpecify violations.
- SonarQube detects additional null-safety issues -- the SonarQube analysis step in the workflow will catch violations NullAway may not cover.
