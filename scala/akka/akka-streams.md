## Have Akka Streams SBT Dependency in `build.sbt`

`libraryDependencies += "com.typesafe.akka" %% "akka-stream" % "2.5.26",`

#### Creates ActorSystem | Creates a Source | Run the Source | Completes Stream Processing | Closes ActorSystem

scala
implicit val actorSystem = ActorSystem()
implicit val actorMaterializer = ActorMaterializer()

    val source: Source[Int, NotUsed] = Source(1 to 10)
    val done: Future[Done] = source.runForeach(i => println(i))

    done.onComplete(_ => actorSystem.terminate())

#### (1) Transforms values from source | (2) Scans Values [Like a Fold But not a Terminal Application](3) Folds values (Terminal Operation) - Operations where Materialization is triggered.

scala
implicit val actorSystem = ActorSystem()
implicit val actorMaterializer = ActorMaterializer()
val source: Source[Int, NotUsed] = Source(1 to 10)
//From 1

    val doubles: Source[Double, NotUsed] = source.map(_ * 2.0)

    val factorials: Source[BigInt, NotUsed] = source.scan(BigInt(1))((acc, next) => acc * next)

    val doubledFactorials: Source[Double, NotUsed] = factorials.fold(1.0)(_ * 2.0)

    //From 1
    val done: Future[Done] = source.runForeach(i => println(i))
    done.onComplete(_ => actorSystem.terminate())

#### Defines a Source | Flow | Sink -> takes a Source to Sink via Flow!

scala
implicit val actorSystem = ActorSystem()
implicit val actorMaterializer = ActorMaterializer()
val source: Source[Int, NotUsed] = Source(1 to 10)
//From 1

    val source: Source[Int, NotUsed] = Source(1 to 10)
    val anotherSource = Source(10 to 1)

    source
    .zipWi

    val strToByteStrFlow: Flow[String, ByteString, NotUsed] = Flow[String].map(s => ByteString("[" + s + "]"))

    val fileSink: Sink[ByteString, Future[IOResult]] = FileIO.toPath(Paths.get("fileSink.txt"))

    source
      .map(_.toString)
      .via(strToByteStrFlow)
      .to(fileSink)

    //From 1
    val done: Future[Done] = source.runForeach(i => println(i))
    done.onComplete(_ => actorSystem.terminate())

#### Zips a Source to Another Source

scala
val source: Source[Int, NotUsed] = Source(1 to 10)
val anotherSource = Source(10 to 1)

    val zippedSource: Source[Int, NotUsed] =
      source
        .zipWith(anotherSource){
          case (s1, s2) => s1 + s2
        }

#### Throttling involved

scala
import scala.concurrent.duration.\_

    val source: Source[Int, NotUsed] = Source(1 to 10)
    source
        .throttle(elements = 1, 1.second)
        .runForeach(println)

Note: You always run a Stream with a Sink.

So a code snippet like :- `source.runForeach(println)`

means the same as :- `source.run(Sink.foreach(println))`

#### Map from one element to multiple elems and receive a "flattened" Stream - `mapConcat`

scala
case class User(name: String, contacts: List[String])

    val users = List(User("User1", List("987654321", "987654322")))
    val usersSource: Source[User, NotUsed] = Source(users)

    val contacts: Source[String, NotUsed] = usersSource.mapConcat(_.contacts)
    contacts.runForeach(println)

    val contactsList: Source[List[String], NotUsed] = usersSource.map(_.contacts)
    contactsList.runForeach(println)

#### Splitting a Stream into multiple Streams - _Junctions_ e.g. a `Broadcast`

scala

    case class User(name: String, contacts: List[String])
    val users = List(User("User1", List("987654321", "987654322")), User("User2", List("887654320", "887654321")))
    val usersSource: Source[User, NotUsed] = Source(users)

    val broadcast = RunnableGraph.fromGraph(GraphDSL.create() { implicit b =>
      import GraphDSL.Implicits._

      val bcast = b.add(Broadcast[User](2))     //2 output ports
      usersSource ~> bcast.in                   //usersSource is the Inlet

      bcast.out(0) ~> Flow[User].map(_.name) ~> Sink.foreach(println)           //outPort 0 is user's name and it Sinks as println
      bcast.out(1) ~> Flow[User].mapConcat(_.contacts)  ~> Sink.foreach(println)    //outPort 0 is user's contacts and it Sinks as println

      ClosedShape
    })

    broadcast.run() //Running the Broadcast!

#### Materializing values out of a Stream

scala

    case class User(name: String, contacts: List[String])
    val users = List(User("User1", List("987654321", "987654322")), User("User2", List("887654320", "887654321")))
    val usersSource: Source[User, NotUsed] = Source(users)

    val count: Flow[User, Int, NotUsed] = Flow[User].map(_ => 1)
    val sumSink: Sink[Int, Future[Int]] = Sink.fold[Int, Int](0)(_ + _)

    val counterGraph: RunnableGraph[Future[Int]] =
      usersSource
        .via(count)
        .toMat(sumSink)(Keep.right)
    //`toMat` is used to transform  the materialized value of the source and sink, and convenience function `Keep.right` to say that we are only interested in the materialized value of the sink.

    counterGraph.run().foreach(println)

- `Mat` type parameters on `Source[+Out, +Mat]`, `Flow[-In, +Out, +Mat]` and `Sink[-In, +Mat]`, They represent the type of values these processing parts return when materialized.
- In above snippet, _materialized_ type of `sumSink` is `Future[Int]`. `Keep.right` keeps the RunnableGraph to the right and it also has a type parameter of `Future[Int]`.
- A `RunnableGraph` is also a blueprint of the stream, can be reused, materialized multiple times.

#### Connecting `source` to `sink` and running to produce materialized value of `sink`

scala

    case class User(name: String, contacts: List[String])
    val users = List(User("User1", List("987654321", "987654322")), User("User2", List("887654320", "887654321")))
    val usersSource: Source[User, NotUsed] = Source(users)

    val sumSink: Sink[Int, Future[Int]] = Sink.fold[Int, Int](0)(_ + _)
    val valueProduced: Future[Int] = usersSource.map(_ => 1).runWith(sumSink)

#### Defining Sources, Sinks and Flows

scala
Source(List(1, 2, 3))

    Source.fromFuture

    Source.single

    Source.empty

    Sink.fold[Int, Int](0)(_ + _)

    Sink.head

    Sink.ignore

    Sink.foreach[String](println(_))

#### Wiring parts of a Stream!

scala
//Wiring up a course, Sink and Flow
Source(1 to 6).via(Flow[Int].map(\_ \* 2)).to(Sink.foreach(println))

    Source(1 to 6).map(_ * 2).to(Sink.foreach(println))

    Source(1 to 6).to(Sink.foreach(println))

    Source(1 to 6).to(Flow[Int].alsoTo(Sink.foreach(println(_))).to(Sink.ignore))

#### Operator Fusion - `asyncBoundary`

scala

    Source(List(1, 2, 3)).map(_ + 1).async.map(_ * 2).to(Sink.ignore)
    //Two regions withing the flow which will be executed in one Actor each - assuming adding and multiplying ints is an extremely costly operation
    //this will lead to a performance gain since two CPUs can work on the tasks in parallel.

#### Source pre-materialization

scala

    val matValuePoweredSource =
        Source.actorRef[String](bufferSize = 100, overflowStrategy = OverflowStrategy.fail)

    val (actorRef, source) = matValuePoweredSource.preMaterialize()

    actorRef ! "Hello!"

    source.runWith(Sink.foreach(println))

### Terminologies

#### Source

- Exactly One _Output_

#### Sink

- Exactly One _Input_

#### Flow

- Exactly One _Output_ and _Input_

#### RunnableGraph

     A Flow which has both ends attached to a Source and Sink respectively and it's ready to be `run()`.

### Actor Materializer Lifecycle

    The Materializer is bound to lifecycle of the ActorRefFactory, it is created from, which will be either an ActorSystem or ActorContext(When the materializer is created within n ActorSystem).

        implicit val system = ActorSystem()
        implicit val mat = ActorMaterializer()

### Graphs

Graphs are needed whenever you want to perform any kind of fan-in ("multiple inputs") or fan-out ("multiple outputs") operations.

Constructed using Flows and junctions, which serve as fan-in and fan-out points for Flows.
Examples for junctions :-
Fan-out:
Broadcast[T] - 1 Input and N outputs
Balance[T] - 1 Input and N outputs (an input emits to one of its outputs)
Fan-in:
Merge[In] - N Inputs and 1 Output
MergePreferred[In] - N Inputs and 1 Output
MergePrioritized[In] - N Inputs and 1 Output
MergeLatest[In] - N Inputs and 1 Output
Zip[A, B] - 2 Inputs 1 Output, zips A and B into (A, B)
Concat[A] - 2 Inputs 1 Output, concatenates two streams(first consume one then the second one)

#### Use Graph DSL

For a Simple Graph like:

                   .- - f2 - -.

IN -- f1 --> BCast Merge -- f4 --> Out
.\_ _ f3 _ \_;

Start From `RunnableGraph.fromGraph(GraphDSL.create(){<Create the Graph here>})`

Example might look like:
scala
val g = RunnableGraph.fromGraph(GraphDSL.create() { implicit builder: GraphDSL.Builder[NotUsed] =>
import GraphDSL.Implicits._
val in = Source(1 to 10)
val out = Sink.ignore
val bcast = builder.add(Broadcast[Int](2))
val merge = builder.add(Merge[Int](2))
val f1, f2, f3, f4 = Flow[Int].map(_ + 10)
in ~> f1 ~> bcast ~> f2 ~> merge ~> f3 ~> out
bcast ~> f4 ~> merge

ClosedShape
})
