# Introduction

This repository has two  examples that show how your existing Apache Spark applications can be deployed and configured seamlessly using CDAP. 

This capability allows users of CDAP to run their existing Spark application in CDAP without making any modification to the Spark code.

This repository contains two pre-built example spark applications.

  - word-count-java: contains word count example written in Java 
  - sparkpi-scala: contains sparkpi example written in Scala [how-to](SCALA-SPARK.md)

# How To 

This section will describe how an exisiting Apache Spark code can be integrate into CDAP. So, let us take you through a step-by-step process to get it up and running in CDAP. We will use the examples in this directory to run through it. 

## Step 1 - Building your Spark Application

Assume, you have a project maven or sbt that you used to build your Apache Spark application. We have one in this repository called ```word-count-java``` that is a maven project and it's a Java Spark application. It's a pure Spark application and has no code introduction coming from CDAP. None of the dependencies of CDAP are introduced, [check here](https://github.com/caskdata/sample-spark-app/blob/develop/word-count-java/src/main/java/com/example/spark/JavaWordCount.java#L1)

So, let's first get to building one of the project, but the process is same for the second one.

```
  cd word-count-java
  mvn clean package
```
This will generate a regular Spark application that is now ready to be integrated with CDAP. 

**Only one minor configuration that needs to be done is to setup your Maven POM or SBT project with Felix to create bundled JAR. The artifact will be slightly bigger, but CDAP handles that seamlessly.**

> [Here](BUNDLE-JAR.md) are the changes you need to do to your POM file

## Step 2 - Deploying your Spark application as Plugin to CDAP

Now, this vanilla Spark project is treated as a plugin within CDAP system. So, let's deploy this Spark application using standard REST API's provided by CDAP. 

Deploy `wordcount-1.0.0.jar` in CDAP as a plugin.
```
curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/artifacts/word-count-program" \
   -H 'Artifact-Plugins: [ { "name": "WordCount", "type": "sparkprogram", "className": "com.example.spark.JavaWordCount" }]' \
   -H "Artifact-Version: 1.0.0" \
   -H "Artifact-Extends: system:cdap-notifiable-workflow[1.0.0, 1.0.0]" \
   --data-binary @<path-to-wordcount-1.0.0.jar>
```

> Note that the Spark application is now turned into a CDAP plugin without you doing anything that will be running within a wrapped CDAP application. ```Artifact-Extends``` header specifies the CDAP application this Spark application will be a plugin for. 

Also, note that in the configuration above you specify the main class name that CDAP should use to invoke the your Spark application. 

## Step 3 - Create a CDAP application

> Note that we haven't changed any bits of your Spark application to wrap this into CDAP.

Now, we use a JSON configuration (Note you should be able to modify to use a different format, if JSON is not your favourite choice) to specify the how the application should be configured. Let's inspect the JSON we have for creating this application

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
         "notificationEmailBody": "Execution of Workflow running the WordCount program failed."###
      }
    }
```

### Application Template Artifact 

Key ```$.artifact``` in the above JSON specifies the CDAP application template into which your Spark application will be embedded into. 

### Application Name

Key ```$.config.plugin``` references your Spark application and provides it a name and version as specified in **Step 2**

### Configurations

Rest of the JSON is specifies the configuration for sending emails in case there is an issue with your Spark Application. You can specify multiple recipents in case there is any issue with your Spark application. 

> At this point you should be able to go to your CDAP UI and see your Spark application.

## Run your Spark application in CDAP

Now, that we have deployed your Spark application, wrapped it with CDAP, it's time to run your CDAP application. This specific example that we have used for illustration expects two arguments as specified [here](https://github.com/caskdata/sample-spark-app/blob/develop/word-count-java/src/main/java/com/example/spark/JavaWordCount.java#L19).

Following is the REST you can you use to start the application
```
curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/apps/WordCountApp/workflows/NotifiableWorkflow/start" \
     -d '{"program.args": "/data/input.txt /data/output.dir"}'
```

Few things to note here

* As the CDAP application template is wrapping it in ```NotifiableWorkflow```, you start the workflow. 
* You pass the runtime argument as POST body.
* And because it's a workflow it can scheduled to run periodically based on the set schedule. 

> **We are done, it was quick & easy to integrate your existing application into CDAP.**

## Optional - Working with Schedules - Create, Update & List schedule for your application

### Create a Schedule for your Spark application

Following shows how a schedule can be set to trigger a run of your Spark application at 4:00 AM

```
curl -w"\n" -X PUT "localhost:11015/v3/namespaces/default/apps/WordCountApp/schedules/DailySchedule" \
     -d  '{ "scheduleType": "TIME", "program": { "programName": "NotifiableWorkflow", "programType": "WORKFLOW" }, "properties": {}, "schedule": { "cronExpression": "0 4 * * *", "name": "DailySchedule"} }'
```

Few important things to notice, in the POST body, the configuration for the schedule is defined in there. So, let's look deeper into what's being specified as configuration for adding a new schedule.

* ```scheduleType``` This defines the type of schedule you are attempting to set, there are different kinds supported within CDAP, but, for this application, we only support ```TIME``` based. 
* ```programName``` Specifies the CDAP Program that wraps your spark application. 
* ```schedule``` Specifies the [crontab expression](http://www.adminschoice.com/crontab-quick-reference) to schedule and the name of the schedule. If the schedule with the same name is already present, the REST API call will fail. In that case, please use update to modify the schedule parameters. 

> Note that you have the ability to specify multiple schedules for your Spark application and each one can be managed indepdently of others. 

### List all schedules associated with your Spark application

Verify the schedule is attached to the program by querying the list of schedules.
```
  curl -w"\n" "localhost:11015/v3/namespaces/default/apps/WordCountApp/workflows/NotifiableWorkflow/schedules"
```

### Update the schedule

Following updates the existing schedule to run your Spark application it at 10 AM instead of 4:00 AM that we original had when we created the schedule. 

```
curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/apps/WordCountApp/schedules/DailySchedule/update" \
   -d '{ "scheduleType": "TIME", "program": { "programName": "NotifiableWorkflow", "programType": "WORKFLOW" }, "properties": {}, "schedule": { "cronExpression": "0 10 * * *", "name": "DailySchedule" } }'
```

Nothing special, the command is very similar to CREATE above with minor difference of URI with ```update``` being appended to the end. ```localhost:11015/v3/namespaces/default/apps/WordCountApp/schedules/DailySchedule/update```

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
