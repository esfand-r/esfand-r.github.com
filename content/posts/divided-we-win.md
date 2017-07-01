+++
title = "divided we win"
description = ""
date = "2016-10-18T10:48:47-04:00"

+++



This is the republished version of the great article written by [Andriy Redko](http://aredko.blogspot.com/2015/08/divided-we-win-event-sourcing-cqrs.html).

In today's post we are going to unveil some very interesting (in my opinion) architecture styles: [event sourcing](http://martinfowler.com/eaaDev/EventSourcing.html) and [command query responsibility segregation](http://martinfowler.com/bliki/CQRS.html) (**CQRS**). Essentially, in both of them events are in the heart of the system design and reflect any changes of the state which are happening. It is quite different from the traditional [CRUD](https://en.wikipedia.org/wiki/Create,_read,_update_and_delete) architecture where usually only the last known state is kept, with any historical background effectively lost.

Another appealing option which [CQRS](http://martinfowler.com/bliki/CQRS.html) brings on the table is a natural separation of the write model (state modification initiated by commands and resulted in events) and read model (query for, in most case, last know state). It gives you a lot of freedom in picking the right data storage (or storages) for serving a different application use cases and demands. But, as always, there is no free lunch: the price to pay is the increased complexity of the system.

To keep the post reasonably short, the whole discussion is split into two parts: **Commands and Events** (this one), and **Queries** (upcoming). Using very simple model of the **user** entity as an example, we are going to design the application which uses [CQRS](http://martinfowler.com/bliki/CQRS.html) architecture. There are few libraries available on the JVM platform, but the one we are going to use is [Akka](http://akka.io/), more precisely its two recently added components [Akka Persistence](http://doc.akka.io/docs/akka/snapshot/scala/persistence.html) and [Akka HTTP](http://doc.akka.io/docs/akka-stream-and-http-experimental/current/scala.html). Please do not worry if [Akka](http://akka.io/) is somewhat new to you, the examples are going to be really basic and (hopefully) easy to follow. So let us get started!

As we mentioned earlier, the changes in our system are happening as the result of a command and may lead to zero or more events. Let us model that by defining two traits:

<pre class="brush: java;" name="code">trait Event
trait Command
</pre>

In turn, those events may trigger state changes within application entities so let us model that with a generic trait as well:

<pre class="brush: java;" name="code">trait State[T] {
  def updateState(event: Event): State[T]
}
</pre>

Quite simple so far. Now, as we are modeling a user, **UserAggregate** is going to constitute our model and be responsible for applying updates to a particular **User**, identifiable by its **id**. Also, every user is going to have an **email** property which could be changed upon receiving **UserEmailUpdate** command and could cause **UserEmailUpdated** event to be created as the result. The following code snippet defines all that in terms of **state**, **command** and **event** within **UserAggregate** companion object.

<pre class="brush: java;" name="code">object UserAggregate {
  case class User(id: String, email: String = "") extends State[User] {
    override def updateState(event: Event): State[User] = event match {
      case UserEmailUpdated(id, email) => copy(email = email)
    }
  }

  case class UserEmailUpdate(email: String) extends Command
  case class UserEmailUpdated(id: String, email: String) extends Event
}
</pre>

For the readers familiar with [CQRS](http://martinfowler.com/bliki/CQRS.html) and [event sourcing](http://martinfowler.com/eaaDev/EventSourcing.html), this example may look very naïve but I think it is good enough for grasping the basics.

Having our foundational blocks defined, it is a time to look on the most interesting part: accepting commands, transforming them into **persisted** events and applying the state changes, all of that are responsibilities of **UserAggregate**. In the context of [CQRS](http://martinfowler.com/bliki/CQRS.html) and [event sourcing](http://martinfowler.com/eaaDev/EventSourcing.html), persisting events is a crucial capability of the system. Events are the single source of truth and the state of any single entity could be reconstructed to any point in time by replaying all the events relevant to it. This is a moment were [Akka Persistence](http://doc.akka.io/docs/akka/snapshot/scala/persistence.html) is joining the stage.

To take one step back, [Akka](http://akka.io/) is a brilliant library (or even toolkit) to build distributed systems based on [actor model](https://en.wikipedia.org/wiki/Actor_model). On top of that, [Akka Persistence](http://doc.akka.io/docs/akka/snapshot/scala/persistence.html) enriches actors with persistence capabilities, letting them to store their messages (or events) in the durable journal. One might say that journal can grow very, very large and replaying all the events to reconstruct the state could take a lot of time. It is a valid point so [Akka Persistence](http://doc.akka.io/docs/akka/snapshot/scala/persistence.html) also adds the capability to store persistent snapshots in the durable storage as well. It speeds up the things a lot as only the events happening since the last snapshot should be replayed. By default, [LevelDB](https://github.com/google/leveldb) is a durable storage engine used by [Akka Persistence](http://doc.akka.io/docs/akka/snapshot/scala/persistence.html) but it is pluggable feature with many alternatives available.

With that, let us take a look on **UserAggregate** actor which is going to be responsible for managing a **User** entity.

<pre class="brush: java;" name="code">class UserAggregate(id: String) extends PersistentActor with ActorLogging {
  import UserAggregate._

  override def persistenceId = id
  var state: State[User] = User(id)

  def updateState(event: Event): Unit = {
    state = state.updateState(event)
  }

  val receiveCommand: Receive = {
    case UserEmailUpdate(email) => {
      persist(UserEmailUpdated(id, email)) { event =>
        updateState(event)
        sender ! Acknowledged(id)
      }
    }
  }

  override def receiveRecover: Receive = {
    case event: Event => updateState(event)
    case SnapshotOffer(_, snapshot: User) => state = snapshot
  }
}
</pre>

So far it is the most complicated part so let us dissect the key pieces. First of all, **UserAggregate** extends **PersistentActor**, which adds the persistent capabilities to it. Second, every persistent actor must have an unique **persistenceId**: it is used as an identifier inside events journal and snapshot storage. And lastly, in contrast to regular [Akka](http://akka.io/) actors, persistent actors do have two entry points: **receiveCommand** to process commands and **receiveRecover** to replay events.

Getting back to our example, once the **UserAggregate** receives **UserEmailUpdate** command, first of all it persists the **UserEmailUpdated** event in the journal using **persist(...)** call and then updates aggregate's state using **updateState(...)** call, replying with acknowledgement to the sender. To see the complete example in action, let us create a simple [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) endpoint using another great project from [Akka](http://akka.io/)'s family, [Akka HTTP](http://doc.akka.io/docs/akka-stream-and-http-experimental/current/scala.html), emerged from terrific [Spray](http://spray.io/) framework.

For now, we are going to define a simple route to handle a **PUT** request at **/api/v1/users/{id}** location, where **{id}** essentially is a placeholder for user identifier. It will accept a single form-encoded parameter **email** to update user's email address.

<pre class="brush: java;" name="code">object UserRoute {
  import scala.concurrent.ExecutionContext.Implicits.global
  import scala.language.postfixOps

  implicit val system = ActorSystem()
  implicit val materializer = ActorMaterializer()
  implicit val timeout: Timeout = 5 seconds

  val route = {
    logRequestResult("eventsourcing-example") {
      pathPrefix("api" / "v1" / "users") {
        path(LongNumber) { id =>
          (put & formFields('email.as[String])) { email =>
            complete {
              system
                .actorSelection(s"user/user-$id")
                .resolveOne
                .recover {
                  case _: ActorNotFound =>
                    system.actorOf(Props(new UserAggregate(id.toString)), s"user-$id")
                }
                .map {                 
                  _ ? UserEmailUpdate(email) map {
                    case Acknowledged(_) =>
                      HttpResponse(status = OK, entity = "Email updated: " + email)
                    case Error(_, message) =>
                      HttpResponse(status = Conflict, entity = message)
                  }
                }
            }
          }
        }
      }
    }
  }
}
</pre>

The only missed piece is the runnable class to plug the handler for this [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) endpoint so let us define one, thanks to [Akka HTTP](http://doc.akka.io/docs/akka-stream-and-http-experimental/current/scala.html) it is trivial:

<pre class="brush: java;" name="code">object Boot extends App with DefaultJsonProtocol {
  Http().bindAndHandle(route, "localhost", 38080)
}
</pre>

Nothing gives more assurance that the things are really working except an easy and readable test cases, built using [ScalaTest](http://www.scalatest.org/) framework complemented by [Akka HTTP TestKit](http://doc.akka.io/docs/akka-stream-and-http-experimental/current/scala.html).

<pre class="brush: java;" name="code">class UserRouteSpec extends FlatSpec
      with ScalatestRouteTest
      with Matchers
      with BeforeAndAfterAll {
  import com.example.domain.user.UserRoute

  implicit def executionContext = scala.concurrent.ExecutionContext.Implicits.global

  "UserRoute" should "return success on email update" in {
    Put("http://localhost:38080/api/v1/users/123", FormData("email" ->  "a@b.com")) ~> UserRoute.route ~> check {
      response.status shouldBe StatusCodes.OK
      responseAs[String] shouldBe "Email updated: a@b.com"
    }
  }
}
</pre>

Lastly, let us run the complete example and use **curl** from command line to perform a real call to the [REST](https://en.wikipedia.org/wiki/Representational_state_transfer) endpoint, forcing all the parts of the application to work together.

<pre>$ curl -X PUT http://localhost:38080/api/v1/users/123 -d email=a@b.com
Email updated: a@b.com
</pre>

Nice! The results are matching our expectations. Before we finish up with this part, there is one more thing: please notice that when application is restarted, the state will be restored for the users for which aggregates did already exist. Another way to say it using [Akka Persistence](http://doc.akka.io/docs/akka/snapshot/scala/persistence.html) specifics, if event journal has events/snapshots associated with actor's **persistenceId** (here is why it should be unique and permanent), the recovery process takes place by applying more recent snapshot (if any) and replaying events.

In this post we looked behind the curtain of [event sourcing](http://martinfowler.com/eaaDev/EventSourcing.html) and [CQRS](http://martinfowler.com/bliki/CQRS.html), covering only the write (or command) part of the architecture. There are many things to work on like for example, email uniqueness validation, user latest state retrieval, ... which represent the read (or query) flow of the application and are going to be discussed in the consequent blog post. As for now, I hope that [event sourcing](http://martinfowler.com/eaaDev/EventSourcing.html) and [CQRS](http://martinfowler.com/bliki/CQRS.html) are a bit demystified and might become a part of your future or existing applications.

**Final disclaimer:** [Akka HTTP](http://doc.akka.io/docs/akka-stream-and-http-experimental/current/scala.html) is marked as **experimental** component at the moment (nonetheless the main goals stand still, the API may change slightly) while [Akka Persistence](http://doc.akka.io/docs/akka/snapshot/scala/persistence.html) just graduated out of experimental status with recent [Akka](http://akka.io/)'s **2.4.0-RC1** release. Please be aware of that.
