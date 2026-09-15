# public-transfer

#1: 2026-09-15T09:37:00+02:00

Ja, Andreas. Für ein Maven-Multi-Module-Projekt, bei dem **alle Module gemeinsam dieselbe Version haben**, würde ich die Version ausschließlich über die Parent-POM steuern. Die Submodule erben ihre eigene Projektversion vom Parent; lediglich die `<parent><version>` muss in jedem Submodul explizit stehen.

Das passt sehr gut zum `maven-release-plugin`. Aktuell ist Version **3.3.1** des Plugins dokumentiert. Für Multi-Module-Projekte gibt es `autoVersionSubmodules=true`, wodurch Release- und nächste Development-Version für alle Module gemeinsam gesetzt werden. ([Apache Maven][1])

### Projektstruktur

```text
my-project/
├── pom.xml
├── module-api/
│   └── pom.xml
├── module-core/
│   └── pom.xml
└── module-app/
    └── pom.xml
```

Angenommen, der Entwicklungsstand ist:

```text
1.2.0-SNAPSHOT
```

Dann enthält die **Root-/Parent-POM** die eigentliche Projektversion:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
             http://maven.apache.org/POM/4.0.0
             https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>de.example</groupId>
    <artifactId>my-project</artifactId>
    <version>1.2.0-SNAPSHOT</version>

    <packaging>pom</packaging>

    <modules>
        <module>module-api</module>
        <module>module-core</module>
        <module>module-app</module>
    </modules>

    <properties>
        <maven.compiler.release>21</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <scm>
        <connection>
            scm:git:https://github.com/example/my-project.git
        </connection>
        <developerConnection>
            scm:git:https://github.com/example/my-project.git
        </developerConnection>
        <url>
            https://github.com/example/my-project
        </url>
        <tag>HEAD</tag>
    </scm>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-release-plugin</artifactId>
                <version>3.3.1</version>

                <configuration>
                    <autoVersionSubmodules>true</autoVersionSubmodules>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

Eine SCM-Konfiguration mit `developerConnection` sowie eine festgelegte Version des Release-Plugins gehören zur empfohlenen Grundkonfiguration. ([Apache Maven][2])

## Die Submodule

Der entscheidende Punkt ist jetzt, dass die Submodule **keine eigene `<version>`** bekommen.

`module-api/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>de.example</groupId>
        <artifactId>my-project</artifactId>
        <version>1.2.0-SNAPSHOT</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <artifactId>module-api</artifactId>

</project>
```

`module-core/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>de.example</groupId>
        <artifactId>my-project</artifactId>
        <version>1.2.0-SNAPSHOT</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <artifactId>module-core</artifactId>

    <dependencies>
        <dependency>
            <groupId>de.example</groupId>
            <artifactId>module-api</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>

</project>
```

Und `module-app/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>de.example</groupId>
        <artifactId>my-project</artifactId>
        <version>1.2.0-SNAPSHOT</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <artifactId>module-app</artifactId>

    <dependencies>
        <dependency>
            <groupId>de.example</groupId>
            <artifactId>module-core</artifactId>
            <version>${project.version}</version>
        </dependency>

        <dependency>
            <groupId>de.example</groupId>
            <artifactId>module-api</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>

</project>
```

Damit ergibt sich effektiv:

```text
de.example:my-project:1.2.0-SNAPSHOT
de.example:module-api:1.2.0-SNAPSHOT
de.example:module-core:1.2.0-SNAPSHOT
de.example:module-app:1.2.0-SNAPSHOT
```

Obwohl bei den drei Modulen **keine eigene Projektversion** definiert ist.

Der Parent ist gleichzeitig Parent und Aggregator. Maven unterscheidet diese Konzepte grundsätzlich, aber es ist völlig üblich, dass dieselbe POM beide Rollen übernimmt. ([Apache Maven][3])

### Warum steht trotzdem `1.2.0-SNAPSHOT` dreimal in den Child-POMs?

Das ist eine Eigenheit des Maven-POM-Modells:

```xml
<parent>
    ...
    <version>1.2.0-SNAPSHOT</version>
</parent>
```

muss eine Version enthalten.

Das hier geht also nicht:

```xml
<parent>
    <groupId>de.example</groupId>
    <artifactId>my-project</artifactId>
</parent>
```

Die gute Nachricht ist: Genau diese Referenzen aktualisiert das Release Plugin für Dich.

Vor dem Release:

```xml
<parent>
    ...
    <version>1.2.0-SNAPSHOT</version>
</parent>
```

Beim Release wird daraus temporär:

```xml
<parent>
    ...
    <version>1.2.0</version>
</parent>
```

und nach dem Release beispielsweise:

```xml
<parent>
    ...
    <version>1.2.1-SNAPSHOT</version>
</parent>
```

Du musst diese Stellen also nicht manuell ändern.

---

## Ein Release durchführen

Aus dem Root-Verzeichnis:

```bash
mvn release:clean release:prepare
```

Mit

```xml
<autoVersionSubmodules>true</autoVersionSubmodules>
```

fragt Maven nicht für jedes Modul einzeln nach einer Version, sondern verwendet dieselbe Version für das gesamte Projekt. Genau dafür ist diese Option vorgesehen. ([Apache Maven][1])

Bei

```text
1.2.0-SNAPSHOT
```

wirst Du sinngemäß gefragt:

```text
What is the release version?
1.2.0

What is SCM release tag?
my-project-1.2.0

What is the new development version?
1.2.1-SNAPSHOT
```

`release:prepare` macht dabei im Wesentlichen:

```text
1. Prüfen, ob Git Working Tree sauber ist

2. POMs:
   1.2.0-SNAPSHOT
       ↓
   1.2.0

3. Release-POMs committen

4. Git Tag erzeugen:
   my-project-1.2.0

5. POMs:
   1.2.0
       ↓
   1.2.1-SNAPSHOT

6. Development-POMs committen
```

Dieser grundsätzliche Prepare-Ablauf – POM-Version ändern, committen, SCM-Tag erzeugen und anschließend die nächste Development-Version setzen – ist genau das Modell des Release Plugins. ([Apache Maven][4])

Danach:

```bash
mvn release:perform
```

Dabei wird der erzeugte Tag ausgecheckt und typischerweise

```bash
mvn deploy
```

darauf ausgeführt.

Für `release:perform` brauchst Du deshalb normalerweise noch ein `distributionManagement`, beispielsweise für Nexus oder Artifactory:

```xml
<distributionManagement>

    <repository>
        <id>releases</id>
        <url>
            https://nexus.example.de/repository/maven-releases/
        </url>
    </repository>

    <snapshotRepository>
        <id>snapshots</id>
        <url>
            https://nexus.example.de/repository/maven-snapshots/
        </url>
    </snapshotRepository>

</distributionManagement>
```

---

## Für CI/CD würde ich es nicht interaktiv machen

Beispielsweise:

```bash
mvn --batch-mode release:clean release:prepare \
    -DreleaseVersion=1.2.0 \
    -DdevelopmentVersion=1.2.1-SNAPSHOT \
    -Dtag=my-project-1.2.0
```

danach:

```bash
mvn --batch-mode release:perform
```

Das ist wesentlich geeigneter für Jenkins, GitLab CI, GitHub Actions etc.

---

## Ein Detail würde ich noch verbessern: interne Dependencies

Anstatt überall

```xml
<version>${project.version}</version>
```

einzutragen, würde ich bei größeren Projekten die internen Modulversionen zentral im Parent unter `dependencyManagement` verwalten:

```xml
<dependencyManagement>
    <dependencies>

        <dependency>
            <groupId>de.example</groupId>
            <artifactId>module-api</artifactId>
            <version>${project.version}</version>
        </dependency>

        <dependency>
            <groupId>de.example</groupId>
            <artifactId>module-core</artifactId>
            <version>${project.version}</version>
        </dependency>

    </dependencies>
</dependencyManagement>
```

Dann kann `module-app` einfach schreiben:

```xml
<dependencies>

    <dependency>
        <groupId>de.example</groupId>
        <artifactId>module-api</artifactId>
    </dependency>

    <dependency>
        <groupId>de.example</groupId>
        <artifactId>module-core</artifactId>
    </dependency>

</dependencies>
```

Das ergibt meiner Ansicht nach die sauberste Struktur:

```text
Parent:
    version = 1.2.0-SNAPSHOT

module-api:
    Parent = 1.2.0-SNAPSHOT
    eigene Version = keine

module-core:
    Parent = 1.2.0-SNAPSHOT
    eigene Version = keine

module-app:
    Parent = 1.2.0-SNAPSHOT
    eigene Version = keine
```

Beim Release:

```text
                 development       release        danach

Parent           1.2.0-SNAPSHOT → 1.2.0       → 1.2.1-SNAPSHOT
module-api       1.2.0-SNAPSHOT → 1.2.0       → 1.2.1-SNAPSHOT
module-core      1.2.0-SNAPSHOT → 1.2.0       → 1.2.1-SNAPSHOT
module-app       1.2.0-SNAPSHOT → 1.2.0       → 1.2.1-SNAPSHOT
                                      │
                                      └── Git Tag
                                          my-project-1.2.0
```

**Wichtig:** Ich würde hier zunächst bewusst **nicht mit `${revision}` / CI-friendly versions** wie

```xml
<version>${revision}</version>
```

arbeiten. Das ist ein anderes Versionsmodell und bringt gerade in Kombination mit Release- und Publishing-Workflows zusätzliche Überlegungen hinein. Für einen klassischen Maven-Release-Plugin-Workflow ist eine normale `1.2.0-SNAPSHOT`-Version im Parent sehr transparent und robust.

Ein guter nächster Schritt wäre ein **komplettes, tatsächlich ausführbares Beispielprojekt inklusive Java-Klassen, Nexus/Artifactory-Konfiguration und GitLab-CI/Jenkins-Release-Job**. Dann kann ich Dir auch zeigen, was `release:prepare` konkret an den vier POM-Dateien verändert.

[1]: https://maven.apache.org/maven-release/maven-release-plugin/usage/prepare-release.html?utm_source=chatgpt.com "Prepare a Release – Maven Release plugin"
[2]: https://maven.apache.org/maven-release/maven-release-plugin/usage.html?utm_source=chatgpt.com "Usage – Maven Release plugin"
[3]: https://maven.apache.org/pom.html?utm_source=chatgpt.com "POM Reference – Maven"
[4]: https://maven.apache.org/guides/mini/guide-releasing.html?utm_source=chatgpt.com "Releasing – Maven"








-----

setzen des Parameters in JobDSL
```
job('example-job') {
    parameters {
        stringParam('ENVIRONMENT', 'dev', 'Target environment')
        stringParam('VERSION', '', 'Version to deploy')
    }

    steps {
        scriptlerScript('my-script.groovy') {
            propagateParams(true)
        }
    }
}
```

```
job('example-job') {

    parameters {
        stringParam('ENVIRONMENT', 'dev', 'Target environment')
        stringParam('VERSION', '', 'Version to deploy')
    }

    steps {
        configure { project ->

            def builders = project / builders

            builders.appendNode(
                'org.jenkinsci.plugins.scriptler.builder.ScriptlerBuilder'
            ).with {
                appendNode('builderId', 'scriptler')
                appendNode('scriptId', 'my-script.groovy')
                appendNode('propagateParams', 'true')
                appendNode('parameters')
            }
        }
    }
}
```

```
Conditional steps (multiple)
│
├── Run?
│   └── And
│       │
│       ├── Regular expression match
│       │   Expression: ^true$
│       │   Label: ${vm_klone}
│       │
│       └── Regular expression match
│           Expression: .*I_Dialog.*
│           Label: ${vm_snapshot}
│
└── Build Steps
    └── Execute shell
        └── ./starte_server_1

Conditional steps (multiple)
│
├── Run?
│   └── And
│       │
│       ├── Regular expression match
│       │   Expression: ^true$
│       │   Label: ${vm_klone}
│       │
│       └── Regular expression match
│           Expression: .*C_Dialog.*
│           Label: ${vm_snapshot}
│
└── Build Steps
    └── Execute shell
        └── ./starte_server_2
```

```
steps {

    // I_dialog -> lokales Shell-Skript
    conditionalSteps {
        condition {
            and(
                {
                    booleanCondition('${vm_klone}')
                },
                {
                    expression('.*I_dialog.*', '${vm_snapshot}')
                }
            )
        }

        runner('DontRun')

        steps {
            shell('./starte_server_1')
        }
    }


    // C_dialog -> Downstream-Job
    conditionalSteps {
        condition {
            and(
                {
                    booleanCondition('${vm_klone}')
                },
                {
                    expression('.*C_dialog.*', '${vm_snapshot}')
                }
            )
        }

        runner('DontRun')

        steps {
            downstreamParameterized {
                trigger('starte_server_2') {
                    parameters {
                        currentBuild()
                    }

                    block {
                        buildStepFailure('FAILURE')
                        failure('FAILURE')
                        unstable('UNSTABLE')
                    }
                }
            }
        }
    }
}
```
