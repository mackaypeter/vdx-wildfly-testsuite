# vdx-wildfly-testsuite
Testsuite for projectodd/vdx [https://github.com/projectodd/vdx/] pretty print feature in WildFly

[![Build Status](https://travis-ci.org/jboss-eap-qe/vdx-wildfly-testsuite.svg?branch=master)](https://travis-ci.org/jboss-eap-qe/vdx-wildfly-testsuite)

Running the Testsuite
-------------------

Ensure you have JDK 8 (or newer) installed

> java -version

Ensure you have Maven 3.2.5 (or newer) installed

> mvn -version

To run against local (e.g. latest WildFly master build) server use following command:

> mvn -P all test -Djboss.home=/path/to/location/of/the/server

To run against released WildFly 10.1.0.Final (will be downloaded) use following command, :

> mvn -P all test

Profile `all` redirects test output to file, so look at $TEST_NAME-output.txt files in target/surefire directory if needed.


Running one test
-------------------

To run test against application server running in standalone mode:

> mvn test -Djboss.home=/path/to/location/of/the/server -Dtest=$TEST

To run test against application server running in domain mode:

> mvn test -Djboss.home=/path/to/location/of/the/server -Ddomain -Dtest=$TEST

`-Ddomain` is hint to use correct `wildfly-arquillian-container` artifact

Of course you can specify concrere test not just whole TestCase class - for example `JBossWSTestCase#incorrectValueOfModifyWsdlAddressElement`


Running internationalization and localization tests
-------------------

SmokeStandaloneTestCase#emptyConfigFile test is enhanced to support check based on selected language.
Supported locales are:
 * de_DE
 * es_ES
 * fr_FR
 * ja_JP
 * pt_BR
 * zh_CN

Iterating the test over all supported locales:
```bash
LANGUAGES=(
'-Duser.country=DE -Duser.language=de'
'-Duser.country=ES -Duser.language=es'
'-Duser.country=JP -Duser.language=ja'
'-Duser.country=FR -Duser.language=fr'
'-Duser.country=CN -Duser.language=zh'
'-Duser.country=BR -Duser.language=pt'
)

for index in ${!LANGUAGES[*]}; do
    LANGUAGE=${LANGUAGES[$index]}
    SUFFIX=`echo "$LANGUAGE" | sed "s/.*=\(..\).*=\(..\).*/\2_\1/g"`
    echo "$LANGUAGE ... $SUFFIX"
    mvn test -Duser.country.and.language="$LANGUAGE" -Dsurefire.reportNameSuffix="$SUFFIX" \
      -Dtest=SmokeStandaloneTestCase#emptyConfigFile -Dversion.org.jboss.wildfly.dist=11.0.0.CR1
done
```

Used technologies
-----------------

JDK 8 (https://docs.oracle.com/javase/8)
 
Apache Maven 3.2.5 (https://maven.apache.org)

Creaper (https://github.com/wildfly-extras/creaper/)

Arquillian (http://arquillian.org/)

JUnit (http://junit.org/junit4/)

Groovy (http://www.groovy-lang.org/)


Travis CI
---------

There is Travis CI configured to test PRs. 


Technical Details 
-----------------
Tests for standalone and domain mode are differentiated by JUnit annotation `@Category(StandaloneTests.class)` or `@Category(DomainTests.class)`

Modification of the configuration file is driven by ServerConfig annotation, underlying technology is based on xml transformations done via Creaper and Groovy.
Custom configuration files or modification scripts can be specified by ServerConfig annotation, modification scripts can be parametrized.
There is support to use directly offline Creaper commands if ServerConfig annotation does not provide means necessary for test scenario.
Groovy transformations are taken from `src/test/resources/org/wildfly/test/integration/vdx/transformations`


Test-suite takes care of creating backup of the configuration before changes are applied. Configuration is restored after every test to avoid interference with other tests.
Logs from the server and modified server configurations are archived in `target/test-output/` directory, structure is multi-level based on TestCase and test names.

Example of typical TestCase class definition for standalone mode:
```java
@RunAsClient
@RunWith(Arquillian.class)
@Category(StandaloneTests.class)
public class YourNewTestCase extends TestBase {
    ...
}
```

Example of test definition for webservices:
```java
    @Test
    @ServerConfig(configuration = "standalone.xml", xmlTransformationGroovy = "webservices/AddIncorrectlyNamedModifyWsdlAddressElement.groovy",
            subtreeName = "webservices", subsystemName = "webservices")
    public void newTest() throws Exception {
        container().tryStartAndWaitForFail();

        String errorLog = container().getErrorMessageFromServerStart();
        assertContains(errorLog, "OPVDX001: Validation error");
        ...
    }
```

Code coverage
-----------------
JaCoCo is used to generate code coverage data, `jacoco` profile was introduced to generate coverage data which are stored in `./target/jacoco.exec` file.

```bash
mvn clean test -Djboss.home=/path/to/location/of/the/server -Pjacoco,all
```

To view results you need to compile VDX sources and push data to SonarQube or generate the report using JaCoCo maven plugin
```bash
git clone git@github.com:projectodd/vdx.git
cd vdx
mvn package
```

JaCoCo report:
```bash
mvn org.jacoco:jacoco-maven-plugin:0.7.9:report -Djacoco.dataFile=/home/rsvoboda/git/vdx-wildfly-testsuite/target/jacoco.exec
firefox core/target/site/jacoco/index.html &
firefox wildfly/target/site/jacoco/index.html &
```

SonarQube:
```bash
wget https://sonarsource.bintray.com/Distribution/sonarqube/sonarqube-6.5.zip && unzip -q sonarqube-6.5.zip
sonarqube-6.5/bin/linux-x86-64/sonar.sh start

mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.2:sonar -Dsonar.jacoco.reportPaths=/home/rsvoboda/git/vdx-wildfly-testsuite/target/jacoco.exec
firefox http://localhost:9000/ &

sonarqube-6.5/bin/linux-x86-64/sonar.sh stop
```