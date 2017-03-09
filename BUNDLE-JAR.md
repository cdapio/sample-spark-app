# Introduction

Following are the updates that you would need to apply to your POM or sbt project. They are simple and easy. 

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
