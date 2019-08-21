# Deprecated
This way of running existing Spark applications has been removed from newer versions of CDAP. See https://www.youtube.com/watch?v=gDnINRBzg2s.

# Introduction

This guide demonstrates how to manage and run your existing Apache Spark applications in CDAP. This repository has two examples and step-by-step instructions for achieving this. 

You will be able to run an existing Spark application in CDAP without making any modifications to the Spark code.

Sample Spark examples used in this guide:

  - word-count-java: contains a word count example written in Java 
  - sparkpi-scala: contains a sparkpi example written in Scala 

These instructions are for deploying the word-count-java Spark application. For deploying the sparkpi-scala application, follow [these instructions](SCALA-SPARK.md). 

# Summary of steps

1. Package the word-count-java Spark application as a bundle JAR
2. Deploy the Spark application JAR as a CDAP plugin 
3. Create a CDAP Application using the deployed plugin
4. Run your Spark application in CDAP   

## Step 1: Package the word-count-java Spark application as a bundle JAR

In this section, instructions are shown for a maven project; the steps are similar for other build tools such as ant or sbt. 
Please note that in these applications, no dependencies from CDAP are used and the compiled application JAR would work directly on Apache Spark: [check here](https://github.com/caskdata/sample-spark-app/blob/develop/word-count-java/src/main/java/com/example/spark/JavaWordCount.java#L1)

Let's build the Spark word-count-java application. The process will be same for any Spark-based application that you would like to run in CDAP.

```
  git clone git@github.com:caskdata/sample-spark-app
  cd sample-spark-app/word-count-java
  mvn clean package
```
These commands will generate a Spark application JAR that is now ready to be integrated with CDAP. 

**Note:** The packaged jar should have all the dependencies required by the application. The pom.xml uses the Felix maven-bundle-plugin to create a bundled JAR with all dependencies.

> Refer to [How-To-Create-A-Bundle-JAR](BUNDLE-JAR.md) to see what modification are needed to your pom.xml file. 

## Step 2: Deploy the Spark application JAR as a CDAP plugin

The CDAP plugin architecture allows reusing existing code to create new CDAP application. The Spark application built in Step 1 can be deployed as a plugin in CDAP using a RESTful API. 

Deploy `wordcount-1.0.0.jar` in CDAP as a plugin:

```
curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/artifacts/word-count-program" \
   -H 'Artifact-Plugins: [ { "name": "WordCount", "type": "sparkprogram", "className": "com.example.spark.JavaWordCount" }]' \
   -H "Artifact-Version: 1.0.0" \
   -H "Artifact-Extends: system:cdap-notifiable-workflow[1.0.0, 1.0.0]" \
   --data-binary @/PATH/TO/wordcount-1.0.0.jar
```

> Note that the Spark application is now turned into a CDAP plugin without any modifications to the Spark code. ```Artifact-Extends``` header specifies that the CDAP application that will run the Spark code. The main class name for the Spark application is specified using the ```className``` parameter. 

Make sure you replace ```/PATH/TO/wordcount-1.0.0.jar``` with the appropriate JAR and its path. 

## Step 3: Create a CDAP application using the deployed plugin

A CDAP application can be created from the Spark plugin deployed in the previous step using a RESTful API. 

Use this JSON configuration file ```app.json``` for configuring the CDAP application:

```
  {
    "artifact": {
      "name": "cdap-notifiable-workflow",
      "version": "1.0.0",
      "scope": "system"
    },
    "config": {
      "plugin": {
        "name": "WordCount",
        "type": "sparkprogram",
        "artifact": {
          "name": "word-count-program",
          "scope": "user",
          "version": "1.0.0"
        }
      },

      "notificationEmailSender": "sender@example.domain.com",
      "notificationEmailIds": ["recipient@example.domain.com"],
      "notificationEmailSubject": "[Critical] Workflow execution failed.",
      "notificationEmailBody": "Execution of Workflow running the WordCount program failed."
    }
  }
```

RESTful API to create the CDAP WordCount Application:

```
curl -v "localhost:11015/v3/namespaces/default/apps/WordCount" -X PUT -d @app.json
```

### Application Template Artifact 

The key ```artifact``` in the above JSON specifies the CDAP application template into which your Spark application will be embedded into. 

### Application Name

The key ```config.plugin``` references your Spark application and provides it with a name and version as specified in **Step 2**

> At this point, you should be able to go to your CDAP UI and see the WordCount application.

## Step 4: Run your Spark application in CDAP

The Spark application can now be run in CDAP. This example that we have used to illustrate this process expects two arguments (an input file and an output directory) as specified [here](https://github.com/caskdata/sample-spark-app/blob/develop/word-count-java/src/main/java/com/example/spark/JavaWordCount.java#L19).

The application can be started using the CDAP UI or the RESTful API:
```
curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/apps/WordCount/workflows/NotifiableWorkflow/start" \
     -d '{"program.args": "/data/input.txt /data/output.dir"}'
```

The status of the application can be monitored through the CDAP UI. Once the Spark application finishes running, the output will be available in the output directory specified.

## Advanced Options

Now that the Spark is running in CDAP, you can leverage the additional capabilities provided by the CDAP platform such as scheduling, logs, metadata, and security. The next section describes the scheduling capability. 

## Working with Schedules: Create, Update, and List schedules for your application

### Create a Schedule for your Spark application

Create a daily schedule to trigger a run of your Spark application at 4:00 AM every day using this RESTful API:

```
curl -w"\n" -X PUT "localhost:11015/v3/namespaces/default/apps/WordCount/schedules/DailySchedule" \
     -d  '{ "scheduleType": "TIME", "program": { "programName": "NotifiableWorkflow", "programType": "WORKFLOW" }, "properties": {}, "schedule": { "cronExpression": "0 4 * * *", "name": "DailySchedule"} }'
```

The POST body contains the configuration for the schedule to be created:

* ```scheduleType``` This defines the type of schedule you are attempting to set. 
* ```programName``` Specifies the CDAP Program that runs the Spark plugin. 
* ```schedule``` Specifies the [cron expression](http://www.quartz-scheduler.org/documentation/quartz-2.1.x/tutorials/crontrigger.html) for the schedule and the name of the schedule. 

> Note: multiple schedules can be set for the Spark application, and each one can be managed independently of others. 

### List all schedules associated with your Spark application

Verify the schedule is attached to the program by querying the list of schedules:

```
  curl -w"\n" "localhost:11015/v3/namespaces/default/apps/WordCount/workflows/NotifiableWorkflow/schedules"
```

### Update the schedule

To change the schedule to run the application at 10:00 AM instead of 4:00 AM, use this RESTful API: 

```
curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/apps/WordCount/schedules/DailySchedule/update" \
   -d '{ "scheduleType": "TIME", "program": { "programName": "NotifiableWorkflow", "programType": "WORKFLOW" }, "properties": {}, "schedule": { "cronExpression": "0 10 * * *", "name": "DailySchedule" } }'
```

# Mailing Lists

CDAP User Group and Development Discussions:

- [cdap-user@googlegroups.com](https://groups.google.com/d/forum/cdap-user)

The *cdap-user* mailing list is primarily for users using the product to develop
applications or building plugins for appplications. You can expect questions from 
users, release announcements, and any other discussions that we think will be helpful 
to the users.

# License and Trademarks

Copyright © 2017 Cask Data, Inc.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except
in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the 
License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, 
either express or implied. See the License for the specific language governing permissions 
and limitations under the License.

Cask is a trademark of Cask Data, Inc. All rights reserved.

Apache, Apache HBase, and HBase are trademarks of The Apache Software Foundation. Used with
permission. No endorsement by The Apache Software Foundation is implied by the use of these marks.
