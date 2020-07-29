# Payara Showcase

[![GitHub last commit](https://img.shields.io/github/last-commit/stephan-mueller/payara-showcase)](https://github.com/stephan-mueller/payara-showcase/commits) 
[![GitHub](https://img.shields.io/github/license/stephan-mueller/payara-showcase)](https://github.com/stephan-mueller/payara-showcase/blob/master/LICENSE)
[![CircleCI](https://circleci.com/gh/stephan-mueller/payara-showcase.svg?style=shield)](https://app.circleci.com/pipelines/github/stephan-mueller/payara-showcase)
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=stephan-mueller_payara-showcase&metric=alert_status)](https://sonarcloud.io/dashboard?id=stephan-mueller_payara-showcase)
[![Coverage](https://sonarcloud.io/api/project_badges/measure?project=stephan-mueller_payara-showcase&metric=coverage)](https://sonarcloud.io/dashboard?id=stephan-mueller_payara-showcase)

This is a project based on the microservice framework [Payara Micro](https://www.payara.fish/products/payara-micro/). 
It contains a hello world application, which demonstrates features of Payara Micro and Eclipse Microprofile.

Software requirements to run the samples are `maven`, `openjdk-8` (or any other JDK 8) and `docker`.
When running the Maven lifecycle it will create the war package and uses the `payara-micro-maven-plugin` to create a executable JAR (fat 
JAR) which contains the application and the microservices-ready version of the Payara application server. The fat JAR will be copied into a 
Docker image using Spotify's `dockerfile-maven-plugin` during the package phase.

**Notable Features:**
* Dockerfiles for executable JAR & Server
* Integration of MP Health, MP Metrics and MP OpenAPI
* Testcontainer tests with REST-assured, Cucumber and Postman/Newman
* [CircleCI](https://circleci.com) Integration
* [Sonarcloud](https://sonarcloud.io) Integration


## How to run

Before running the application it needs to be compiled and packaged using `Maven`. It creates the executable JAR and Docker image and can 
be run via `docker`:

```shell script
$ mvn clean package
$ docker run --rm -p 8080:8080 payara-showcase
```

For the Payara Server a multi-stage Docker image is provided. It can be created and run via `docker`:    
```shell script
$ docker build -f Dockerfile.server -t payara-server-showcase .
$ docker run --rm -p 8080:8080 payara-server-showcase
```

Wait for a message log similar to this:

> [2020-07-14T17:39:34.789+0200] [] [INFO] [] [PayaraMicro] [tid: _ThreadID=1 _ThreadName=main] [timeMillis: 1594741174789] [levelValue: 800] Payara Micro  5 #badassmicrofish (build 547) ready in 7.127 (ms)

If everything worked you can access the OpenAPI UI via [http://localhost:8080/swagger-ui](http://localhost:8080/swagger-ui).

### Resolving issues

Sometimes it may happen that the containers did not stop as expected when trying to stop the pipeline early. This may
result in running containers although they should have been stopped and removed. To detect them you need to check
Docker:

```shell script
$ docker ps -a | grep payara-showcase
```

If there are containers remaining although the application has been stopped you can remove them:

```shell script
$ docker rm <ids of the containers>
```


## Features

### Application 

The application is a very simple "Hello World" greeting service. It supports GET requests for generating a greeting message, and a PUT 
request for changing the greeting itself. The response is encoded using JSON.

Try the application
```shell script
curl -X GET http://localhost:8080/api/greet
{"message":"Hello World!"}

curl -X GET http://localhost:8080/api/greet/Stephan
{"message":"Hello Stephan!"}

curl -X PUT -H "Content-Type: application/json" -d '{"greeting" : "Hola"}' http://localhost:8080/api/greet/greeting

curl -X GET http://localhost:8080/api/greet/greeting
{"greeting":"Hola"}

curl -X GET http://localhost:8080/api/greet/Max
{"message":"Hola Max!"}
```

### Health, Metrics and OpenAPI

The application server provides built-in support for health, metrics and openapi endpoints.

Health liveness and readiness
```shell script
curl -s -X GET http://localhost:8080/health

curl -s -X GET http://localhost:8080/health/live

curl -s -X GET http://localhost:8080/health/ready
```

Metrics in Prometheus / JSON Format
```shell script
curl -s -X GET http://localhost:8080/metrics

curl -H 'Accept: application/json' -X GET http://localhost:8080/metrics
```

OpenAPI in YAML / JSON Format
```shell script
curl -s -X GET http://localhost:8080/openapi

curl -H 'Accept: application/json' -X GET http://localhost:8080/openapi
```

### Testcontainer tests with REST-assured, Cucumber and Postman/Newman

For the application a set of integration tests is provided. The tests bases on Testcontainers combined with other testing frameworks like 
REST-assured, Cucumber and Postman/Newman. The docker container for the application is build by the `dockerfile-maven-plugin` during the 
`package` phase.

To improve the runtime of the testcontainer tests by avoid starting and stopping the container for every test class, the 
[singleton container](https://www.Testcontainers.org/test_framework_integration/manual_lifecycle_control/) pattern is used.

The container is started only once when the base class is loaded. The container can then be used by all inheriting test classes. At the end 
of the test suite the Ryuk container that is started by Testcontainers core will take care of stopping the singleton container.

AbstractIntegrationTest - Superclass for all Testcontainers tests providing the containerized application
```java
public abstract class AbstractIntegrationTest {

  protected static final String NETWORK_ALIAS_APPLICATION = "application";

  protected static final Network NETWORK = Network.newNetwork();

  protected static final GenericContainer<?> APPLICATION = new GenericContainer<>("payara-showcase")
    .withExposedPorts(8080)
    .withNetwork(NETWORK)
    .withNetworkAliases(NETWORK_ALIAS_APPLICATION)
    .waitingFor(Wait.forLogMessage(".* ready in .*", 1));

  static {
    APPLICATION.start();
  }
}
```  

#### Integration tests with Testcontainer and REST-assured

[REST-assured](http://rest-assured.io) is a popular testframework for testing and validating REST services that brings the the simplicity 
of dynamic languages into the Java domain. 

To ease making HTTP requests to the containerized application, REST-assured provides specifications to reuse response expectations and/or 
request parameters for different tests. The `RequestSpecBuilder` is used to define the dynamic port of the application for all requests only 
once. 

GreetingResourceIT - Integration tests for the GreetingResource
```java
class GreetResourceIT extends AbstractIntegrationTest {

  private static final Logger LOG = LoggerFactory.getLogger(GreetResourceIT.class);

  private static RequestSpecification requestSpecification;

  @BeforeAll
  static void setUpUri() {
    APPLICATION.withLogConsumer(new Slf4jLogConsumer(LOG));

    requestSpecification = new RequestSpecBuilder()
        .setPort(APPLICATION.getFirstMappedPort())
        .build();

    RestAssured.given(requestSpecification)
        .contentType(MediaType.APPLICATION_JSON)
        .body("{ \"greeting\" : \"Hello\" }")
        .when()
        .put("/api/greet/greeting")
        .then()
        .statusCode(Response.Status.NO_CONTENT.getStatusCode());
  }

  @Test
  void greetTheWorld() {
    RestAssured.given(requestSpecification)
        .accept(MediaType.APPLICATION_JSON)
        .when()
        .get("/api/greet")
        .then()
        .statusCode(Response.Status.OK.getStatusCode())
        .contentType(MediaType.APPLICATION_JSON)
        .body("message", Matchers.equalTo("Hello World!"));
  }
}
```

#### Acceptance tests with Testcontainer, REST-assured and Cucumber

[Cucumber](https://github.com/cucumber/cucumber-jvm) is one of the most popular tools that supports Behaviour-Driven Development(BDD) for 
the Java language. Cucumber reads executable specifications written in natural language and validates that the software does what those 
specifications say. The specifications consist of several examples or scenarios - which is why this approach is known as 
[Specification by Example](https://en.wikipedia.org/wiki/Specification_by_example).

Greeting.feature - Acceptance tests in natural language (Gherkin syntax)
```gherkin
Feature: Greeting

  Scenario: Greet the world
    Given a greeting "Hello"
    When a user wants to greet
    Then the message is "Hello World!"

  Scenario Outline: Greet someone
    Given a greeting "<greeting>"
    When a user wants to greet "<name>"
    Then the message is "<greeting> <name>!"

    Examples:
      | greeting | name      |
      | Hola     | Christian |
      | Hey      | Max       |
      | Moin     | Stephan   |
```

To run cucumber tests, you still have to to use the `Cucumber` JUnit4 runner, due to missing support for JUnit5.

GreetingCucumberIT - JUnit4 based test class that runs all acceptance tests of the project
```java
@RunWith(Cucumber.class)
@CucumberOptions(plugin = {"pretty"}, features = "src/test/resources/it/feature")
public class GreetingCucumberIT {
}
```

Due to its BDD-oriented nature, REST-assured seamlessly integrates with Cucumber to implement acceptance tests for RESTful APIs. To 

GreetingCucumberSteps - Step definitions matching the steps in the feature file
```java
public class GreetingCucumberSteps extends AbstractIntegrationTest {

  private RequestSpecification requestSpecification;

  private io.restassured.response.Response response;

  @Before
  public void beforeScenario() {
    requestSpecification = new RequestSpecBuilder()
        .setPort(APPLICATION.getFirstMappedPort())
        .build();
  }

  @Given("a greeting {string}")
  public void given_a_greeting(final String greeting) {
    RestAssured.given(requestSpecification)
        .contentType(MediaType.APPLICATION_JSON)
        .body(new GreetingDTO(greeting))
        .when()
        .put("/api/greet/greeting")
        .then()
        .statusCode(Response.Status.NO_CONTENT.getStatusCode());
  }

  @When("a user wants to greet")
  public void when_a_user_wants_to_greet() {
    response = RestAssured.given(requestSpecification)
        .accept(MediaType.APPLICATION_JSON)
        .when()
        .get("/api/greet");
  }

  @When("a user wants to greet {string}")
  public void when_a_user_wants_to_greet(final String name) {
    response = RestAssured.given(requestSpecification)
        .accept(MediaType.APPLICATION_JSON)
        .pathParam("name", name)
        .when()
        .get("/api/greet/{name}");
  }

  @Then("the message is {string}")
  public void then_the_message_is(final String message) {
    response.then()
        .statusCode(Response.Status.OK.getStatusCode())
        .contentType(MediaType.APPLICATION_JSON)
        .body("message", Matchers.equalTo(message));
  }
}
```

As expected the execution of the specification examples can also be easily followed in the log output

Cucucmber log output
```text
[INFO] Running de.openknowledge.projects.greet.GreetingCucumberIT

Scenario: Greet the world                # src/test/resources/it/feature/Greeting.feature:3
  Given a greeting "Hello"               # de.openknowledge.projects.greet.GreetingCucumberSteps.given_a_greeting(java.lang.String)
  When a user wants to greet             # de.openknowledge.projects.greet.GreetingCucumberSteps.when_a_user_wants_to_greet()
  Then the message is "Hello World!"     # de.openknowledge.projects.greet.GreetingCucumberSteps.then_the_message_is(java.lang.String)

Scenario Outline: Greet someone          # src/test/resources/it/feature/Greeting.feature:15
  Given a greeting "Hola"                # de.openknowledge.projects.greet.GreetingCucumberSteps.given_a_greeting(java.lang.String)
  When a user wants to greet "Christian" # de.openknowledge.projects.greet.GreetingCucumberSteps.when_a_user_wants_to_greet(java.lang.String)
  Then the message is "Hola Christian!"  # de.openknowledge.projects.greet.GreetingCucumberSteps.then_the_message_is(java.lang.String)

Scenario Outline: Greet someone          # src/test/resources/it/feature/Greeting.feature:16
  Given a greeting "Hey"                 # de.openknowledge.projects.greet.GreetingCucumberSteps.given_a_greeting(java.lang.String)
  When a user wants to greet "Max"       # de.openknowledge.projects.greet.GreetingCucumberSteps.when_a_user_wants_to_greet(java.lang.String)
  Then the message is "Hey Max!"         # de.openknowledge.projects.greet.GreetingCucumberSteps.then_the_message_is(java.lang.String)

Scenario Outline: Greet someone          # src/test/resources/it/feature/Greeting.feature:17
  Given a greeting "Moin"                # de.openknowledge.projects.greet.GreetingCucumberSteps.given_a_greeting(java.lang.String)
  When a user wants to greet "Stephan"   # de.openknowledge.projects.greet.GreetingCucumberSteps.when_a_user_wants_to_greet(java.lang.String)
  Then the message is "Moin Stephan!"    # de.openknowledge.projects.greet.GreetingCucumberSteps.then_the_message_is(java.lang.String)

[INFO] Tests run: 4, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.916 s - in de.openknowledge.projects.greet.GreetingCucumberIT
```

#### API Tests with Testcontainer and Postman/Newman

Postman is an popular API client that supports automated API testing. Test collections developed in Postman can be exported and integrated 
with your CI/CD pipeline by using [Newman](https://github.com/postmanlabs/newman), Postman's command line Collection Runner. 

Newman allows you to run and test a Postman Collection directly from the command line. It is built with extensibility in mind so that it 
can easily integrate it with continuous integration servers, build systems and even Testcontainers.

To automate Postman test collections with Testcontainers the newman docker image is required. The collection and the environment file has 
to be copied to the docker image, and a file system bind has to be configured, to be able to access the test reports.

**IMPORTANT**: The newman container is started and stopped for the execution of a single command - running the collection. To prevent that 
the containers is stopped before the test collection is executed, a `OneShotStartupCheckStrategy` with a timeout of 5 to 10 seconds has to 
be configured for the newman container.

GreetingPostmanIT - Newman container that runs a Postman collection against the containerized application.
```java
class GreetingPostmanIT extends AbstractIntegrationTest {

  private static final Logger LOG = LoggerFactory.getLogger(GreetResourceIT.class);

  private static final GenericContainer<?> NEWMAN = new GenericContainer<>("postman/newman:5.1.0-alpine")
      .withNetwork(NETWORK)
      .dependsOn(APPLICATION)
      .withCopyFileToContainer(MountableFile.forClasspathResource("postman/hello-world.postman_collection.json"),
                               "/etc/newman/hello-world.postman_collection.json")
      .withCopyFileToContainer(MountableFile.forClasspathResource("postman/hello-world.postman_environment.json"),
                               "/etc/newman/hello-world.postman_environment.json")
      .withFileSystemBind("target/postman/reports", "/etc/newman/reports", BindMode.READ_WRITE)
      .withStartupCheckStrategy(new OneShotStartupCheckStrategy().withTimeout(Duration.ofSeconds(5)));

  @Test
  void run() {
    NEWMAN.withCommand("run", "hello-world.postman_collection.json",
                       "--environment=hello-world.postman_environment.json",
                       "--reporters=cli,junit",
                       "--reporter-junit-export=reports/hello-world.newman-report.xml");
    NEWMAN.start();

    LOG.info(NEWMAN.getLogs());

    assertThat(NEWMAN.getCurrentContainerInfo().getState().getExitCode()).isZero();
  }
}
```