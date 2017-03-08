## About
This repo contains two example spark apps that can run on Apache Spark. 
  - word-count-java: contains word count example written in Java 
  - sparkpi-scala: contains sparkpi example written in Scala

Notes below have steps to run these standard examples on directly on Spark and using CDAP without any code change
  
  
## Running examples directly on Spark

1. Word Count
   - Package 
   ```
   cd word-count-java
   mvn clean package
   ``` 
   - Deploy 
   - Run
2. Spark Pi
    - Package 
    - Deploy 
    - Run
 
## Running examples using CDAP without any code change

We will use Notifiable Workflow app to execute these spark programs. Notifiable Workflow app is
a pluggable application which can execute any spark programs without requiring any code changes.
Following steps assume that the deployed version of Notifiable Workflow app in CDAP is 1.0.0

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
    curl -v "localhost:11015/v3/namespaces/default/apps/JavaWordCountApp" -X PUT -d @app.json
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

    At this point you should see the `JavaWordCountApp` application created in the CDAP UI (`localhost:11011`).

   - Run the NotifiableWorkflow program in the JavaWordCountApp through CDAP UI. `main` method of the JavaWordCount
   spark program expects two arguments, name of the input file and name of the output directory. These arguments
   can be supplied to the NotifiableWorkflow program in CDAP through program.args option as
   ```
   program.args = /tmp/input.file /tmp/output.dir
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

    At this point you should see the `SparkPiApp` application created in the CDAP UI (`localhost:11011`).

   - Run the NotifiableWorkflow program in the SparkPiApp through CDAP UI. `main` method of the SparkPi program
    expects number of slices as an input argument. This argument can be supplied to the NotifiableWorkflow program
    in CDAP through `program.args` option as
    ```
    program.args = 3
    ```
   
   
## References 
1. Notifiable workflow app https://github.com/caskdata/cdap-notifiable-workflow-app
      
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
