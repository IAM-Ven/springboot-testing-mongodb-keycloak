# `springboot-testing-mongodb-keycloak`

The goals of this project are:

- Create a REST API application that manages books, `book-service`. The data is stored in [`MongoDB`](https://www.mongodb.com) database.
The application will have its sensitive endpoints (add/update/delete book) secured.
- Use [`Keycloak`](https://www.keycloak.org) as authentication and authorization server;
- Explore the utilities and annotations that Spring Boot provides when testing applications.

# Start Environment

***Note. In order to run some commands/scripts, you must have [`jq`](https://stedolan.github.io/jq) installed on you machine***

## Docker Compose

1. Open one terminal

2. Inside `/springboot-testing-mongodb-keycloak` root folder run
```
docker-compose up -d
```
> To stop and remove containers, networks and volumes
> ```
> docker-compose down -v
> ```

## Configure Keycloak

You have two options to configure `Keycloak`: running `init-keycloak.sh` script or manually using `Keycloak UI`.

### Automatically running script

1. Inside `/springboot-testing-mongodb-keycloak` root folder, un the following script to initialize `Keycloak`.
```
./init-keycloak.sh
```
It will create automatically the `company-services` realm, `book-service` client, `manage_books` client role and the
user `ivan.franchin` with the role `manage_books` assigned.

3. `BOOKSERVICE_CLIENT_SECRET` is shown in the end of the script outputs. It will be needed whenever we call `Keycloak`
to get token for `book-service` application.
```
BOOKSERVICE_CLIENT_SECRET=...
```

### Manually using Keycloak UI

![keycloak](images/keycloak.png)

1. Access the link: http://localhost:8181/auth/admin/master/console

2. Login with the credentials
```
Username: admin
Password: admin
```

3. Create a new Realm
- Go to top-left corner and hover the mouse over `Master` realm. A blue button `Add realm` will appear. Click on it.
- On `Name` field, write `company-services`. Click on `Create`.

4. Create a new Client
- Click on `Clients` menu on the left.
- Click `Create` button.
- On `Client ID` field type `book-service`.
- Click on `Save`.
- On `Settings` tab, set the `Access Type` to `confidential`.
- Still on `Settings` tab, set the `Valid Redirect URIs` to `http://localhost:8080/*`.
- Click on `Save`.
- Go to `Credentials` tab. Copy the value on `Secret` field. It will be used on the next steps.
- Go to `Roles` tab.
- Click `Add Role` button.
- On `Role Name` type `manage_books`.
- Click on `Save`.

5. Create a new User
- Click on `Users` menu on the left.
- Click on `Add User` button.
- On `Username` field set `ivan.franchin`.
- Click on `Save`.
- Go to `Credentials` tab.
- Set to `New Password` and `Password Confirmation` the value `123`.
- Turn off the `Temporary` field.
- Click on `Reset password`.
- Confirm the pop up clicking on `Change Password`.
- Go to `Role Mappings` tab.
- Select `book-service` on the combo-box `Client Roles`.
- Add the role `manage_books` to `ivan.franchin`.

**Done!** That is all the configuration needed on `Keycloak`. 

## Start book-service

1. Open a new terminal

2. Start `book-service` application

In `springboot-testing-mongodb-keycloak` root folder, run:
```
./gradlew clean bootRun
```

# Test using cURL

1. Open a new terminal

2. Call the endpoint `GET /api/books` using the cURL command bellow.
```
curl -i http://localhost:8080/api/books
```

It will return:
```
HTTP/1.1 200
[]
```

3. Try to call the endpoint `POST /api/books` using the cURL command bellow.
``` 
curl -i -X POST http://localhost:8080/api/books \
  -H "Content-Type: application/json" \
  -d '{ "authorName": "ivan", "title": "java 8", "price": 10.5 }'
```
It will return:
```
HTTP/1.1 302
```

Here, the application is trying to redirect the request to an authentication link.

4. Export to `BOOKSERVICE_CLIENT_SECRET` environment variable, the `Secret` value generated by `Keycloak` to `book-service`
application. See *Configure Keycloak* section.
```
export BOOKSERVICE_CLIENT_SECRET=...
```

5. Run the commands bellow to get an access token for `ivan.franchin` user.

```
MY_ACCESS_TOKEN=$(curl -s -X POST \
  http://localhost:8181/auth/realms/company-services/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=ivan.franchin" \
  -d "password=123" \
  -d "grant_type=password" \
  -d "client_secret=$BOOKSERVICE_CLIENT_SECRET" \
  -d "client_id=book-service" | jq -r .access_token)
```

6. Call the endpoint `POST /api/books` using the cURL command bellow.
```
curl -i -X POST http://localhost:8080/api/books \
  -H "Authorization: Bearer $MY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "authorName": "ivan", "title": "java 8", "price": 10.5 }'
```

It will return:

```
HTTP/1.1 201
{
  "id":"01d984be-26bc-49f5-a201-602293d62b82",
  "authorName":"ivan",
  "title":"java 8",
  "price":10.5
}
```

# Test using Swagger

![swagger](images/swagger.png)

1. Access the link: http://localhost:8080/swagger-ui.html

2. Click on `GET /api/books` to open it. Then, click on `Try it out` button and, finally, click on `Execute` button.
It will return a status code `200` and an empty list or a list with some books if you already add them.

3. Now click on `POST /api/books`, it is a secured endpoint. Let's try it without authentication.

4. Click on `Try it out` button (you can use the default values) and then on `Execute` button
It will return:
```
TypeError: Failed to fetch
```

5. In order to access the private endpoint, you need an access token. To get it, run the following commands in a terminal.
```
MY_ACCESS_TOKEN=$(curl -s -X POST \
  http://localhost:8181/auth/realms/company-services/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=ivan.franchin" \
  -d "password=123" \
  -d "grant_type=password" \
  -d "client_secret=$BOOKSERVICE_CLIENT_SECRET" \
  -d "client_id=book-service" | jq -r .access_token)
  
echo "Bearer $MY_ACCESS_TOKEN" 
```

6. Copy (`Ctr-C`) the token generated (something like that starts with `Bearer ...`) and go back to `Swagger`.

7. Click on the `Authorize` button, paste (`Ctr-V`) the copied access token in the value field. Then, click on `Authorize` and, to finalize, click on `Close`.

8. Go to `POST /api/books`, click on `Try it out` and then on `Execute` button
It will return:
```
HTTP/1.1 201
{
  "id": "5cf212c3-7902-4141-968b-82ae7a3443f1",
  "authorName": "Craig Walls",
  "title": "Spring Boot",
  "price": 10.5
}
```

9. The access token default expiration period is `5 minutes`. So, wait for this time and, using the same access token,
try to call the private endpoint. It will return:
```
HTTP/1.1 401 
WWW-Authenticate: Bearer realm="company-services", error="invalid_token", error_description="Token is not active"
```

# Useful Links

### jwt.io

With [jwt.io](https://jwt.io) you can inform the JWT token you have received from Keycloak and the online tool decodes
the token, showing its header and payload.

## Running unit and integration testing

1. In order to run unit and integration testing type
```
./gradlew test integrationTest
```

2. From `springboot-testing-mongodb-keycloak` root folder, unit testing report can be found in
```
/build/reports/tests/test/index.html
```

3. From `springboot-testing-mongodb-keycloak` root folder, integration testing report can be found in
```
/build/reports/tests/integrationTest/index.html
```

# More about testing Spring Boot Applications

Spring Boot provides a number of utilities and annotations to help when testing your application.

## Unit Testing

The idea of the unit testing is to test each layer of the application (repository, service and controller) individually.
The repository classes usually don't depends on any other classes, so we can write test cases without any mocking.
On the other hand, the services classes depend on repositories. So, as we have already test cases to cover the repositories, while writing test cases for the services we don't need to care about the quality of the repositories classes. So, every calls to repositories classes should be mocked.
The same happens to controller classes that depends on the services classes. While writing tests for the controllers, service calls on the controller classes should be mocked.

### Repository Testing

You can use `@DataMongoTest` to test `MongoDB` applications.
By default, it configures an in-memory embedded `MongoDB` (if available), configures a `MongoTemplate`, scans for `@Document` classes, and configures Spring Data `MongoDB` repositories.
The embedded `MongoDB` added in the `build.gradle` and can by found in the link: https://github.com/flapdoodle-oss/de.flapdoodle.embed.mongo
Regular `@Component` beans are not loaded into the `ApplicationContext`.
An example of utilization is:

```
@ExtendWith(SpringExtension.class)
@DataMongoTest
public class BookRepositoryTests {

    @Autowired
    private MongoTemplate mongoTemplate;

	@Autowired
	private BookRepository bookRepository;

	// Tests ..
}
```

### Service Testing

In order to test the application services, we can use a something similar as shown bellow, as we create an instance of `BookServiceImpl` and mock the `bookRepository` 

```
@ExtendWith(SpringExtension.class)
public class BookServiceImplTest {

    private BookService bookService;
    private BookRepository bookRepository;

    @Before
    public void setUp() {
        bookRepository = mock(BookRepository.class);
        bookService = new BookServiceImpl(bookRepository);
    }
    
    // Tests
}
```

### Controller Testing

`@WebMvcTest` annotation can be used to test whether Spring MVC controllers are working as expected.
`@WebMvcTest` is limited to a single controller and is used in combination with `@MockBean` to provide mock implementations for required dependencies.
`@WebMvcTest` also auto-configures `MockMvc`. Mock MVC offers a powerful way to quickly test MVC controllers without needing to start a full HTTP server.
In the example bellow, you can see that we mocking the services (in this case `bookService`) used by `BookController`.
The annotation `@AutoConfigureMockMvc(secure = false)` is used to disable security configuration. 

```
@ExtendWith(SpringExtension.class)
@WebMvcTest(BookController.class)
@AutoConfigureMockMvc(secure = false)
public class BookControllerTests {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private BookService bookService;
    
    // Tests ... 
}
```

### DTO Testing

`@JsonTest` annotation can be used to test whether object JSON serialization and deserialization is working as expected.
In the example bellow, it is used `JacksonTester`. However, `GsonTester`, `JsonbTester` and `BasicJsonTester` could also be used instead.
Btw, I've tried to use all of them, but just `JacksonTester` worked easily and as expected.  

```
@ExtendWith(SpringExtension.class)
@JsonTest
public class MyJsonTests {

	@Autowired
	private JacksonTester<VehicleDetails> json;

	// Tests ...
}
```

## Integration Testing

The main goal of the integration tests is, as its name suggests, to integrate the different layers of the application. Here, no mocking is involved and a full running HTTP server is needed. 
So, in order to have it, we can use the `@SpringBootTest(webEnvironment=WebEnvironment.RANDOM_PORT)` annotation. What this annotation does is to start a full running server running in a random ports. Spring Boot also provides a `TestRestTemplate` facility, for example:

```
@ExtendWith(SpringExtension.class)
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class RandomPortTestRestTemplateExampleTests {

	@Autowired
	private TestRestTemplate restTemplate;

	// Tests ...

}
```

Integration tests should run separated from the unit tests and, mainly, it should runs after unit tests. In this project, we created a new integrationTest Gradle task to handle exclusively integration tests.

# References

- https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html
- http://www.baeldung.com/spring-boot-testing

# Issues

Unable to update `Gradle Wrapper` to `4.10.3` (see **LOG-1**). But, with `spring boot` version `2.1.3` it doesn't
complain. However, updating `spring boot` to version `2.1.3` the exception show on **LOG-2** is thrown. There is this
work around (https://stackoverflow.com/questions/53318134/unable-to-use-keycloak-in-spring-boot-2-1-due-to-duplicated-bean-registration-ht)
that starts the application normally. However, it breaks the test cases present in `BookControllerTest`.

#### LOG-1
```
/springboot-testing-mongodb-keycloak/src/main/java/com/mycompany/bookservice/dto/UpdateBookDto.java:13: warning: lombok.javac.apt.LombokProcessor could not be initialized. Lombok will not run during this compilation: java.lang.ClassCastException: org.gradle.api.internal.tasks.compile.processing.IncrementalFiler cannot be cast to com.sun.tools.javac.processing.JavacFiler
public class UpdateBookDto {
       ^
        at lombok.javac.apt.LombokProcessor.getJavacFiler(LombokProcessor.java:447)
        at lombok.javac.apt.LombokProcessor.init(LombokProcessor.java:90)
        at lombok.core.AnnotationProcessor$JavacDescriptor.want(AnnotationProcessor.java:112)
        at lombok.core.AnnotationProcessor.init(AnnotationProcessor.java:165)
        at lombok.launch.AnnotationProcessorHider$AnnotationProcessor.init(AnnotationProcessor.java:73)
        at org.gradle.api.internal.tasks.compile.processing.DelegatingProcessor.init(DelegatingProcessor.java:57)
        at org.gradle.api.internal.tasks.compile.processing.IsolatingProcessor.init(IsolatingProcessor.java:44)
        at com.sun.tools.javac.processing.JavacProcessingEnvironment$ProcessorState.<init>(JavacProcessingEnvironment.java:500)
        at com.sun.tools.javac.processing.JavacProcessingEnvironment$DiscoveredProcessors$ProcessorStateIterator.next(JavacProcessingEnvironment.java:597)
        at com.sun.tools.javac.processing.JavacProcessingEnvironment.discoverAndRunProcs(JavacProcessingEnvironment.java:690)
        at com.sun.tools.javac.processing.JavacProcessingEnvironment.access$1800(JavacProcessingEnvironment.java:91)
        at com.sun.tools.javac.processing.JavacProcessingEnvironment$Round.run(JavacProcessingEnvironment.java:1035)
        at com.sun.tools.javac.processing.JavacProcessingEnvironment.doProcessing(JavacProcessingEnvironment.java:1176)
        at com.sun.tools.javac.main.JavaCompiler.processAnnotations(JavaCompiler.java:1170)
        at com.sun.tools.javac.main.JavaCompiler.compile(JavaCompiler.java:856)
        at com.sun.tools.javac.main.Main.compile(Main.java:523)
        at com.sun.tools.javac.api.JavacTaskImpl.doCall(JavacTaskImpl.java:129)
        at com.sun.tools.javac.api.JavacTaskImpl.call(JavacTaskImpl.java:138)
        at org.gradle.api.internal.tasks.compile.AnnotationProcessingCompileTask.call(AnnotationProcessingCompileTask.java:89)
        at org.gradle.api.internal.tasks.compile.ResourceCleaningCompilationTask.call(ResourceCleaningCompilationTask.java:57)
        at org.gradle.api.internal.tasks.compile.JdkJavaCompiler.execute(JdkJavaCompiler.java:50)
        at org.gradle.api.internal.tasks.compile.JdkJavaCompiler.execute(JdkJavaCompiler.java:36)
        at org.gradle.api.internal.tasks.compile.NormalizingJavaCompiler.delegateAndHandleErrors(NormalizingJavaCompiler.java:100)
        at org.gradle.api.internal.tasks.compile.NormalizingJavaCompiler.execute(NormalizingJavaCompiler.java:52)
        at org.gradle.api.internal.tasks.compile.NormalizingJavaCompiler.execute(NormalizingJavaCompiler.java:38)
        at org.gradle.api.internal.tasks.compile.AnnotationProcessorDiscoveringCompiler.execute(AnnotationProcessorDiscoveringCompiler.java:49)
        at org.gradle.api.internal.tasks.compile.AnnotationProcessorDiscoveringCompiler.execute(AnnotationProcessorDiscoveringCompiler.java:35)
        at org.gradle.api.internal.tasks.compile.CleaningJavaCompilerSupport.execute(CleaningJavaCompilerSupport.java:39)
        at org.gradle.api.internal.tasks.compile.incremental.IncrementalCompilerFactory$2.execute(IncrementalCompilerFactory.java:110)
        at org.gradle.api.internal.tasks.compile.incremental.IncrementalCompilerFactory$2.execute(IncrementalCompilerFactory.java:106)
        at org.gradle.api.internal.tasks.compile.incremental.IncrementalResultStoringCompiler.execute(IncrementalResultStoringCompiler.java:59)
        at org.gradle.api.internal.tasks.compile.incremental.IncrementalResultStoringCompiler.execute(IncrementalResultStoringCompiler.java:43)
        at org.gradle.api.tasks.compile.JavaCompile.performCompilation(JavaCompile.java:153)
        at org.gradle.api.tasks.compile.JavaCompile.compile(JavaCompile.java:121)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.gradle.internal.reflect.JavaMethod.invoke(JavaMethod.java:73)
        at org.gradle.api.internal.project.taskfactory.IncrementalTaskAction.doExecute(IncrementalTaskAction.java:50)
        at org.gradle.api.internal.project.taskfactory.StandardTaskAction.execute(StandardTaskAction.java:39)
        at org.gradle.api.internal.project.taskfactory.StandardTaskAction.execute(StandardTaskAction.java:26)
        at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter$1.run(ExecuteActionsTaskExecuter.java:131)
        at org.gradle.internal.operations.DefaultBuildOperationExecutor$RunnableBuildOperationWorker.execute(DefaultBuildOperationExecutor.java:301)
        at org.gradle.internal.operations.DefaultBuildOperationExecutor$RunnableBuildOperationWorker.execute(DefaultBuildOperationExecutor.java:293)
        at org.gradle.internal.operations.DefaultBuildOperationExecutor.execute(DefaultBuildOperationExecutor.java:175)
        at org.gradle.internal.operations.DefaultBuildOperationExecutor.run(DefaultBuildOperationExecutor.java:91)
        at org.gradle.internal.operations.DelegatingBuildOperationExecutor.run(DelegatingBuildOperationExecutor.java:31)
        at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter.executeAction(ExecuteActionsTaskExecuter.java:120)
        at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter.executeActions(ExecuteActionsTaskExecuter.java:99)
        at org.gradle.api.internal.tasks.execution.ExecuteActionsTaskExecuter.execute(ExecuteActionsTaskExecuter.java:77)
        at org.gradle.api.internal.tasks.execution.OutputDirectoryCreatingTaskExecuter.execute(OutputDirectoryCreatingTaskExecuter.java:51)
        at org.gradle.api.internal.tasks.execution.SkipUpToDateTaskExecuter.execute(SkipUpToDateTaskExecuter.java:59)
        at org.gradle.api.internal.tasks.execution.ResolveTaskOutputCachingStateExecuter.execute(ResolveTaskOutputCachingStateExecuter.java:54)
        at org.gradle.api.internal.tasks.execution.ValidatingTaskExecuter.execute(ValidatingTaskExecuter.java:59)
        at org.gradle.api.internal.tasks.execution.SkipEmptySourceFilesTaskExecuter.execute(SkipEmptySourceFilesTaskExecuter.java:101)
        at org.gradle.api.internal.tasks.execution.FinalizeInputFilePropertiesTaskExecuter.execute(FinalizeInputFilePropertiesTaskExecuter.java:44)
        at org.gradle.api.internal.tasks.execution.CleanupStaleOutputsExecuter.execute(CleanupStaleOutputsExecuter.java:91)
        at org.gradle.api.internal.tasks.execution.ResolveTaskArtifactStateTaskExecuter.execute(ResolveTaskArtifactStateTaskExecuter.java:62)
        at org.gradle.api.internal.tasks.execution.SkipTaskWithNoActionsExecuter.execute(SkipTaskWithNoActionsExecuter.java:59)
        at org.gradle.api.internal.tasks.execution.SkipOnlyIfTaskExecuter.execute(SkipOnlyIfTaskExecuter.java:54)
        at org.gradle.api.internal.tasks.execution.ExecuteAtMostOnceTaskExecuter.execute(ExecuteAtMostOnceTaskExecuter.java:43)
        at org.gradle.api.internal.tasks.execution.CatchExceptionTaskExecuter.execute(CatchExceptionTaskExecuter.java:34)
        at org.gradle.api.internal.tasks.execution.EventFiringTaskExecuter$1.run(EventFiringTaskExecuter.java:51)
        at org.gradle.internal.operations.DefaultBuildOperationExecutor$RunnableBuildOperationWorker.execute(DefaultBuildOperationExecutor.java:301)
        at org.gradle.internal.operations.DefaultBuildOperationExecutor$RunnableBuildOperationWorker.execute(DefaultBuildOperationExecutor.java:293)
        at org.gradle.internal.operations.DefaultBuildOperationExecutor.execute(DefaultBuildOperationExecutor.java:175)
        at org.gradle.internal.operations.DefaultBuildOperationExecutor.run(DefaultBuildOperationExecutor.java:91)
        at org.gradle.internal.operations.DelegatingBuildOperationExecutor.run(DelegatingBuildOperationExecutor.java:31)
        at org.gradle.api.internal.tasks.execution.EventFiringTaskExecuter.execute(EventFiringTaskExecuter.java:46)
        at org.gradle.execution.taskgraph.LocalTaskInfoExecutor.execute(LocalTaskInfoExecutor.java:42)
        at org.gradle.execution.taskgraph.DefaultTaskExecutionGraph$BuildOperationAwareWorkItemExecutor.execute(DefaultTaskExecutionGraph.java:277)
        at org.gradle.execution.taskgraph.DefaultTaskExecutionGraph$BuildOperationAwareWorkItemExecutor.execute(DefaultTaskExecutionGraph.java:262)
        at org.gradle.execution.taskgraph.DefaultTaskPlanExecutor$ExecutorWorker$1.execute(DefaultTaskPlanExecutor.java:135)
        at org.gradle.execution.taskgraph.DefaultTaskPlanExecutor$ExecutorWorker$1.execute(DefaultTaskPlanExecutor.java:130)
        at org.gradle.execution.taskgraph.DefaultTaskPlanExecutor$ExecutorWorker.execute(DefaultTaskPlanExecutor.java:200)
        at org.gradle.execution.taskgraph.DefaultTaskPlanExecutor$ExecutorWorker.executeWithWork(DefaultTaskPlanExecutor.java:191)
        at org.gradle.execution.taskgraph.DefaultTaskPlanExecutor$ExecutorWorker.run(DefaultTaskPlanExecutor.java:130)
        at org.gradle.internal.concurrent.ExecutorPolicy$CatchAndRecordFailures.onExecute(ExecutorPolicy.java:63)
        at org.gradle.internal.concurrent.ManagedExecutorImpl$1.run(ManagedExecutorImpl.java:46)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at org.gradle.internal.concurrent.ThreadFactoryImpl$ManagedThreadRunnable.run(ThreadFactoryImpl.java:55)
        at java.lang.Thread.run(Thread.java:745)

```

#### LOG-2
```
2019-02-19 21:39:59.162  WARN 6843 --- [           main] ConfigServletWebServerApplicationContext : Exception encountered during context initialization - cancelling refresh attempt: org.springframework.beans.factory.support.BeanDefinitionOverrideException: Invalid bean definition with name 'httpSessionManager' defined in class path resource [com/mycompany/bookservice/config/KeycloakSecurityConfig.class]: Cannot register bean definition [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=keycloakSecurityConfig; factoryMethodName=httpSessionManager; initMethodName=null; destroyMethodName=(inferred); defined in class path resource [com/mycompany/bookservice/config/KeycloakSecurityConfig.class]] for bean 'httpSessionManager': There is already [Generic bean: class [org.keycloak.adapters.springsecurity.management.HttpSessionManager]; scope=singleton; abstract=false; lazyInit=false; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=null; initMethodName=null; destroyMethodName=null; defined in URL [jar:file:/Users/ivanfranchin/.gradle/caches/modules-2/files-2.1/org.keycloak/keycloak-spring-security-adapter/4.8.1.Final/eda5d5c8c11562c6e07943e61bdfde9d051e3fd3/keycloak-spring-security-adapter-4.8.1.Final.jar!/org/keycloak/adapters/springsecurity/management/HttpSessionManager.class]] bound.
2019-02-19 21:39:59.194  INFO 6843 --- [           main] ConditionEvaluationReportLoggingListener : 

Error starting ApplicationContext. To display the conditions report re-run your application with 'debug' enabled.
2019-02-19 21:39:59.198 ERROR 6843 --- [           main] o.s.b.d.LoggingFailureAnalysisReporter   : 

***************************
APPLICATION FAILED TO START
***************************

Description:

The bean 'httpSessionManager', defined in class path resource [com/mycompany/bookservice/config/KeycloakSecurityConfig.class], could not be registered. A bean with that name has already been defined in URL [jar:file:/Users/ivanfranchin/.gradle/caches/modules-2/files-2.1/org.keycloak/keycloak-spring-security-adapter/4.8.1.Final/eda5d5c8c11562c6e07943e61bdfde9d051e3fd3/keycloak-spring-security-adapter-4.8.1.Final.jar!/org/keycloak/adapters/springsecurity/management/HttpSessionManager.class] and overriding is disabled.

Action:

Consider renaming one of the beans or enabling overriding by setting spring.main.allow-bean-definition-overriding=true
```