## 2026-09-21

#2: 2026-09-22 14:25

Ja, Andreas. Mit diesen URLs wird die Ursache ziemlich klar: **`https://kaiser.dev.drv` ist sehr wahrscheinlich eure JFrog-Platform-URL**, und genau diese URL sollte für `--url` verwendet werden. JFrog beschreibt `--url` als Basis-URL der JFrog Platform; Artifactory und Xray hängen typischerweise darunter. ([JFrog Docs][1])

Bei euch wäre das voraussichtlich:

```text
JFrog Platform:
https://kaiser.dev.drv

Artifactory:
https://kaiser.dev.drv/artifactory

Xray:
https://kaiser.dev.drv/xray

Proxy für releases.jfrog.io:
https://kaiser.dev.drv/artifactory/jfrog-generic-releases-remote
```

Der Xray-Version-Endpunkt ist laut aktueller JFrog-Dokumentation tatsächlich:

```text
https://kaiser.dev.drv/xray/api/v1/system/version
```

([JFrog Docs][2])

Genau diesen Request sollte die JFrog CLI erfolgreich ausführen können.

Der Fehler entsteht wahrscheinlich durch eure jetzige Kombination:

```bash
jf audit \
  --mvn \
  --server-id=xxx \
  --url=https://kaiser.dev.drv/artifactory/jfrog-generic-releases-remote \
  ...
```

Damit sagt ihr der CLI gleichzeitig:

> „Benutze Server `xxx`, aber die JFrog-Platform-URL ist `.../artifactory/jfrog-generic-releases-remote`.“

Daraufhin versucht die CLI ihren Xray-Endpunkt relativ dazu zu bestimmen. Deshalb landet sie sinngemäß bei etwas wie:

```text
/artifactory/jfrog-generic-releases-remote/.../xray/api/v1/system/version
```

Das erklärt euren 404 sehr gut.

## So würde ich es konfigurieren

Zunächst die Server-Konfiguration:

```bash
jf c add xxx \
  --url=https://kaiser.dev.drv \
  --user="$JFROG_USER" \
  --password="$JFROG_PASSWORD" \
  --interactive=false \
  --overwrite
```

JFrog dokumentiert genau dieses Schema: Die `--url` ist die **JFrog Platform URL**, nicht eine Repository-URL. ([JFrog Docs][1])

Ich würde bei euch sogar explizit alle relevanten URLs setzen. Das beseitigt jede Mehrdeutigkeit:

```bash
jf c add xxx \
  --url=https://kaiser.dev.drv \
  --artifactory-url=https://kaiser.dev.drv/artifactory \
  --xray-url=https://kaiser.dev.drv/xray \
  --user="$JFROG_USER" \
  --password="$JFROG_PASSWORD" \
  --interactive=false \
  --overwrite
```

`--artifactory-url` und `--xray-url` sind offizielle Optionen der CLI-Konfiguration. ([JFrog Docs][1])

Danach:

```bash
jf c show xxx
```

Dabei sollte sinngemäß herauskommen:

```text
Server ID:         xxx
JFrog Platform URL: https://kaiser.dev.drv/
Artifactory URL:    https://kaiser.dev.drv/artifactory/
Xray URL:           https://kaiser.dev.drv/xray/
```

### Und euer Proxy-Repository?

Das ist der zweite, separate Teil.

Euer

```text
https://kaiser.dev.drv/artifactory/jfrog-generic-releases-remote
```

wird **nicht als `--url` übergeben**.

Stattdessen:

```bash
export JFROG_CLI_RELEASES_REPO="xxx/jfrog-generic-releases-remote"
```

Das ist exakt der von JFrog vorgesehene Mechanismus. Die Dokumentation sagt ausdrücklich, dass `JFROG_CLI_RELEASES_REPO` das Repository angibt, aus dem die von Maven/Gradle benötigten JARs geladen werden. Das Format ist:

```text
<server-id>/<repository-name>
```

und das Repository soll `https://releases.jfrog.io` proxyen. Außerdem gilt diese Variable ausdrücklich auch für `jf audit` bei Maven-/Gradle-Projekten. ([JFrog Docs][3])

Dann wird euer Aufruf:

```bash
export JFROG_CLI_RELEASES_REPO="xxx/jfrog-generic-releases-remote"

jf audit \
  --mvn \
  --server-id=xxx \
  --fail=false \
  --format=json
```

**Kein `--url`, `--user` und `--password` mehr beim `jf audit`**, weil das bereits in `xxx` steckt.

## Ich würde als Erstes diesen Test machen

Direkt vom Jenkins-Agenten:

```bash
curl -vk \
  -u "$JFROG_USER:$JFROG_PASSWORD" \
  https://kaiser.dev.drv/xray/api/v1/system/version
```

Wenn alles korrekt geroutet ist, sollte hier **kein 404** kommen, sondern eine JSON-Antwort mit Xray-Versionsinformationen. Der Endpoint ist offiziell genau so aufgebaut. ([JFrog Docs][2])

Wenn beispielsweise sowas zurückkommt:

```json
{
  "xray_version": "...",
  "xray_revision": "..."
}
```

wissen wir:

```text
Jenkins
   |
   v
https://kaiser.dev.drv
   |
   +------ /artifactory ------> Artifactory
   |
   +------ /xray -------------> Xray
```

Dann ist die Sache praktisch geklärt.

Falls dagegen

```bash
curl https://kaiser.dev.drv/xray/api/v1/system/version
```

ebenfalls einen `404` liefert, liegt **noch ein zweites Problem im Reverse Proxy / Load Balancer vor**: `/xray` wird dann vermutlich nicht an den JFrog Router bzw. Xray weitergeleitet. Das wäre dann unabhängig vom Remote Repository.

### Für eure Jenkins-Pipeline wäre die Zielkonfiguration also ungefähr

```groovy
withCredentials([
    usernamePassword(
        credentialsId: 'jfrog-credentials',
        usernameVariable: 'JFROG_USER',
        passwordVariable: 'JFROG_PASSWORD'
    )
]) {
    sh '''
        jf c add xxx \
          --url="https://kaiser.dev.drv" \
          --artifactory-url="https://kaiser.dev.drv/artifactory" \
          --xray-url="https://kaiser.dev.drv/xray" \
          --user="$JFROG_USER" \
          --password="$JFROG_PASSWORD" \
          --interactive=false \
          --overwrite

        export JFROG_CLI_RELEASES_REPO="xxx/jfrog-generic-releases-remote"

        jf audit \
          --mvn \
          --server-id=xxx \
          --fail=false \
          --format=json
    '''
}
```

Damit sind die beiden Dinge sauber getrennt:

```text
                   JFrog Platform
                https://kaiser.dev.drv
                         |
            +------------+-------------+
            |                          |
            v                          v
      /artifactory                   /xray
            |                          |
            v                          v
      Artifactory                    Xray
            |
            |
jfrog-generic-releases-remote
            |
            v
   https://releases.jfrog.io
```

**Ich halte insbesondere das gleichzeitige `--server-id=xxx` und `--url=<Remote-Repo>` in eurem bisherigen `jf audit` für den Kernfehler.** Das Remote Repository gehört in `JFROG_CLI_RELEASES_REPO`; als Platform-URL gehört `https://kaiser.dev.drv` hinein.

Wenn Du mir als Nächstes die Ausgabe von

```bash
jf c show xxx
```

und von

```bash
curl -vk https://kaiser.dev.drv/xray/api/v1/system/version
```

zeigst (Credentials/Tokens natürlich entfernen), kann man sehr schnell erkennen, ob danach noch ein Reverse-Proxy-/Xray-Routingproblem übrig bleibt.

[1]: https://docs.jfrog.com/integrations/docs/jf-config-add?utm_source=chatgpt.com "Add a JFrog CLI Server Configuration"
[2]: https://docs.jfrog.com/security/reference/get-version?utm_source=chatgpt.com "Get Version"
[3]: https://docs.jfrog.com/integrations/docs/jfrog-cli-environment-variables?utm_source=chatgpt.com "JFrog CLI Environment Variables"




-----------------------------------------------------



# public-transfer


```
stage('Gate on Critical') {
  steps {
    sh '''
      set -euo pipefail
      python3 - << 'PY'
import json, sys
data = json.load(open('jfrog-audit.json','r',encoding='utf-8'))
vulns = data.get('vulnerabilities') or data.get('vulns') or []
crit = [v for v in vulns if str(v.get('severity','')).lower() == 'critical']
print(f"Critical findings: {len(crit)}")
if crit:
    # optional: kurz ausgeben
    for v in crit[:10]:
        print(v.get('cve') or v.get('id'), v.get('summary','')[:120])
    sys.exit(1)
PY
    '''
  }
}
```



--------


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
