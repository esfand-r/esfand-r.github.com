---
layout: post
title: "Fun & Games in the Play Ground - Authentication and Authorization"
published: true
categories: articles
tags: 
  - Play Framework
  - Spring
  - Java
  - Scala
  - PlayAuthenticate
  - Mongo
  - Morphia
author: esfandiar_amirrahimi
comments: true
share: true
---



It’s been awhile I’ve wanted to have some fun with Play framework and get my hands dirty with technologies that live in or work with its ecosystem. In this article I’m going to tell the story of a small activator template I developed to learn and have fun during the process.

There are a few frameworks to help us with authentication in Play applications.  Usually a decent framework offers customizable abstractions over OAuth1, OAuth2, OpenID, Credentials, Basic Authentication, Two Factor Authentication or custom authentication schemes. [SecureSocial](http://securesocial.ws/), which has been available the longest, [Play Silhouette](https://github.com/mohiva/play-silhouette) and [Play Authenticate](http://joscha.github.io/play-authenticate/) all look like very capable frameworks for my story.

I decided to start with Play Authenticate and my starting point was a [great example of its usage](https://github.com/ntenisOT/play-authenticate-mongo). I will continue with experimenting with all the other frameworks to get a good feel of ease of use and capabilities in each one of the alternatives. The main reason for this choice was that, at first glance, it seemed to be easier to use with Java. While I have and will keep using both Java and Scala in this project, I will stick to Java for a lot of parts. I won’t get too much into the details of this decision.

> "Everyone on earth spoke the same language. As people migrated from the east, they settled in the land of Shinar. People there sought to make bricks and build a city and a tower with its top in the sky, to make a name for themselves, so that they not be scattered over the world. God came down to look at the city and tower, and remarked that as one people with one language, nothing that they sought would be out of their reach. God went down and confounded their speech, so that they could not understand each other, and scattered them over the face of the earth, and they stopped building the city. Thus the city was called Babel." [Tower of Babel](http://en.wikipedia.org/wiki/Tower_of_Babel)

Whether it was a god given destiny or a pure joy to appreciate contradictions of a polyglot world, I coudn’t stay away from it. I have been a lot more keen on Scala especially after picking up the great book of Debasish Ghosh [DSLs in Action](http://www.amazon.ca/DSLs-Action-Debasish-Ghosh/dp/1935182455/ref=sr_1_1?ie=UTF8&qid=1431472859&sr=8-1&keywords=dsl+in+action) , who also writes the fantastic blog  [Ruminations of a Programmer](http://debasishg.blogspot.ca/). So I’m definitely planning to learn it better and use it more widely and have started playing around with an idea of [a DSL](https://github.com/esfand-r/appdirect-client-dsl). 

Play Authenticate easily works with Deadbolt2, which gives you the ability to add authorization capabilities to a project and works well with both Java and Scala. It also seemed easy to - Configurable templates - Page and email templates were easily are configurable. You don't need to change the module code to achieve that. 

- Have multi language: We can even localize email templates.
- Have multi account support: This allows connection of many providers into one user account. For example a user can login using Github, Google and Twitter and join them as one person 

Overall, it is very easily customizable and we can easily hook up our own stuff and use our own [user services](https://github.com/esfand-r/Play2.3-Spring-PlayAuthenticate-deadbolt2-and-mongo-with-morphia/tree/master/modules/usermanagement/src/main/java/com/mycane/usermanagement). Out of the box, it supports major methods such as Basic Auth, Email/Pass, OpenID, Github, Google, Twitter, Facebook, etc and it is straightforward to add and plug in a new one.

## [The Sample](https://www.typesafe.com/activator/template/Play2.3-Spring-PlayAuthenticate-deadbolt2-and-mongo-with-morphia)
The developed [sample](https://github.com/esfand-r/Play2.3-Spring-PlayAuthenticate-deadbolt2-and-mongo-with-morphia) which is deployed on my [Rackspace ubuntu server](http://www.mycane.io) is a multi module play project made of:

- common
- securitycommon: This module was created since some of the core entities used in authentication needed to be shared across the modules.
- notification: Contains email services and deals with sending notifications and only emails at this point.
- usermanagement: Contains all the necessary user and account services.
- web: Contains controllers and all the front end elements of the webapp.

Notification and web are play applications and the rest are simple libraries. Notification is kept as a Play app since it uses some of the templating capabilities of play to generate email content dynamically based on the [notification type](https://github.com/esfand-r/Play2.3-Spring-PlayAuthenticate-deadbolt2-and-mongo-with-morphia/tree/master/modules/notification/app/com/mycane/notification/template). But I'm going to change that to have one Play App using a bunch of libraries soon. All these modules at the end get aggregated and build the [final webapp](https://github.com/esfand-r/Play2.3-Spring-PlayAuthenticate-deadbolt2-and-mongo-with-morphia/blob/master/build.sbt)

### Dependency Injection
There are a thousands of articles out there on benefits of DI and there are the ones that [call it evil](http://johannesbrodwall.com/2010/11/10/this-dependency-injection-madness-must-end/). I like it so I decided to use it. Using Spring was my first choice simply because I know it better. Spring is very well-tested, mature and has a lot of useful extras, if they float someone’s boat. I’ve been having mixed feeling here. On one hand, it can be very useful. For projects requiring a certain level of maturity and lots of functionality from a framework it could mean a certain marriage, if something like Dagger is more of a mistress.  But the thing is, if we let go and get too coupled with Spring, it could end in a very painful divorce. Spring supports more than dependency injection and provides aop right out of the box, which save us some boilerplate code. It also offers very nice documentation and thousands of examples are available online.

However, I kept using JSR-330 since it is a standard spec, and it’s supported on all J2ee. I wanted to be able to change whenever I wanted and use [Dagger](http://square.github.io/dagger/) and then Guice, which comes as default with [Play 2.4](https://www.playframework.com/documentation/2.4.x/JavaDependencyInjection). 

Dagger is super cool. It doesn’t really have any [dependency to anything](http://search.maven.org/#artifactdetails%7Ccom.squareup%7Cdagger%7C0.9.1%7Cjar) and it’s only 60 KB. This makes it a lot smaller than 694 KB of Guice, which interestingly has some light dependencies to [spring libraries](http://search.maven.org/#artifactdetails%7Ccom.google.inject%7Cguice%7C4.0%7Cjar), and a lot smaller than 1003 KB of spring-context.

Here is [a sample I created](https://www.typesafe.com/activator/template/play-java-dagger-dependency-injection) using Dagger for Dependency Injection in Playframework. 

For implementation using Dagger, instead of checking for Spring annotation [global](https://github.com/esfand-r/Play2.3-Spring-PlayAuthenticate-deadbolt2-and-mongo-with-morphia/blob/master/app/WebGlobal.java), we can do

 	@Override
 	public <A> A getControllerInstance(Class<A> controllerClass) throws Exception {
        return objectGraph.get(controllerClass);
 	}

Only thing left here is to change [configuration](https://github.com/esfand-r/Play2.3-Spring-PlayAuthenticate-deadbolt2-and-mongo-with-morphia/blob/master/modules/usermanagement/src/main/java/com/mycane/usermanagement/DBConfig.java)

So for example instead of using 

	@Configuration
	public class DBConfig {
	  	@Bean
	  	Morphia morphia() {
	     return new Morphia();
	  	}
	}

we could do

	@Module
	class DBConfig {
  		@Provides Morphia morphia() {
    		return new Morphia();
  		}
	}

And we can build our object graph in Global settings before starting the application.

  	public void beforeStart(Application app) {
      	super.beforeStart(app);
      	objectGraph = ObjectGraph.create(new ProductionModule());
  	}

Everything should be working the same as before... 

### Persistence
I have always enjoyed using MongoDB. Everything is easier until we get to transactions, which can be done but might not be as straight forward. I’m seriously considering playing around with [FoundationDB](https://foundationdb.com/) which is multimodel and promises the goodies of both worlds. 

I was thinking of using spring-mongo initially but at that point I would feel too dependent on it. Software development is kind of like a long war. It’s not a good idea to place all your bets on one strong ally. I have considered development using both Morphia and ReactiveMongo drivers and I will provide alternative implementation for both. But since for using ReactiveMongo we would have to go all Scala, it will be done in a separate sbt module providing the same implementation: this time using a non-blocking library. 

Other reasons to stick with Morphia include a handy out of the box support for DAOs. Experimenting with [QueryDSL](http://www.querydsl.com) for MongoDB was too good to miss; I took note of Morphia’s QueryDSL-compatibility. Using this library we can generate type-safe queries in a syntax similar to SQL. It currently has a wide range of support for various backends, though spread in separate modules including JPA, SQL, Lucene, Hibernate Search, MongoDB and etc. This will help to achieve consistency when introducing new technologies such as elasticsearch to the stack .

There is a [plugin](https://github.com/CedricGatay/play-querydsl/blob/master/library/src/main/scala/QueryDSLPlugin.scala) available for Play. But unfortunately it uses com.mysema.query.apt.jpa.JPAAnnotationProcessor explicitly. I ended up playing around with code generation using GenericExporter to generate query classes at compile time, but it’s not really done yet: [build.sbt](https://github.com/esfand-r/Play2.3-Spring-PlayAuthenticate-deadbolt2-and-mongo-with-morphia/blob/master/build.sbt)[QueryDSLClassGeneratorRun](https://github.com/esfand-r/Play2.3-Spring-PlayAuthenticate-deadbolt2-and-mongo-with-morphia/blob/master/project/QueryDSLClassGeneratorRun.scala) and [QueryDSLClassGenerator](https://github.com/esfand-r/Play2.3-Spring-PlayAuthenticate-deadbolt2-and-mongo-with-morphia/blob/master/modules/securitycommon/src/main/java/com/mycane/security/model/QueryDSLClassGenerator.java) 

It makes sense, maybe, to fork the current plugin and try to have it work with Morphia annotation processor in the future maybe, or maybe make a new one to handle different types of annotation processor… You could tell I’m a little uncertain here ;)

### Testing
This is where I decided not to go Java at all but keep using spring-test. Java sucks when it comes to testing. It’s simply too ugly. There is no problem configuring and using spring-test and wiring components in Scala tests: 
[MongoIntegrationTestBase](https://github.com/esfand-r/Play2.3-Spring-PlayAuthenticate-deadbolt2-and-mongo-with-morphia/blob/master/modules/usermanagement/src/test/scala/com/mycane/usermanagement/MongoIntegrationTestBase.scala), [TestConfig](https://github.com/esfand-r/Play2.3-Spring-PlayAuthenticate-deadbolt2-and-mongo-with-morphia/blob/master/modules/usermanagement/src/test/scala/com/mycane/usermanagement/TestConfig.java) and [UserServiceIntegrationTest](https://github.com/esfand-r/Play2.3-Spring-PlayAuthenticate-deadbolt2-and-mongo-with-morphia/blob/master/modules/usermanagement/src/test/scala/com/mycane/usermanagement/service/user/UserServiceIntegrationTest.scala)

[Flapdoodle](https://github.com/flapdoodle-oss/de.flapdoodle.embed.mongo) is and will be used in tests, so they won’t depend on a running instance of MongoDB.

### What’s next
I would like to try and implement the same app using other alternative authentication libraries, a test to find my ‘go to’ for writing future Play applications. Replacing the front-end with a single page app will be done also, even though making templates with Play are handy and pretty amazing. 

Also providing alternative implementation using ReactiveMongo is at works.

This all will be continued in a serie of upcoming articles ...
