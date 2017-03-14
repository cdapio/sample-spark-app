
# Introduction

This section will show how you can use your scala Spark application and run it in CDAP. 
The procedure is the same when compared to what was done for Java. 

# Getting Started

- Package

```
  cd sparkpi-scala
  mvn clean package
```

- Deploy `sparkpi-1.0.0.jar` in CDAP as a plugin.

```
  curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/artifacts/sparkpi-program" \
    -H 'Artifact-Plugins: [ { "name": "SparkPi", "type": "sparkprogram", "className": "com.example.spark.SparkPi" }]' \
    -H "Artifact-Version: 1.0.0" \
    -H "Artifact-Extends: system:cdap-notifiable-workflow[1.0.0, 1.0.0]" \
    --data-binary @/PATH/TO/sparkpi-1.0.0.jar
```

- Create CDAP Application with the plugin just deployed.

```
  curl -v "localhost:11015/v3/namespaces/default/apps/SparkPiApp" -X PUT -d @app.json
```

where `app.json` contains the configurations which are used to configure the applications.

```
  {
    "artifact": {
      "name": "cdap-notifiable-workflow",
      "version": "1.0.0",
      "scope": "system"
    },
    "config": {
      "plugin": {
      "name": "SparkPi",
      "type": "sparkprogram",
      "artifact": {
        "name": "sparkpi-program",
        "scope": "user",
        "version": "1.0.0"
      }
    },

      "notificationEmailSender": "sender@example.domain.com",
      "notificationEmailIds": ["recipient@example.domain.com"],
      "notificationEmailSubject": "[Critical] Workflow execution failed.",
      "notificationEmailBody": "Execution of Workflow running the SparkPi program failed."
    }
  }
```

At this point `SparkPiApp` application should be created in CDAP.

- Run the `NotifiableWorkflow` program in the `SparkPiApp`. `main` method of the `SparkPi` program
expects number of slices as an input argument. This argument can be supplied to the `NotifiableWorkflow` program
through `program.args` runtime argument as

```
  curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/apps/SparkPiApp/workflows/NotifiableWorkflow/start" \
    -d '{"program.args": "3"}'
```

- Similar to `WordCount`, `NotifiableWorkflow` in `SparkPiApp` can be scheduled to run at a desired interval.

