## Purpose

You use want to use Json [jto-validation](https://github.com/jto/validation) in conjunction with [akka-http](http://doc.akka.io/docs/akka/2.4.11/scala/http/introduction.html).
Here is a small set of missing directives and marshallers/unmarshallers to make it easy.

Feel free to contribute.

## Install

Add the project to your sbt project dependencies :

```
resolvers += Resolver.bintrayRepo("taraxe", "maven")

libraryDependencies ++= Seq(
    "com.taraxe" %% "akka-http-json-jto-validation" % "0.1.0"
)
```


## Usage

```
import akka.actor.ActorSystem
import akka.http.scaladsl.model.StatusCodes._
import akka.http.scaladsl.server.Directives
import akka.stream.Materializer
import com.taraxe.JsonJtoValidationSupport
import jto.validation._
import play.api.libs.json._
import scala.concurrent.ExecutionContextExecutor

case class Message(foo:String, bar:String)
object Message {
  import playjson.Rules._
  import playjson.Writes._
  implicit val read = Rule.gen[JsValue, Message]
  implicit val write = Write.gen[Message, JsObject]
}

trait TestService extends Directives with JsonJtoValidationSupport {
  implicit val system: ActorSystem
  implicit def executor: ExecutionContextExecutor
  implicit val materializer: Materializer

  val rule: Rule[JsValue, (String, String)] = From[JsValue] { __ =>
    import playjson.Rules._
    (
      (__ \ "foo").read[String] ~
        (__ \ "bar").read[String]
      ).tupled
  }

  val routes = pathPrefix("sample") {
    path("1") {
      post {
        // provide a Rule and get a valid result
        expect(rule) {
          case (foo: String, bar) =>
            complete(OK -> foo)
        }
      }
    } ~
    path("2") {
      post {
        // provide a Rule and get a validation result
        validate(rule) {
          case Valid((foo, bar)) =>
            complete(OK -> foo)
          case Invalid(errs) =>
            complete(BadRequest -> "Could not validate json")
        }
      }
    } ~
    path("3") {
      post {
        // reads the whole body as JsValue
        bodyAsJson { js: JsValue =>
          complete(OK -> js)
        }
      }
    } ~
    path("4") {
      post {
        // use generated Rule/Write to read an entity from Json
        entity(as[Message]) { m: Message =>
          complete(OK -> m)
        }
      }
    }
  }
}

object WebServer extends App with TestService {
  override implicit val system = ActorSystem()
  override implicit val executor = system.dispatcher
  override implicit val materializer = ActorMaterializer()
  Http().bindAndHandle(routes, "localhost", 9000)
}


```
