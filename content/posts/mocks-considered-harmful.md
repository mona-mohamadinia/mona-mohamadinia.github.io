---
author:
  name: "Mona"
date: 2020-04-05
linktitle: mocks-considered-harmful
title: Mocks Considered Harmful
type:
- post
- posts
weight: 10
summary: "Even though Mocks, Spies, Fakes, and Stubs were part of our testing toolchain in the past, we should avoid using them in modern software development!"
images: 
- "/images/spy.jpg"
aliases:
- /blog/mocks-considered-harmful/
---
Even though *Mocks*, *Spies*, *Fakes*, and *Stubs* were part of our testing toolchain in the past, we should avoid using them in modern software development.

That may sound like a bold statement at first. However, I believe those ideas are backed by arguments that **simply are not valid anymore**. 

We might as well consider test doubles as our humble beginning to the testing world! But there are definitely better ways to do it these days. 

*Let's see how we got here and explore new ways!*

## Test Doubles<sup>*</sup>
---

Gerard Meszaros, in his seminal work *[xUnit Test Patterns](http://xunitpatterns.com/index.html)*, defined the *Test Double* concept as the following:

> *Test doubles are **test-specific equivalents** for components that our subject under test depends on.*

Software components don't live in isolation. As a result of these component dependencies, it may be hard to test one specific component without touching the others. To be more specific, sometimes:
 - We may not have access to other components in tests.
 - We may not be able to control other components of what they accept or return.
 - Other components may have unwanted side-effects.
 - Those components may make our tests to run slower.

Then he proposed a specific variation of test doubles to solve these issues or a subset of them. Let's get acquainted with those variations real quick.

### Stub
**Some components may provide some *indirect* inputs to our subject under test (SUT)**. To control those components and what they might return, we can use *Stubs*:
```kotlin
class UserService(private userRepository: UserRepository) {
    
    fun register(u: User) {
        val exists = userRepository.exists(u) // indirect input for SUT
        if (exists) throw UserAlreadyExistsException()

        // go ahead and register the user
    }
}
```
To stub out the `UserRepository` here, we can do something like:
```kotlin
@Test
fun `Register should throw an exception when the user already exists`() {
    val u = // some user
    stub {
        on { userRepository.exists(u) } doReturn true
    }

    // go examine and assert
}
```
Here we simply control what the `userRepository` returns to the SUT. 
### Mock/Spy
Mocks and Spies are just like Stubs with recording capabilities. Most mocking frameworks, however, consider these three concepts almost identical.

We can simply record what has been passed to the mocks/spies and verify them later on:
```kotlin
@Test
fun `Register should throw an exception when the user already exists`() {
    val passedUser = argumentCaptor<User>()
    stub {
        on { userRepository.exists(passedUser.capture()) } doReturn true
    }

    val u = // actual user
    userService.register(u)

    // verifying that u has been passed to the repository
    assertThat(passedUser.firstValue).isEqualTo(u)
}
```
Sometimes with spies, there is a real object and we just spying or stubbing specific methods of it. On the other hand, when using mocks, we usually mock the whole thing out!
### Fake
In the dark days, spinning up Oracle, Postgres, RabbitMQ, etc. instances in test environments was quite a hassle. In addition to this complexity, using *actual* instances of those *infrastructures* was incurring a significant overhead. **Hence the innovative concept of Slow Tests!**

Then we turned our attention towards *Fakes*. Fakes are usually providing the *same interface* as the actual components but with a more *lighter-weight implementation*. For instance, instead of using Postgres in our test environments, we used to run our tests using the likes of H2, HSQLDB, etc.

*This way, we managed to go through those dark days as our tests were running fast!*

### The Test Pyramid
Mike Cohn, in yet another seminal work *[Succeeding with Agile](https://www.amazon.com/Succeeding-Agile-Software-Development-Using/dp/0321579364)*, coined the term *Test Pyramid*:
{{< figure src="/images/pyramid.png" title="Test Pyramid" caption="Figure 1: Test Pyramid" >}}
Basically, the test pyramid is a visual depiction of different layers of testing with two fundamental properties:
 - There are more tests as we approach unit tests.
 - The bottom layer tests are faster.

This layered approach promotes the idea of *having tests in different levels of granularity*, which is a good thing. Also, it seems that we should have more fine-grained, i.e. unit, tests compared to those coarse-grained ones.

## Embracing Integration Tests
---
After a quick overview of a few ideas in the software testing literature, let's measure them with today's standards. It seems there are two fundamental motivations for using mocks and test doubles in general:
 - Faster execution time.
 - Better support for isolation and consequently, unit tests.

Nowadays, even our smartphones are capable of running some heavy stuff like Postgres! So, **the faster execution time is not a good excuse to mock everything out!**

Additionally, software components are mostly collaborating with each other to do something meaningful. Therefore, better support for isolated tests doesn't seem to be a good reason either.

Don't get the wrong idea here. Unit tests are sitting at a great level of granularity when we're going to test some algorithm or data structure or whatever that has a *meaningful identity in isolation*.

However, it's not a good idea to write unit tests for controllers, database interactions or anything else that does not play well with this level of granularity.

*The key takeaway here is writing [mostly integration tests](https://kentcdodds.com/blog/write-tests)*. Instead of thinking out of the box, *think upside down* this time around regarding the test pyramid!

## Tools of the Trade
---
As light-weight containers gained more popularity and ubiquity in recent years, the landscape of testing with real components improved dramatically.

### Modern CI/CD Environments

These days, most modern CI/CD environments run their stages in container-based runners. So they naturally support the idea of running other containers to facilitate integration tests:
 - Gitlab has a concept called *services* which provides a way to link another container with the container running the current job. The most common use-case for this is to run some infrastructural components for the sake of testing, e.g. [MySQL](https://docs.gitlab.com/ee/ci/services/mysql.html).
 - CircleCI also supports the idea of [service images](https://circleci.com/docs/2.0/circleci-images/#service-images).
 - Similarly, TravisCI provides a way to define these [servcies](https://docs.travis-ci.com/user/database-setup/).  

The bottom line is, if you're using any modern CI/CD environment, chances are that you can run a few extra docker containers for your tests.

### Test Containers
If we choose to rely on what our CI/CD of the choice offers, we are limited to run at least some of our integration tests *only during CI/CD stages*. Still better than mocking, but there are some rooms for improvements.

One such improvement is to use *[Test Containers](https://www.testcontainers.org)*. **Using testcontainers, we can spin up a container before running tests and tearing it down afterward**. For example, it's possible to write a simple fixture setup to spin a MySQL instance:
```kotlin
class MySqlExtension : BeforeAllCallback, AfterAllCallback {
 
    lateinit var container: MySQLContainer<*>
 
    override fun beforeAll(context: ExtensionContext?) {
        container = MySQLContainer<Nothing>("mysql:8")
        container.start()
    }
 
    override fun afterAll(context: ExtensionContext?) {
        container.stop()
    }
}
```
Then in our tests we can actually hit a real database and avoid mocking:
```kotlin
@ExtendWith(MySqlExtension::class)
class UserServiceIntegrationTest {

    // omitted

    @Test
    fun `Register should throw an exception when the user already exists`() {
        val u = // some user
        userRepository.save(u) // making it to already exist, no mocking!

        // go examine and assert
    }
}
```
**The good news is there are test containers ports for [Golang](https://github.com/testcontainers/testcontainers-go), [Rust](https://github.com/testcontainers/testcontainers-rs), [Python](https://github.com/testcontainers/testcontainers-python), [NodeJS](https://github.com/testcontainers/testcontainers-node), etc. so we can take advantage of this idea in different platforms.**
### External APIs
Calling external APIs, either local Microservices or other providers, is *inevitable* these days. There are some tools to facilitate wrting tests for these types of services, too. For example, [WireMock](http://wiremock.org/docs/) can be used to write more robust HTTP oriented tests.

Moreover, there are tools for writing tests based on the *Consumer Driven Contract* idea such as [Pact](https://docs.pact.io):
> Contract testing is a technique for testing an integration point by checking each application in isolation to ensure the messages it sends or receives conform to a shared understanding that is documented in a "contract".

## Conclusion
---
In this article we tried to convice you that *we can do much better than mocks!*. A few years ago, *DHH* talked about the same topic in [Test-induced design damage](https://dhh.dk/2014/test-induced-design-damage.html), which is highly recommended.

We discourage the use of any form of test doubles these days. However, we're not questioning the mentioned works in the article, specifically *xUnit Test Patterns* which is a masterpiece.

#### *footnotes*
<sup>*</sup> *Gerard Meszaros about the origins of test double: When the producers of a movie want to film something that is potentially risky or dangerous for the leading actor to carry out, they hire a "stunt double" to take the place of the actor in the scene. The stunt double is a highly trained individual who is capable of meeting the specific requirements of the scene. The stunt double may not be able to act, but he or she knows how to fall from great heights, crash a car, or do whatever the scene calls for. How closely the stunt double needs to resemble the actor depends on the nature of the scene. Usually, things can be arranged such that someone who vaguely resembles the actor in stature can take the actor's place.*
