# public-transfer

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
