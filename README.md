## About
This repository contains two example spark apps that can run on Apache Spark.
  - word-count-java: contains word count example written in Java 
  - sparkpi-scala: contains sparkpi example written in Scala

Below are the steps to run these standard examples on CDAP.

## Updates to the `pom.xml`

- Changes required to create bundle jar

```
      <plugin>
        <groupId>org.apache.felix</groupId>
        <artifactId>maven-bundle-plugin</artifactId>
        <version>2.3.7</version>
        <extensions>true</extensions>
        <configuration>
          <instructions>
            <Embed-Dependency>*;inline=false;scope=compile</Embed-Dependency>
            <Embed-Transitive>true</Embed-Transitive>
            <Embed-Directory>lib</Embed-Directory>
            <_exportcontent>*</_exportcontent>
          </instructions>
        </configuration>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>bundle</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
```

- Dependencies for `spark-core`, `spark-streaming`, `spark-mllib`, and `spark-sql` are provide by CDAP.
 So if the legacy program is using these dependencies then the `pom.xml` should be updated to
 have `provided` scope for them.

For example:

```
    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-core_2.10</artifactId>
      <version>1.6.1</version>
      <scope>provided</scope>
    </dependency>

    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-streaming_2.10</artifactId>
      <version>1.6.1</version>
      <scope>provided</scope>
    </dependency>

    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-mllib_2.10</artifactId>
      <version>1.6.1</version>
      <scope>provided</scope>
    </dependency>

    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-sql_2.10</artifactId>
      <version>1.6.1</version>
      <scope>provided</scope>
    </dependency>
```
  
## Running examples using CDAP

We will use Notifiable Workflow app(https://github.com/caskdata/cdap-notifiable-workflow-app) to execute
these spark programs. Notifiable Workflow app is a pluggable CDAP application which can execute any
spark programs without requiring any code changes. Following steps assume that the deployed version of
Notifiable Workflow app in CDAP is 1.0.0.

1. Word Count 
   - Package  
    ```
    cd word-count-java
    mvn clean package
    ```

   - Deploy `wordcount-1.0.0.jar` in CDAP as a plugin.
    ```
    curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/artifacts/word-count-program" \
      -H 'Artifact-Plugins: [ { "name": "WordCount", "type": "sparkprogram", "className": "com.example.spark.JavaWordCount" }]' \
      -H "Artifact-Version: 1.0.0" \
      -H "Artifact-Extends: system:cdap-notifiable-workflow[1.0.0, 1.0.0]" \
      --data-binary @<path-to-wordcount-1.0.0.jar>
    ```

   - Create a CDAP Application with the plugin just deployed.
    ```
    curl -w"\n" "localhost:11015/v3/namespaces/default/apps/WordCountApp" -X PUT -d @app.json
    ```

    where app.json contains the configurations which are used to configure the applications.
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

    At this point `WordCountApp` should be created for you.

   - Run the `NotifiableWorkflow` program in the `WordCountApp`. `main` method of the `JavaWordCount`
   spark program expects two arguments, name of the input file and name of the output directory.
   These arguments can be supplied to the `NotifiableWorkflow` program through `program.args` runtime argument as
   ```
   curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/apps/WordCountApp/workflows/NotifiableWorkflow/start" \
   -d '{"program.args": "/data/input.txt /data/output.dir"}'
   ```

   - `NotifiableWorkflow` program can be scheduled to run at desired interval.
   Following command schedules it to run at every day 4 AM.
   ```
   curl -w"\n" -X PUT "localhost:11015/v3/namespaces/default/apps/WordCountApp/schedules/DailySchedule" \
    -d  '{ "scheduleType": "TIME", "program": { "programName": "NotifiableWorkflow", "programType": "WORKFLOW" }, "properties": {}, "schedule": { "cronExpression": "0 4 * * *", "name": "DailySchedule"} }'
   ```

   - Verify the schedule is attached to the program by querying the list of schedules.
   ```
   curl -w"\n" "localhost:11015/v3/namespaces/default/apps/WordCountApp/workflows/NotifiableWorkflow/schedules"
   ```

   - Schedule associated with the `NotifiableWorklow` can be easily updated.
   Following command update the schedule to run it at 10 AM instead.
   ```
   curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/apps/WordCountApp/schedules/DailySchedule/update" \
    -d '{ "scheduleType": "TIME", "program": { "programName": "NotifiableWorkflow", "programType": "WORKFLOW" }, "properties": {}, "schedule": { "cronExpression": "0 10 * * *", "name": "DailySchedule" } }'
   ```

2. Spark Pi 
   - Package 
    ```
    cd spark-pi-scala
    mvn clean package
    ```

   - Deploy `sparkpi-1.0.0.jar` in CDAP as a plugin.
    ```
    curl -w"\n" -X POST "localhost:11015/v3/namespaces/default/artifacts/sparkpi-program" \
     -H 'Artifact-Plugins: [ { "name": "SparkPi", "type": "sparkprogram", "className": "com.example.spark.SparkPi" }]' \
     -H "Artifact-Version: 1.0.0" \
     -H "Artifact-Extends: system:cdap-notifiable-workflow[1.0.0, 1.0.0]" \
     --data-binary @<path-to-sparkpi-1.0.0.jar>
    ```

   - Create CDAP Application with the plugin just deployed.
    ```
    curl -v "localhost:11015/v3/namespaces/default/apps/SparkPiApp" -X PUT -d @app.json
    ```

    where app.json contains the configurations which are used to configure the applications.
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

   - Similar to `WordCountApp`, `NotifiableWorkflow` in `SparkPiApp` can be scheduled to run at a desired interval.

## Mailing Lists

CDAP User Group and Development Discussions:

- `cdap-user@googlegroups.com <https://groups.google.com/d/forum/cdap-user>`__

The *cdap-user* mailing list is primarily for users using the product to develop
applications or building plugins for appplications. You can expect questions from 
users, release announcements, and any other discussions that we think will be helpful 
to the users.

## License and Trademarks

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
