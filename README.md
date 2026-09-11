# public-transfer

#1:

Ja. Beim **Maven Release Plugin** gibt es einige typische Ursachen, warum bei `release:prepare` bzw. einem Versionswechsel Module nicht korrekt erkannt werden und deren Version in den `pom.xml` Dateien nicht angepasst wird.

Besonders häufig sind diese Ursachen:

* **Module sind nicht Teil des Maven-Reaktors.** Wenn ein Modul zwar im Repository liegt, aber nicht über `<modules>` vom Parent/Aggregator eingebunden ist, betrachtet das Release Plugin es nicht als Teil des Builds. Beispiel:

  ```xml
  <modules>
      <module>module-a</module>
      <module>module-b</module>
  </modules>
  ```

  Fehlt `module-c`, wird dessen `pom.xml` normalerweise auch nicht aktualisiert.

* **Parent und Aggregator werden verwechselt.** Ein Maven-Projekt kann einen Parent haben, ohne dass dieser Parent das Modul über `<modules>` aggregiert. Das Release Plugin arbeitet im Wesentlichen mit dem Maven-Reaktor. Eine Parent-Beziehung allein bedeutet also nicht automatisch, dass Maven das Modul beim Release mitverarbeitet.

* **Falscher Einstiegspunkt für `release:prepare`.** Wenn du beispielsweise innerhalb eines Submoduls startest:

  ```bash
  cd module-a
  mvn release:prepare
  ```

  sieht Maven möglicherweise nur dieses Modul. Bei einem Multi-Module-Projekt solltest du den Release normalerweise auf dem obersten Aggregator-POM starten:

  ```bash
  mvn release:prepare
  ```

* **Versionsnummer wird über Properties definiert.** Ein sehr häufiger Sonderfall ist etwa:

  ```xml
  <version>${revision}</version>

  <properties>
      <revision>1.2.3-SNAPSHOT</revision>
  </properties>
  ```

  Solche sogenannten CI-friendly Versions (`revision`, `sha1`, `changelist`) werden vom Maven Release Plugin je nach Plugin-/Maven-Version und Projektaufbau nicht genauso behandelt wie eine direkt eingetragene `<version>1.2.3-SNAPSHOT</version>`. Auch eigene Properties wie:

  ```xml
  <properties>
      <project.version>1.2.3-SNAPSHOT</project.version>
  </properties>
  ```

  können dazu führen, dass das Release Plugin die erwartete Stelle nicht verändert.

* **Module referenzieren den Parent mit einer Property-Version.** Problematisch können Konstrukte wie dieses sein:

  ```xml
  <parent>
      <groupId>com.example</groupId>
      <artifactId>parent</artifactId>
      <version>${revision}</version>
  </parent>
  ```

  Maven hat bei Parent-Versionen historisch strengere Regeln als bei normalen Dependencies. Je nach Maven-Version und Aufbau kann das Release Plugin solche Beziehungen nicht sauber auflösen oder umschreiben.

* **`relativePath` zeigt auf den falschen Parent.** Beispiel:

  ```xml
  <parent>
      <groupId>com.example</groupId>
      <artifactId>parent</artifactId>
      <version>1.0-SNAPSHOT</version>
      <relativePath>../pom.xml</relativePath>
  </parent>
  ```

  Wenn `../pom.xml` nicht der erwartete Parent ist oder ein Zwischen-Parent existiert, kann Maven ein anderes Modell auflösen als du erwartest. Besonders bei verschachtelten Multi-Module-Projekten führt das zu schwer nachvollziehbaren Release-Effekten.

* **`groupId`, `artifactId` oder Version stimmen zwischen Parent und Modul nicht exakt überein.** Beispiel Parent:

  ```xml
  <groupId>com.company</groupId>
  <artifactId>my-parent</artifactId>
  <version>1.5.0-SNAPSHOT</version>
  ```

  aber Modul:

  ```xml
  <parent>
      <groupId>com.company</groupId>
      <artifactId>my-parent</artifactId>
      <version>1.4.0-SNAPSHOT</version>
  </parent>
  ```

  Dann betrachtet Maven das unter Umständen nicht als dieselbe Reaktorbeziehung, die du erwartest.

* **Das Modul verwendet eine explizite eigene Version.** Zum Beispiel:

  ```xml
  <parent>
      <groupId>com.example</groupId>
      <artifactId>parent</artifactId>
      <version>1.0.0-SNAPSHOT</version>
  </parent>

  <artifactId>module-a</artifactId>
  <version>4.2.0-SNAPSHOT</version>
  ```

  Das Modul hat damit einen eigenen Release-Zyklus. Das Release Plugin kann diese Version separat behandeln. Je nach gewählten Parametern entsteht dann leicht der Eindruck, dass die Version "nicht mit hochgesetzt" wurde.

* **Unterschiedliche Versionsstände innerhalb des Reaktors.** Zum Beispiel:

  ```text
  parent        1.5.0-SNAPSHOT
  module-a      1.5.0-SNAPSHOT
  module-b      1.4.2-SNAPSHOT
  ```

  Gerade wenn `autoVersionSubmodules` nicht aktiviert ist, kann Maven nach Versionen fragen oder Module unterschiedlich behandeln.

Ein sehr wichtiger Parameter ist deshalb:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-release-plugin</artifactId>
    <configuration>
        <autoVersionSubmodules>true</autoVersionSubmodules>
    </configuration>
</plugin>
```

Damit übernimmt das Release Plugin für Submodule automatisch dieselbe Versionsstrategie. Auf der Kommandozeile geht entsprechend:

```bash
mvn release:prepare \
  -DautoVersionSubmodules=true
```

Auch **Profile** können die Ursache sein. Wenn Module nur innerhalb eines Profils definiert sind:

```xml
<profiles>
    <profile>
        <id>special-modules</id>
        <modules>
            <module>module-c</module>
        </modules>
    </profile>
</profiles>
```

und du den Release ohne das Profil startest:

```bash
mvn release:prepare
```

existiert `module-c` aus Sicht des Maven-Reaktors gar nicht. Dann müsste beispielsweise gelten:

```bash
mvn -Pspecial-modules release:prepare
```

Ein weiterer häufiger Fall sind **interne Dependencies zwischen Modulen**. Beispiel:

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>module-a</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

Normalerweise erkennt das Release Plugin solche Reaktor-Abhängigkeiten und ersetzt sie durch die Release-Version. Werden Versionen aber über Properties angegeben:

```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>module-a</artifactId>
    <version>${module-a.version}</version>
</dependency>

<properties>
    <module-a.version>1.0.0-SNAPSHOT</module-a.version>
</properties>
```

kann die Aktualisierung komplizierter werden. Das Plugin muss dann erkennen, dass diese Property ausschließlich bzw. tatsächlich diese Reaktor-Abhängigkeit repräsentiert.

Sehr hilfreich zur Diagnose ist zunächst:

```bash
mvn help:effective-pom
```

und:

```bash
mvn validate
```

Außerdem würde ich einmal prüfen, welche Projekte Maven tatsächlich in den Reaktor nimmt:

```bash
mvn clean install -DskipTests
```

Am Anfang der Ausgabe steht normalerweise etwas wie:

```text
Reactor Build Order:

my-parent
module-a
module-b
module-c
```

Wenn das problematische Modul **dort bereits fehlt**, liegt das Problem fast sicher nicht primär am Maven Release Plugin, sondern am Maven-Projektaufbau, `<modules>`, Profilen oder dem Ort, von dem der Maven-Aufruf gestartet wird.

Wenn das Modul hingegen **im Reactor Build Order auftaucht, aber `release:prepare` seine Version trotzdem nicht ändert**, würde ich als Nächstes speziell diese Punkte untersuchen:

1. `<version>` des Moduls und seines Parents
2. Verwendung von `${revision}`, `${changelist}` oder eigenen Version-Properties
3. `autoVersionSubmodules`
4. `relativePath`
5. interne Dependency-Versionen
6. aktive Maven-Profile
7. verwendete Version von `maven-release-plugin`
8. Inhalt von `release.properties`
9. vorher fehlgeschlagene `release:prepare`-Runs

Gerade Punkt 8 und 9 werden oft übersehen. Ein vorheriger Release-Versuch kann Dateien wie

```text
release.properties
pom.xml.releaseBackup
```

hinterlassen. Dann sollte man vor einem neuen Versuch häufig:

```bash
mvn release:clean
```

ausführen.

Wenn du mir deinen **Parent-POM plus 1–2 betroffene Modul-POMs** und den verwendeten `mvn release:prepare ...` Aufruf zeigst, kann ich ziemlich gezielt feststellen, warum genau diese Module beim Versionswechsel ausgelassen werden.









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
