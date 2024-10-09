---
title: "Project Loom: Structured Concurrency in Java"
date: 2024-10-10
header:
    image: "https://res.cloudinary.com/dkoypjlgr/image/upload/f_auto,q_auto:good,c_auto,w_1200,h_300,g_auto,fl_progressive/v1715952116/blog_cover_large_phe6ch.jpg"
tags: [java, loom, structured-concurrency]
excerpt: ""
toc: true
toc_label: "In this article"
---

_By [Riccardo Cardin](https://github.com/rcardin)_

TODO ForkJoinPool

Concurrency is a beast that every developer must face at some point in their career. It is a complex topic that requires a deep understanding of the underlying mechanisms of the programming language and the runtime environment. Java developers have been dealing with concurrency for a long time, and the Java platform provides a rich set of tools to manage it. However, writing concurrent code in Java is not an easy task, and developers often struggle to write correct and efficient concurrent programs. Fortunately, Project Loom is here to help. We already covered the introduction of virtual threads in the JVM in the article [The Ultimate Guide to Java Virtual Threads](https://blog.rockthejvm.com/ultimate-guide-to-java-virtual-threads/). In this article, we will explore the concept of structured concurrency and how Project Loom simplifies writing concurrent code in Java. 

## 1. Setting up the project

Java 23 was released in mid-September, and it brought a lot of new features and improvements to the Java platform. We're interested in the third preview of structured concurrency, which is given a dedicated JEP: [JEP 480: Structured Concurrency (Third Preview)](https://openjdk.org/jeps/480). As you can imagine, we need to enable the preview features with a dedicated directive to the compiler. We'll use Maven as the build tool for our project. So, to enable structured concurrency in our project, we need to add the following configuration to the `pom.xml` file:

```xml
<build>
  <pluginManagement>
    <plugin>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.10.1</version>
      <configuration>
        <release>23</release>
        <compilerArgs>--enable-preview</compilerArgs>
      </configuration>
    </plugin>
  </pluginManagement>
</build>
```

At the end of the article, we'll share the whole `pom.xml` file for your reference.

Once we have set up the project, we can start writing some code to explore structured concurrency in Java. In detail, we need an example to work with. This time, we'll take as an example the interaction with GitHub APIs (or at least, a simplified version of them). So, without further ado, let's start writing some code.

## 2. The problem with concurrency

We'll model the retrieval of the information needed to display the repositories of a GitHub user with some other data of the user itself. The final data structure we want to make available is the following:

```java
record GitHubUser(User user, List<Repository> repositories) {}

record User(UserId userId, UserName name, Email email) {}

record UserId(long value) {}

record UserName(String value) {}

record Email(String value) {}

record Repository(String name, Visibility visibility, URI uri) {}

enum Visibility {
    PUBLIC,
    PRIVATE
}
```

Unfortunately, Java still lacks of value classes, so we need to use regular types to model data. Once [project Valhalla](https://openjdk.org/projects/valhalla/) will be completed the `UserName`, `Email`, and `Repository` classes will be replaced with proper value classes.

We define also a logger that we'll use to print some useful information to the console:

```java
private static final Logger LOGGER = LoggerFactory.getLogger("GitHubApp");
```

How can we retrieve the needed information? Well, the GitHub API provides a RESTful interface that we can use to retrieve the information we need. In details, we need to perform two requests: one to retrieve the user information and one to retrieve the repositories of the user. From the first one, we retrieve the username and the email address, while from the second one, we retrieve the repositories of the user.

We don't need to implement the actual interaction with the GitHub API. So, we'll use a simplified version of the clients. Here is the interfaces we'll use:

```java
interface FindUserByIdPort {
  User findUser(UserId userId) throws InterruptedException;
}
interface FindRepositoriesByUserIdPort {
  List<Repository> findRepositories(UserId userId) throws InterruptedException;
}
```

The implementations of these interfaces are straightforward. They simply return fixed information and simulate a delay in the response. Here is the implementation of both the interfaces:

```java
static class GitHubRepository implements FindUserByIdPort, FindRepositoriesByUserIdPort {
  @Override
  public User findUser(UserId userId) throws InterruptedException {
    LOGGER.info("Finding user with id '{}'", userId);
    delay(Duration.ofMillis(500L));
    LOGGER.info("User '{}' found", userId);
    return new User(userId, new UserName("rcardin"), new Email("rcardin@rockthejvm.com"));
  }
  @Override
  public List<Repository> findRepositories(UserId userId) throws InterruptedException {
    LOGGER.info("Finding repositories for user with id '{}'", userId);
    delay(Duration.ofSeconds(1L));
    LOGGER.info("Repositories found for user '{}'", userId);
    return List.of(
        new Repository(
                "raise4s", Visibility.PUBLIC, URI.create("https://github.com/rcardin/raise4s")),
        new Repository(
                "sus4s", Visibility.PUBLIC, URI.create("https://github.com/rcardin/sus4s")));
  }
}

private static void delay(Duration duration) throws InterruptedException {
  Thread.sleep(duration);
}
```

We added a convenient `delay` method wrapping the common `Thread.sleep` method. 

As you might have noticed, we leave the `InterruptedException` unhandled, and we've not wrapped it into a `RuntimeException`. We did it on purpose. In fact, leaving the `InterruptedException` in the signature is a signal that the thread can suspend (or can be interrupted). We can call it a capability of the method. Although checked exceptions are not the best way to signal capabilities since they don't compose well, they're still the best approach in Java so far to implement an effect system in the language.

Now that we have our clients, we can start writing the code to retrieve the information we need. We'll start with the sequential version of the code.

```java
interface FindGitHubUserUseCase {
  GitHubUser findGitHubUser(UserId userId) throws InterruptedException;
}
static class FindGitHubUserSequentialService implements FindGitHubUserUseCase {
  private final FindUserByIdPort findUserByIdPort;
  private final FindRepositoriesByUserIdPort findRepositoriesByUserIdPort;
  public FindGitHubUserSequentialService(
      FindUserByIdPort findUserByIdPort,
      FindRepositoriesByUserIdPort findRepositoriesByUserIdPort) {
    this.findUserByIdPort = findUserByIdPort;
    this.findRepositoriesByUserIdPort = findRepositoriesByUserIdPort;
  }
  
  @Override
  public GitHubUser findGitHubUser(UserId userId) throws InterruptedException {
    var user = findUserByIdPort.findUser(userId);
    var repositories = findRepositoriesByUserIdPort.findRepositories(userId);
    return new GitHubUser(user, repositories);
  }
}
```

Despite all the ceremony needed by Java, the above code is quite simple. The tasks are performed in sequence, and we expect to wait at least 1.5 seconds to get the result. 

Although the code is not efficient, it has a lot of good features. First, it's easy to understand. Second, the computation has a clear scope. In fact, we know for sure that after the completion of the `findGitHubUser` method the computation is done and the JVM knows it can clean up all the resources used by the computation. Third, the exception handling management is straightforward. Let's say the `findUserByIdPort.findUser(userId)` call throws an exception. The exception is propagated to the caller, and the `findRepositoriesByUserIdPort.findRepositories(userId)` will never start. No resources are wasted in this case.

We can now write the concurrent version of the code since the call to the two external APIs have no dependency. Here is the code:

```java
@Override
public GitHubUser findGitHubUser(UserId userId)
    throws InterruptedException, ExecutionException {
  try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var user = executor.submit(() -> findUserByIdPort.findUser(userId));
    var repositories =
        executor.submit(() -> findRepositoriesByUserIdPort.findRepositories(userId));
    return new GitHubUser(user.get(), repositories.get());
  }
}
```

We're using virtual threads in the above solution. We guess everybody is familiar with them since they have been available for a while now. (However, if you're not familiar with virtual thread, please refer to our previous article [The Ultimate Guide to Java Virtual Threads](https://blog.rockthejvm.com/ultimate-guide-to-java-virtual-threads/)).

The code is quite simple. We use an `ExecutorService` to create a dedicated virtual thread for every submitted execution. The result is a `Future<T>` object that we can use to retrieve the result of the computation with the `get` method. We added the `ExecutionException` to our method signature because it's possible that we'll try to retrieve a value from a `Future` that has been completed exceptionally. Clearly, we need to add the exception also to the interface method's signature.

We can try to define a `main` method to test the code we've written so far. Here is the code:

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
  final GitHubRepository gitHubRepository = new GitHubRepository();
  FindGitHubUserConcurrentService service =
      new FindGitHubUserConcurrentService(gitHubRepository, gitHubRepository);
  final GitHubUser gitHubUser = service.findGitHubUser(new UserId(1L));
  LOGGER.info("GitHub user: {}", gitHubUser);
}
```

The output of the execution of the above code is the following:

```
08:50:34.159 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
08:50:34.159 [virtual-20] INFO GitHubApp -- Finding user with id 'UserId[value=1]'
08:50:34.679 [virtual-20] INFO GitHubApp -- User 'UserId[value=1]' found
08:50:35.179 [virtual-22] INFO GitHubApp -- Repositories found for user 'UserId[value=1]'
08:50:35.183 [main] INFO GitHubApp -- GitHub user: GitHubUser[user=User[userId=UserId[value=1], name=UserName[value=rcardin], email=Email[value=rcardin@rockthejvm.com]], repositories=[Repository[name=raise4s, visibility=PUBLIC, uri=https://github.com/rcardin/raise4s], Repository[name=sus4s, visibility=PUBLIC, uri=https://github.com/rcardin/sus4s]]]
```

The output is what we expect. Both thread starts together and their execution interleaves. After more or less 0.5 seconds, the user is found, and after more or less 1 second, the repositories are found. So far so good.

What if the `findUser` method throws an exception? Maybe the network has a glitch, or the GitHub API is down. Let's simulate it by throwing an exception in the `findUser` method. Here is the code:

```java
@Override
public User findUser(UserId userId) throws InterruptedException {
  LOGGER.info("Finding user with id '{}'", userId);
  delay(Duration.ofMillis(100L));
  throw new RuntimeException("Socket timeout");
}
```

We still want to wait a bit before throwing the exception to simulate a real-world scenario. We want to make sure that the `findRepositories` method is not started. We can run the `main` method again and see what happens. The output is the following:

```
08:39:41.945 [virtual-20] INFO GitHubApp -- Finding user with id 'UserId[value=1]'
08:39:41.947 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
08:39:42.969 [virtual-22] INFO GitHubApp -- Repositories found for user 'UserId[value=1]'
Exception in thread "main" java.util.concurrent.ExecutionException: java.lang.RuntimeException: Socket timeout
	at java.base/java.util.concurrent.FutureTask.report(FutureTask.java:122)
	at java.base/java.util.concurrent.FutureTask.get(FutureTask.java:191)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindGitHubUserConcurrentService.findGitHubUser(GitHubApp.java:115)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp.main(GitHubApp.java:125)
Caused by: java.lang.RuntimeException: Socket timeout
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$GitHubRepository.findUser(GitHubApp.java:68)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindGitHubUserConcurrentService.lambda$findGitHubUser$0(GitHubApp.java:112)
```

We should start to understand the problem with plain-old concurrency. The exception is thrown, but the `findRepositories` method is still started and executed till completion. This is a problem because we're wasting resources. We call such a situation as a thread leak. However, it can be worse. Imagine for example that the thread running the `ExecutorService` throws an exception after submitting the two computation. Here is the code:

```java
@Override
public GitHubUser findGitHubUser(UserId userId) throws InterruptedException, ExecutionException {
  try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var user = executor.submit(() -> findUserByIdPort.findUser(userId));
    var repositories =
            executor.submit(() -> findRepositoriesByUserIdPort.findRepositories(userId));
    throw new RuntimeException("Something went wrong"); 
  }
}
```

If we try to run again the `main` method, we'll see the following output:

```
08:52:18.455 [virtual-20] INFO GitHubApp -- Finding user with id 'UserId[value=1]'
08:52:18.455 [virtual-22] INFO GitHubApp -- Finding repositories for user with id 'UserId[value=1]'
08:52:18.975 [virtual-20] INFO GitHubApp -- User 'UserId[value=1]' found
08:52:19.476 [virtual-22] INFO GitHubApp -- Repositories found for user 'UserId[value=1]'
Exception in thread "main" java.lang.RuntimeException: Something went wrong
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp$FindGitHubUserConcurrentService.findGitHubUser(GitHubApp.java:115)
	at virtual.threads.playground/in.rcard.virtual.threads.GitHubApp.main(GitHubApp.java:136)
```

Even if the `main` thread went in error, both the executions of `findUser` and `findRepositories` methods didn't stop and complete their execution. Again, we found a thread leak. What we expected is that the `main` thread should be the root of the computation, the parent thread, and if it goes in error, all the computations started from it, aka its children thread, should stop. However, using plain-old concurrency, this is not the case. We can't express any association within threads.

We can try to implement our own some sort of policy to fix the above problem. However, it's not easy to do it. We need to keep track of all the threads started by the `ExecutorService` and stop them when the parent thread goes in error. It's a lot of work, and it's error-prone. We can't rely on the JVM to do it for us. We need to do it manually.

Fortunately, Project Loom provides a solution to the problem. It's called structured concurrency.

## 3. Structured concurrency

Now that we understood the concerns with what we called plain-old concurrency, let's step back to the sequential solution for a moment:

```java
@Override
public GitHubUser findGitHubUser(UserId userId) throws InterruptedException {
  var user = findUserByIdPort.findUser(userId);
  var repositories = findRepositoriesByUserIdPort.findRepositories(userId);
  return new GitHubUser(user, repositories);
}
```

As we said, despite being sequential and then not optimized, the above computation has some nice features. 

The computation has a clear scope, and the exception handling is straightforward. We can think about the calls to `findUserByIdPort.findUser(userId)` and `findRepositoriesByUserIdPort.findRepositories(userId)` methods as they were children of the `findGitHubUser` method. The language guarantees that when the `findGitHubUser` method completes, all its children computation are completed too. We cannot have children computation that outlive the parent one. 

We cannot have resource starvation or execution leaks neither. Again, it's the stack structure of modern programming languages that guarantees it. Every time a function terminates its execution, the runtime can clean up all the resources used by the computation. Moreover, if the `findUserByIdPort.findUser(userId)` method throws an exception, the `findRepositoriesByUserIdPort.findRepositories(userId)` method is not started.

We can say that the syntactic structure of the code reflects the semantic structure of the computation. We can say that the code is structured. 

Structured concurrency is a concurrency model that satisfies the above properties also in case of code that executes concurrently. It was introduced by Martin Sústrik in his blog post [Structured Concurrency](https://250bpm.com/blog:71/), and then popularized by Nathaniel J. Smith in his blog post [Notes on structured concurrency, or: Go statement considered harmful](https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/). 

At the core, its main objective is to provide a way to structure a concurrent computation in a way that the syntactic structure of the code reflects the semantic structure of the concurrency, satisfying the properties we've seen so far for the sequential code. We say that an operation can fork some children tasks to run concurrently. The relationships between the parent and the children tasks form a tree. 

![Structured Concurrency Tasks Tree](/images/loom-structured-concurrency/structured-concurrency.png)

The execution of the task in a node is guaranteed to be completed only when all the children tasks are completed. If a task throws an exception, all the sibling tasks are stopped, and the exception is propagated to the parent task.

In our example, once we'll set up the structured concurrency, the execution of the `findGitHubUser` method will be completed only when the `findUserByIdPort.findUser(userId)` and `findRepositoriesByUserIdPort.findRepositories(userId)` methods are completed.

Structured concurrency was already implemented in many programming languages. If we focused on the JVM, we can mention the [Kotlin Coroutines](https://blog.rockthejvm.com/kotlin-coroutines-101/), Scala [Cats Effects Fibers](https://typelevel.org/cats-effect/docs/concepts#structured-concurrency), and [ZIO Fibers](https://zio.dev/reference/fiber/#structured-concurrency). Now, Project Loom introduces structured concurrency also for Java.

Java implements structured concurrency though the `java.util.concurrent.StructuredTaskScope` type. Let's lean how to use it directly through coding. First, we need to create the scope into which we want the properties of structured concurrency to be satisfied. Here is the code:

```java
@Override
public GitHubUser findGitHubUser(UserId userId) throws ExecutionException {
  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    // TODO
    return null;
  }
}
```

We'll return on the `ShutdownOnFailure` type very soon. For now, we can say that it's one of the implementation of the `StructuredTaskScope` type, which is defined as follows:

```java
public class StructuredTaskScope<T> implements AutoCloseable
```

The type is a generic type `T` that represents the result of the computation. Moreover, it implements the `AutoCloseable` interface, which means we can use it inside a try-with-resources block. The try-with-resources block delimits the scope of the structured concurrency area. We'll see in a moment that the computation will leave the `try` block only when all the subtask created inside it are completed.

Now, we need to create the subtasks. Using the structured concurrency terminology we introduced so far, the thread creating the `StructuredTaskScope` is the parent task, and the subtasks are the children tasks. Here is the code:

```java
@Override
public GitHubUser findGitHubUser(UserId userId) throws ExecutionException {
  try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var user = scope.fork(() -> findUserByIdPort.findUser(userId));
    var repositories = scope.fork(() -> findRepositoriesByUserIdPort.findRepositories(userId));
    // TODO
    return null;
  }
}
```

As we can see, we can spawn or fork a new child task using the `fork` method of the `StructuredTaskScope` type, which is defined as follows:

```java
// Java SDK
public class StructuredTaskScope<T> implements AutoCloseable { 
  public <U extends T> Subtask<U> fork(Callable<? extends U> task)
}
```

The `fork` method takes a `Callable<T>` object as argument and returns a `Subtask<T>` object. The `Subtask<T>` object is a handle to the child task. To be fair, the `Subtask<T>` is defined as implementation of the `Supplier<T>` interface, which means we can retrieve the result of the computation using the `get` method:

```java
// Java SDK
public sealed interface Subtask<T> extends Supplier<T> permits SubtaskImpl
```

If we don't need to use any of the methods specific to the `Subtask<T>` type, and we only need to retrieve the result of the computation, we should use it as a `Supplier<T>` object. Java architects decided not to return a `java.util.concurrent.Future<T>` instance from the `fork` method to avoid confusion with the computations which are not structured, and give a clear cut with the past. 