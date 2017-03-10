# Introduction

This guide will demonstrate how to manage and run your existing Apache Spark application in CDAP. This repository has two examples and a step-by-step instructions in acheiving this. 

Users will be able to run their existing Spark application in CDAP without making any modifications to the Spark code.

Sample spark examples used in this guide:

  - word-count-java: contains word count example written in Java 
  - sparkpi-scala: contains sparkpi example written in Scala 

The following instructions are for deploying the word-count-java Spark application. For deploying Scala Spark app, follow the instructions [here](SCALA-SPARK.md). 

# Summary of steps

1. Package Spark application (word-count) as a bundle jar
2. Deploy the Spark application jar as a CDAP plugin 
3. Create a CDAP Application using the Spark plugin
4. Run the application in CDAP   

## Step 1 - Package your Spark Application

In this section the instructions are demonstrated for maven project, the steps are similar for other build tools such as ant or sbt. 
are  that you have a maven project for your Apache Spark application. Please note, in these applications none of the dependencies of CDAP are introduced and the compiled application jar would work directly on Apache Spark: [check here](https://github.com/caskdata/sample-spark-app/blob/develop/word-count-java/src/main/java/com/example/spark/JavaWordCount.java#L1)

Let's build the Spark word-count application. The process will be same for any Spark based applications that you would like to run in CDAP.

```
  git clone git@github.com:caskdata/sample-spark-app
  cd sample-spark-app/word-count-java
  mvn clean package
```
The commands above will generate a regular Spark application that is now ready to be integrated with CDAP. 

**Note: The packaged jar should have all the dependencies required by the application. The pom.xml uses Felix maven-bundle-plugin to create a bundled JAR with all dependencies**

> Refer to [How-To-Create-Bundle-JAR](BUNDLE-JAR.md) to see what modification need to be done to your POM file. 

## Step 2 - Deploy Spark application into CDAP

CDAP plugin architecture allows reusing existing code to create new CDAP application. The Spark application built in Step 1 can be deployed as a plugin in CDAP using REST APIs. 

Deploy `wordcount-1.0.0.jar` in CDAP as a plugin:

```
curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/artifacts/word-count-program" \
   -H 'Artifact-Plugins: [ { "name": "WordCount", "type": "sparkprogram", "className": "com.example.spark.JavaWordCount" }]' \
   -H "Artifact-Version: 1.0.0" \
   -H "Artifact-Extends: system:cdap-notifiable-workflow[1.0.0, 1.0.0]" \
   --data-binary @/PATH/TO/wordcount-1.0.0.jar
```

> Note that the Spark application is now turned into a CDAP plugin without any modification to the Spark code. ```Artifact-Extends``` header specifies the CDAP application that will run the Spark code. The driver class name for the Spark application is specified using the ```className``` parameter. 

Make sure you replace ```/PATH/TO/wordcount-1.0.0.jar``` to appropriate JAR and it's path. 

## Step 3 - Create a CDAP application

CDAP application can be created from the Spark plugin deployed in the previous step using REST API. 

Use the following JSON configuration file ```app.json``` for configuring the CDAP application.

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
      }
    }
```

REST API to create the CDAP WordCount Application:

```
curl -v "localhost:11015/v3/namespaces/default/apps/WordCount" -X PUT -d @app.json
```

### Application Template Artifact 

Key ```artifact``` in the above JSON specifies the CDAP application template into which your Spark application will be embedded into. 

### Application Name

Key ```config.plugin``` references your Spark application and provides it a name and version as specified in **Step 2**

> At this point you should be able to go to your CDAP UI and see the WordCount application.

## Step 4 - Run your Spark application in CDAP

The Spark application can now be run in CDAP. This specific example that we have used for illustration expects two arguments (input file and output directory) as specified [here](https://github.com/caskdata/sample-spark-app/blob/develop/word-count-java/src/main/java/com/example/spark/JavaWordCount.java#L19).

The application can be started using the UI or the REST API:
```
curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/apps/WordCount/workflows/NotifiableWorkflow/start" \
     -d '{"program.args": "/data/input.txt /data/output.dir"}'
```

The status of the application can be monitored through the UI. Once the Spark application finishes running the output will be available in the output directory specified.

## Advanced Options

Now that the Spark program is running as a CDAP Application, users can leverage the additional capabilities provided by the CDAP platform such as - scheduling, logs, metadata, security and the like. The following section describes scheduling capability. 

## Working with Schedules - Create, Update & List schedule for your application

### Create a Schedule for your Spark application

Create a daily schedule to trigger a run of your Spark application at 4:00 AM every day using the following REST API:

```
curl -w"\n" -X PUT "localhost:11015/v3/namespaces/default/apps/WordCount/schedules/DailySchedule" \
     -d  '{ "scheduleType": "TIME", "program": { "programName": "NotifiableWorkflow", "programType": "WORKFLOW" }, "properties": {}, "schedule": { "cronExpression": "0 4 * * *", "name": "DailySchedule"} }'
```

The POST body contains the configuration for the schedule created. Details as follows:

* ```scheduleType``` This defines the type of schedule you are attempting to set. 
* ```programName``` Specifies the CDAP Program that runs the Spark plugin. 
* ```schedule``` Specifies the [cron expression](http://www.adminschoice.com/crontab-quick-reference) for the schedule and the name of the schedule. 

> Note: multiple schedules can be set for the Spark application, and each one can be managed indepdently of others. 

### List all schedules associated with your Spark application

Verify the schedule is attached to the program by querying the list of schedules.

```
  curl -w"\n" "localhost:11015/v3/namespaces/default/apps/WordCount/workflows/NotifiableWorkflow/schedules"
```

### Update the schedule

To change the schedule to run the application at 10:00 AM instead of 4:00 AM, use the following REST API: 

```
curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/apps/WordCount/schedules/DailySchedule/update" \
   -d '{ "scheduleType": "TIME", "program": { "programName": "NotifiableWorkflow", "programType": "WORKFLOW" }, "properties": {}, "schedule": { "cronExpression": "0 10 * * *", "name": "DailySchedule" } }'
```

# Mailing Lists

CDAP User Group and Development Discussions:

- `cdap-user@googlegroups.com <https://groups.google.com/d/forum/cdap-user>`__

The *cdap-user* mailing list is primarily for users using the product to develop
applications or building plugins for appplications. You can expect questions from 
users, release announcements, and any other discussions that we think will be helpful 
to the users.

# License and Trademarks

Copyright © 2016-2017 Cask Data, Inc.

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
