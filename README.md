# reactive-streams-with-akka-and-java

1. Introduction
   What are reactive streams?
   Stream: Ongoing generated data, possibly with no end.
   Reactive:
   asynchronous
   non-blocking
   back-pressure
   Reactive Streams - Key Concepts
   Element: item of data
   Publisher: starting point
   Subscriber: end-point
   Processor: transform or process elements

2. Creating our first stream
   Using the akka libraries
   Setting up a stream
   . Using the Akka Streaming libraries
   . Sources, flows and sinks
   . Creating graphs
   . Running graphs
   Creating a source
   Source<Integer, NotUsed> source = Source.range(1, 10);
   Creating a sink
   Sink<String, CompletionStage<Done>> sink = Sink.foreach(System.out::println);
   Creating a flow
   Flow<Integer, String, NotUsed> flow = Flow.of(Integer.class).map(value -> "The next value is  " + value);
   Building and running a graph
   Creating graph
   RunnableGraph<NotUsed> graph = source.via(flow).to(sink);
   Running graph
   ActorSystem actorSystem = ActorSystem.create(Behaviors.empty(), "actorSystem");
   graph.run(actorSystem);

3. Simple Sources and Sinks
   Finite Sources
   # Source with a finite range and step up
   Source<Integer, NotUsed> source = Source.range(1, 10, 2);
   # Source with only a single value.
   Source<Integer, NotUsed> source = Source.single(17);
   # Source with collection of objects.
   List<String> names = List.of("James", "Jill", "Simon", "Sally", "Denise", "David");
   Source<String, NotUsed> source = Source.from(names);

   Infinite Sources
   # Source with repeating value for infinite times
   Source<Double, NotUsed> piSource = Source.repeat(3.141592654);
   # Source with repeating collection of object in iterator for infinite time.
   List<String> names = List.of("James", "Jill", "Simon", "Sally", "Denise", "David");
   Source<String, NotUsed> repeatingNamesSource = Source.cycle(names::iterator);
   # Source with infinite integers
   Iterator<Integer> infiniteRange = Stream.iterate(0, i -> i + 1).iterator();
   Source<Integer, NotUsed> infiniteRangeSource = Source.fromIterator(() -> infiniteRange);

   Basic source operators
   # Use throttle method to throttle or control the rate at which source generate the value.
   Iterator<Integer> infiniteRange = Stream.iterate(0, i -> i + 1).iterator();
   Source<Integer, NotUsed> infiniteRangeSource = Source
   .fromIterator(() -> infiniteRange)
   .throttle(1, Duration.ofSeconds(3));
   The above code will send a maximum of one element every 3 seconds from the given range. First it will send 0 in first 3 seconds, 1 next in next 3 seconds and so on.

        # Use take method to limit the number of elements from a source. It can be used with both finite and infinite sources but when it is used with an infinite source then it can turn it to a finite source by limiting the number of elements.
        	Iterator<Integer> infiniteRange = Stream.iterate(0, i -> i + 1).iterator();
        	Source<Integer, NotUsed> infiniteRangeSource = Source
                .fromIterator(() -> infiniteRange)
                .throttle(1, Duration.ofSeconds(3))
                .take(5); 
            The above source will produce only 5 elements even though we passed an infinite range.

   Creating sinks
   # Sink ignoring all the data processed
   Sink<String, CompletionStage<Done>> ignoreSink = Sink.ignore();

   Graph convenience methods
   # Creating a graph directly with sink
   sink.runWith(repeatingNamesSource, actorSystem);
   // equivalent to repeatingNamesSource.to(sink).run(actorSystem)

        # Creating a graph directly with sink and processing via flow
        	sink.runWith(repeatingNamesSource.via(stringFlow), actorSystem);
        	// equivalent to repeatingNamesSource.via(stringFlow).to(sink).run(actorSystem)

        # Writing the flow directly during creating graph using sink
        	repeatingNamesSource.via(stringFlow).runForeach(System.out::println, actorSystem);

4. Simple Flows
   Map and Filter
   Map: Maps element one to one
   Flow<Integer, String, NotUsed> flow = Flow.of(Integer.class).map(value -> "The next value is  " + value);
   Filter: It decides which elements to discard.
   Flow<Integer, Integer, NotUsed> filterFlow = Flow.of(Integer.class).filter(value -> value % 17 == 0);

   MapConcat
   This MapConcat method is equivalent to flatMap() in Java.
   The mapping function produces an iterable of zero or more elements for each input element, which are then individually pushed downstream.
   Ex with akka streams:
   Flow<Integer, Integer, NotUsed> mapConcatFlow = Flow.of(Integer.class)
   .mapConcat(value -> {
   List<Integer> results = List.of(value, value + 1, value + 2);
   return results;
   });
   The above code will produce three integers for every value. For example if the source is 1, 2 then after mapConcat it will produce 1,2,3,2,3,4.

   Grouped
   In Akka Streams, grouped is a flow operator that accumulates incoming elements into a collection until a certain number of elements have been received, at which point it emits the entire collection downstream.
   Ex:
   If a stream of integers is passed to a grouped(3) operator, it will emit a Seq or List containing three integers at a time.
   Flow<Integer, List<Integer>, NotUsed> groupedFlow = Flow.of(Integer.class)
   .grouped(3);

   Flows of generic data types
   The mapConcat() can be used to ungroup a list of objects but it's only reusable if its chained on the back of the method that created the group because you can't use the generic types within the flow directly.
   Ex:
   Flow<Integer, Integer, NotUsed> groupedFlow = Flow.of(Integer.class)
   .grouped(3)
   .map(value -> {
   List<Integer> newList = new ArrayList<>(value);
   Collections.sort(newList, Collections.reverseOrder());
   return newList;
   })
   .mapConcat(value -> value);

   Combining flows with sinks and sources
   . We can combine flows to get another flow.
   Ex: Flow<Integer, Integer, NotUsed> chainedFlow = filterFlow.via(mapConcatFlow);

    	. We can combine source and flow to get another source.
    		Ex:
    			Source<String, NotUsed> sentencesSource = Source.from(List.of("The sky is 			blue",
                	"The moon is only seen at night",
                	"The planets orbit around the sun"));

        		Flow<String, Integer, NotUsed> howManyWordsFlow = Flow.of(String.class)
                	.map(sentence -> sentence.split(" ").length);

        		Source<Integer, NotUsed> howManyWordsSource = sentencesSource.via(					howManyWordsFlow);

        	We can also combine the flow at the time of creating source.
        		Source<Integer, NotUsed> sentencesSource2 = Source.from(List.of("The sky is 		blue",
                		"The moon is only seen at night",
                		"The planets orbit around the sun"))
                	.map(sentence -> sentence.split(" ").length);

        . We can combine sink and flow to get another sink
        	Sink<Integer, CompletionStage<Done>> sink = Sink.ignore();
        	Sink<String, NotUsed> combinedSink = howManyWordsFlow.to(sink);

5. Exercise 1 - Big Primes
   See BigPrimes project.

6. Materialized values
   What are materialized values?
   A materialized value in Akka Streams is a value produced at the time of a stream's materialization (when it is executed). It provides a way to get information or control a stream from the outside world, and its type is distinct from the data elements flowing through the stream.

   	Each stream operator, whether it's a Source, Flow, or Sink, has a materialized value type, indicated by the last type parameter. 

   	Materialized values in different stream components
   		. Sources: A Source can provide a control handle to the stream. For example, 		Source.actorRef materializes an ActorRef that can be used to inject elements 	into the stream. A timer-based source might return a Cancellable to stop the 		timer. A source with no meaningful value simply materializes to NotUsed. 
   		. Flows: As intermediate stages, Flows often don't have a useful materialized 		value and typically default to NotUsed.
   		. Sinks: Since Sinks are the end of a stream, their materialized value is often 	the only way to get a result out of the stream. For example, Sink.head 			materializes a Future that completes with the first element of the stream, 		while Sink.fold materializes a Future that completes with the final folded 		result.

   The fold method and selecting a materialized value for the graph
   Sink<Integer, CompletionStage<Integer>> sinkWithCounter = Sink
   .fold(0, (counter, value) -> {
   System.out.println(value);
   return counter + 1;
   });

        CompletionStage<Integer> result = source
                .via(greaterThan200Filter)
                .via(evenNumberFilter)
                .toMat(sinkWithCounter, Keep.right())
                .run(actorSystem);

        result.whenComplete((value, throwable) -> {
           if (throwable == null) {
               System.out.println("The graph's materialized value is " + value);
           } else {
               System.out.println("Something went wrong " + throwable);
           }
        });

   Terminating the actor system
   result.whenComplete((value, throwable) -> {
   if (throwable == null) {
   System.out.println("The graph's materialized value is " + value);
   } else {
   System.out.println("Something went wrong " + throwable);
   }
   actorSystem.terminate();
   });
   We can close the actor system when we are dealing finite sources.

   The reduce method
   In Akka Streams, the reduce operator is a Sink operator that performs a reduction (or aggregation) operation on the elements of a stream. It takes a binary function (a "reducer") and applies it iteratively to the stream elements, accumulating them into a single final value.

    	reduce does not require an initial value like fold. Instead, it uses the first element of the stream as the initial accumulator.

    	Getting materialized value from reduce method.
    		Sink<Integer, CompletionStage<Integer>> sinkWithSum = Sink
                .reduce((firstValue, secondValue) -> {
                    System.out.println(secondValue);
                    return firstValue + secondValue;
                });

7. Stream lifecycles
   Converting infinite streams to finite streams with take and limit.
   CompletionStage<Integer> result = source.take(100)
   .via(greaterThan200Filter.limit(110))
   .viaMat(evenNumberFilter, Keep.right())
   .toMat(sinkWithSum, Keep.right())
   .run(actorSystem);
   Note: If we replace 110 with 60 in limit method then we will get below error.
   Something went wrong akka.stream.StreamLimitReachedException: limit of 60 reached

   The takeWhile and takeWithin functions
   takeWhile: In Akka Streams, takeWhile is an operator that processes elements from a 	stream as long as a given predicate function returns true. It passes all matching 		elements downstream. When the first element fails the predicate, takeWhile 			completes the stream and the element that caused the failure, and all subsequent 		elements, are not emitted.

    	takeWithin: In Akka Streams, takeWithin is a stream operator that passes elements 		downstream for a specified duration and then completes the stream. It is primarily 		used for handling timeouts in data processing pipelines. Once the time limit is 	reached, the stream is terminated, and any further elements are ignored.

    	CompletionStage<Integer> result = source
                .take(100)
                .throttle(1, Duration.ofSeconds(1))
                .takeWithin(Duration.ofSeconds(5))
                .via(greaterThan200Filter.limit(110))
                .viaMat(evenNumberFilter.takeWhile(value -> value < 900), Keep.right())
                .toMat(sinkWithSum, Keep.right())
                .run(actorSystem);

8. Logging
   Flow<Integer, Integer, NotUsed> loggedFlow = Flow.of(Integer.class)
   .map(x -> {
   actorSystem.log().debug("flow input " + x);
   int y = x * 2;
   actorSystem.log().debug("flow output " + y);
   return y;
   });

   The below code needs some configuration settings via application.conf file.
   Flow<Integer, Integer, NotUsed> flow = Flow.of(Integer.class)
   .log("flow input")
   .map(x -> x * 2)
   .log("flow output");

9. Back pressure and graph performance
   Asynchronous boundaries
   In Akka Streams, asynchronous boundaries are points within a stream processing graph where the execution of stages can be decoupled and run in separate Akka Actors. By default, all stages in an Akka Stream run sequentially within the same actor. Introducing an asynchronous boundary allows for parallel processing and improved throughput, especially for I/O-bound or computationally intensive operations.

   	CompletionStage<Done> result = source
                .via(bigIntegerGenerator)
                .async()
                .via(primeGenerator)
                .async()
                .via(createGroup)
                .toMat(printSink, Keep.right())
                .run(actorSystem);

   Introducing back-pressure
   In Akka, backpressure is a crucial flow-control mechanism used primarily in Akka Streams to manage the rate of data flowing between processing stages. It prevents a fast data producer from overwhelming a slower consumer, ensuring system stability and resource efficiency. This is an essential feature for building robust, reactive applications.

   Adding buffers to the graph
   CompletionStage<Done> result = source
   .via(bigIntegerGenerator)
   .buffer(16, OverflowStrategy.backpressure())
   .async()
   .via(primeGenerator.addAttributes(Attributes.inputBuffer(16, 32))) // Modifies actor buffer size
   .async()
   .via(createGroup)
   .toMat(printSink, Keep.right())
   .run(actorSystem);

   Other overflow strategies
   dropHead : Discards the oldest elements in the buffer when a new element arrives.

    	dropTail : Discards the newest elements arriving when the buffer is full, allowing older, already-in-the-buffer elements to be processed. 

    	dropNew : Similar to Drop-Tail, but specifically discards the new element being added to the full buffer. 

    	dropBuffer : clears the entire buffer to make space for the new element.

   	Parallelism
   		mapAsync:
   			mapAsync is an operator in Akka Streams that enables parallel and asynchronous processing of stream elements while maintaining the original order. It is used for tasks that return a Future (or CompletionStage in Java) and would otherwise block the stream

   			Flow<BigInteger, BigInteger, NotUsed> primeGeneratorAsync = Flow.of(BigInteger.class)
                .mapAsync(4, input -> {
                    CompletableFuture<BigInteger> futurePrime = new CompletableFuture<>();
                    futurePrime.completeAsync(() -> {
                        BigInteger prime = input.nextProbablePrime();
                        System.out.println("Prime: " + prime);
                        return prime;
                    });
                    return futurePrime;
                });

        mapAsyncUnordered:
        	mapAsyncUnordered is an Akka Streams operator used to perform asynchronous processing on stream elements with a specified level of parallelism, without preserving the original stream's order. This makes it ideal for tasks where throughput is more important than the sequence of results. 

10. Exercise 2 - monitoring vehicle speed (Position Tracker)
    Check the Main class of position speed tracker project.

11. The GraphDSL
    Why do we need the GraphDSL?
    We need Akka's Graph DSL to model and build complex, non-linear data processing pipelines, including fan-in (multiple inputs) and fan-out (multiple outputs) operations, where a standard linear Source-Flow-Sink model isn't sufficient. The Graph DSL provides a visual, diagram-like syntax that allows you to easily translate whiteboard designs into code, creating reusable and shareable streaming graphs.

    Introducing the GraphDSL Syntax
    RunnableGraph<CompletionStage<DataType>> graph =
    RunnableGraph.fromGraph(
    GraphDSL.create(
    gowmv, // graph object with materialized value
    (builder, ref) -> {
    ...
    }
    )
    );

    Constructing a simple Graph using the GraphDSL
    RunnableGraph<CompletionStage<VehicleSpeed>> graph = RunnableGraph.fromGraph(
    GraphDSL.create(sink, (builder, out) -> {
    SourceShape<String> sourceShape = builder.add(Source.repeat("go")
    .throttle(1, Duration.ofSeconds(10)));
    //SourceShape<String> sourceShape = builder.add(source);

                    FlowShape<String, Integer> vehicleIdsShape = builder.add(vehicleIds);
                    FlowShape<Integer, VehiclePositionMessage> vehiclePositionsShape =
                            builder.add(vehiclePositions);
                    FlowShape<VehiclePositionMessage, VehicleSpeed> vehicleSpeedsShape =
                            builder.add(vehicleSpeeds);
                    FlowShape<VehicleSpeed, VehicleSpeed> speedFilterShape = builder.add(speedFilter);

                    // DON"T NEED TO DO THIS - OUT IS OUR SINKSHAPE
                    //SinkShape<VehicleSpeed> sinkShape = builder.add(sink);

                    builder.from(sourceShape)
                            .via(vehicleIdsShape)
                            .via(vehiclePositionsShape)
                            .via(vehicleSpeedsShape)
                            .via(speedFilterShape)
                            .to(out);

                    return ClosedShape.getInstance();
                })
        );

    Understanding the Graph construction and introducing Asynchronous Boundaries
    The below code means that this flow should run in its own dedicated actor. The behavior of this async() method is different from the one we used where a standard linear Source-Flow-Sink model.

    	FlowShape<Integer, VehiclePositionMessage> vehiclePositionsShape =
                            builder.add(vehiclePositions.async());

12. Complex flow types
    Introducing fan-in and fan-out shapes
    fan-in:
    "Fan-in" in the context of Akka, a toolkit for building concurrent and distributed applications, refers to a function or junction that combines multiple inputs into a single output. It is a key concept used in Akka Streams' Graph DSL (Domain-Specific Language) to define how data flows and merges from different sources.
    fan-out:
    In the context of Akka, particularly Akka Streams, a "fan-out" refers to a pattern where a single stream of data is split and routed to multiple different streams or processing paths. This allows for distributing a single source of data to various consumers or downstream operations, enabling parallel processing or differentiated handling of the same data.

    Broadcast and merge
    Check ComplexFlowTypes class in positionSpeedTracker project.

    	RunnableGraph<CompletionStage<Done>> graph = RunnableGraph.fromGraph(
                GraphDSL.create(sink, (builder, out) -> {
                    SourceShape<Integer> sourceShape = builder.add(source);
                    FlowShape<Integer, Integer> flow1Shape = builder.add(flow1);
                    FlowShape<Integer, Integer> flow2Shape = builder.add(flow2);
                    UniformFanOutShape<Integer, Integer> broadcast =
                            builder.add(Broadcast.create(2));
                    UniformFanInShape<Integer, Integer> merge =
                            builder.add(Merge.create(2));

                    /*builder.from(sourceShape)
                            .viaFanOut(broadcast);

                    builder.from(broadcast.out(0))
                            .via(flow1Shape);

                    builder.from(broadcast.out(1))
                            .via(flow2Shape);

                    builder.from(flow1Shape)
                            .toInlet(merge.in(0));

                    builder.from(flow2Shape)
                            .toInlet(merge.in(1));

                    builder.from(merge).to(out);*/

                    builder.from(sourceShape)
                            .viaFanOut(broadcast)
                            .via(flow1Shape);

                    builder.from(broadcast).via(flow2Shape);

                    builder.from(flow1Shape)
                            .viaFanIn(merge)
                            .to(out);
                    builder.from(flow2Shape).viaFanIn(merge);


                    return ClosedShape.getInstance();
                })
        );

        graph.run(actorSystem);

    Using balance for parallelism
    RunnableGraph<CompletionStage<Done>> graph = RunnableGraph.fromGraph(
    GraphDSL.create(sink, (builder, out) -> {
    SourceShape<Integer> sourceShape = builder.add(source);
    //FlowShape<Integer, Integer> flowShape = builder.add(flow);

                    UniformFanOutShape<Integer, Integer> balance =
                            builder.add(Balance.create(4, true));
                    UniformFanInShape<Integer, Integer> merge =
                            builder.add(Merge.create(4));

                    builder.from(sourceShape)
                            .viaFanOut(balance);

                    for (int i = 0; i < 4; i++) {
                        builder.from(balance)
                                .via(builder.add(flow.async()))
                                .toFanIn(merge);
                    }

                    builder.from(merge)
                            .to(out);
                    return ClosedShape.getInstance();
                })
        );

    Implementing parallelism in GraphDSL in Main class of project positionSpeedTracker
    RunnableGraph<CompletionStage<VehicleSpeed>> graph = RunnableGraph.fromGraph(
    GraphDSL.create(sink, (builder, out) -> {
    SourceShape<String> sourceShape = builder.add(Source.repeat("go")
    .throttle(1, Duration.ofSeconds(10)));
    //SourceShape<String> sourceShape = builder.add(source);

                    FlowShape<String, Integer> vehicleIdsShape = builder.add(vehicleIds);
                    /*FlowShape<Integer, VehiclePositionMessage> vehiclePositionsShape =
                            builder.add(vehiclePositions.async());*/
                    FlowShape<VehiclePositionMessage, VehicleSpeed> vehicleSpeedsShape =
                            builder.add(vehicleSpeeds);
                    FlowShape<VehicleSpeed, VehicleSpeed> speedFilterShape = builder.add(speedFilter);

                    UniformFanOutShape<Integer, Integer> balance =
                            builder.add(Balance.create(8, true));

                    UniformFanInShape<VehiclePositionMessage, VehiclePositionMessage> merge =
                            builder.add(Merge.create(8));

                    // DON"T NEED TO DO THIS - OUT IS OUR SINKSHAPE
                    //SinkShape<VehicleSpeed> sinkShape = builder.add(sink);

                    builder.from(sourceShape)
                            .via(vehicleIdsShape)
                            .viaFanOut(balance);

                    for (int i = 0; i < 8; i++) {
                        builder.from(balance).via(builder.add(vehiclePositions.async()))
                                .toFanIn(merge);
                    }

                    builder.from(merge)
                            .via(vehicleSpeedsShape)
                            .via(speedFilterShape)
                            .to(out);

                    return ClosedShape.getInstance();
                })
        );

    Uniform fan-in and fan-out shapes
    Partition:
    In Akka Streams, a "fan-out partition" is an operator that distributes incoming stream elements to multiple downstream consumers, where each element goes to only one consumer based on a partitioner function that determines the target output. This process, often referred to as "Partition," is different from a broadcast, where an element is sent to all consumers. You can implement this in the Graph DSL by using the Partition stage.

    	RunnableGraph<CompletionStage<Done>> graph = RunnableGraph.fromGraph(
                GraphDSL.create(Sink.foreach(System.out::println), (builder, out) -> {
                    SourceShape<Integer> sourceShape = builder.add(
                            Source.repeat(1).throttle(1, Duration.ofSeconds(1))
                                    .map(x -> {
                                        Random r = new Random();
                                        return 1 + r.nextInt(10);
                                    })
                    );

                    FlowShape<Integer, Integer> flowShape = builder.add(
                            Flow.of(Integer.class).map(x -> {
                                System.out.println("Flowing " + x);
                                return x;
                            })
                    );

                    UniformFanOutShape<Integer, Integer> partition = builder.add(
                            Partition.create(2, x -> (x == 1 || x == 10) ? 0 : 1)
                    );

                    UniformFanInShape<Integer, Integer> merge = builder.add(Merge.create(2));

                    builder.from(sourceShape)
                            .viaFanOut(partition);

                    builder.from(partition.out(0))
                            .via(flowShape)
                            .viaFanIn(merge)
                            .to(out);

                    builder.from(partition.out(1))
                            .viaFanIn(merge);

                    return ClosedShape.getInstance();
                })
        );

    Non uniform fan-in and fan-out shapes
    UnzipWith is a fan-out operator in Akka Streams that splits a single stream of elements into multiple streams. It does this by applying a provided splitter function to each incoming element. The number of output streams is determined by the arity of the function you provide

    	zipWith is a fan-in operator in Akka Streams that combines elements from multiple input streams using a user-defined function. It is often used to enrich or correlate data from different sources

    	RunnableGraph<CompletionStage<Done>> graph = RunnableGraph.fromGraph(
                GraphDSL.create(Sink.foreach(System.out::println), (builder, out) -> {

                    SourceShape<Integer> source = builder.add(Source.range(1, 10));
                    FlowShape<Integer, Integer> integerFlow = builder.add(
                            Flow.of(Integer.class).map(x -> {
                                System.out.println("Integer flow " + x);
                                return x;
                            })
                    );
                    FlowShape<Boolean, Boolean> booleanFlow = builder.add(
                            Flow.of(Boolean.class).map(x -> {
                                System.out.println("Boolean flow " + x);
                                return x;
                            })
                    );
                    FlowShape<String, String> stringFlow = builder.add(
                            Flow.of(String.class).map(x -> {
                                System.out.println("String flow " + x);
                                return x;
                            })
                    );

                    FanOutShape3<Integer, Integer, Boolean, String> fanOut = builder.add(
                            UnzipWith.create3(input -> new Tuple3<>(input, input %2 == 0, "It's " + input))
                    );

                    FanInShape3<Integer, Boolean, String, String> fanIn = builder.add(
                            ZipWith.create3((a, b, c) -> {
                               StringBuilder sb = new StringBuilder();
                               sb.append("The number was " + a);
                               sb.append(", which is " + (b ? "even" : "odd"));
                               sb.append(", and the string was " + c);
                               return sb.toString();
                            })
                    );

                    builder.from(source).toInlet(fanOut.in());
                    builder.from(fanOut.out0()).via(integerFlow).toInlet(fanIn.in0());
                    builder.from(fanOut.out1()).via(booleanFlow).toInlet(fanIn.in1());
                    builder.from(fanOut.out2()).via(stringFlow).toInlet(fanIn.in2());
                    builder.from(fanIn.out()).to(out);

                    return ClosedShape.getInstance();
                })
        );

13. Graphs with multiple sources and sinks
    Combining sources with a fan-in shape

    	Flow<Transfer, Transaction, NotUsed> getTransactionsFromTransfer = Flow.of(Transfer.class)
                .mapConcat(transfer -> List.of(transfer.getFrom(), transfer.getTo()));

        Source<Integer, NotUsed> transactionIDsSource = Source.fromIterator(() -> Stream.iterate(1, i -> i + 1).iterator());

        RunnableGraph<CompletionStage<Done>> graph = RunnableGraph.fromGraph(
                GraphDSL.create(Sink.foreach(System.out::println), (builder, out) -> {
                    FanInShape2<Transaction, Integer, Transaction> assignTransactionIDs =
                            builder.add(ZipWith.create((trans, id) -> {
                                trans.setUniqueId(id);
                                return trans;
                            }));

                    builder.from(builder.add(source))
                            .via(builder.add(generateTransfer))
                            .via(builder.add(getTransactionsFromTransfer))
                            .toInlet(assignTransactionIDs.in0());

                    builder.from(builder.add(transactionIDsSource))
                            .toInlet(assignTransactionIDs.in1());

                    builder.from(assignTransactionIDs.out()).to(out);

                    return ClosedShape.getInstance();
                })
        );

    Sending to an interim sink with alsoTo
    alsoTo:
    The alsoTo operator in Akka Streams sends elements to an additional Sink in the middle of a linear stream while allowing the elements to continue flowing downstream. This is a fan-out operation that enables a side-effect, like logging or database storage, without interrupting the main data flow.

    	Sink<Transfer, CompletionStage<Done>> transferLogger = Sink.foreach(transfer -> {
            System.out.println("Transfer from " + transfer.getFrom().getAccountNumber() +
                    " to " + transfer.getTo().getAccountNumber() + " of " + transfer.getTo().getAmount());
        });

        RunnableGraph<CompletionStage<Done>> graph = RunnableGraph.fromGraph(
                GraphDSL.create(Sink.foreach(System.out::println), (builder, out) -> {
                    FanInShape2<Transaction, Integer, Transaction> assignTransactionIDs =
                            builder.add(ZipWith.create((trans, id) -> {
                                trans.setUniqueId(id);
                                return trans;
                            }));

                    builder.from(builder.add(source))
                            .via(builder.add(generateTransfer.alsoTo(transferLogger)))
                            .via(builder.add(getTransactionsFromTransfer))
                            .toInlet(assignTransactionIDs.in0());

                    builder.from(builder.add(transactionIDsSource))
                            .toInlet(assignTransactionIDs.in1());

                    builder.from(assignTransactionIDs.out()).to(out);

                    return ClosedShape.getInstance();
                })
        );

    Diverting outliers to a different sink with divertTo
    divertTo:
    The divertTo operator in Akka is used in Akka Streams to redirect a subset of elements to a separate processing path, or "sink," based on a conditional function. It is a powerful tool for implementing conditional logic in a stream, such as segregating different types of data or handling errors separately.

    	Sink<Transaction, CompletionStage<Done>> rejectedTransactionsSink = Sink.foreach(trans -> {
            System.out.println("REJECTED transaction " + trans + " as account balance is " + accounts.get(trans.getAccountNumber()).getBalance());
        });

        RunnableGraph<CompletionStage<Done>> graph = RunnableGraph.fromGraph(
                GraphDSL.create(Sink.foreach(System.out::println), (builder, out) -> {
                    FanInShape2<Transaction, Integer, Transaction> assignTransactionIDs =
                            builder.add(ZipWith.create((trans, id) -> {
                                trans.setUniqueId(id);
                                return trans;
                            }));

                    builder.from(builder.add(source))
                            .via(builder.add(generateTransfer.alsoTo(transferLogger)))
                            .via(builder.add(getTransactionsFromTransfer))
                            .toInlet(assignTransactionIDs.in0());

                    builder.from(builder.add(transactionIDsSource))
                            .toInlet(assignTransactionIDs.in1());

                    builder.from(assignTransactionIDs.out())
                            .via(builder.add(Flow.of(Transaction.class)
                                    .divertTo(rejectedTransactionsSink, trans -> {
                                        Account account = accounts.get(trans.getAccountNumber());
                                        BigDecimal forecastBalance = account.getBalance().add(trans.getAmount());
                                        return (forecastBalance.compareTo(BigDecimal.ZERO) < 0);
                                    })))
                            .via(builder.add(applyTransactionsToAccounts))
                            .to(out);

                    return ClosedShape.getInstance();
                })
        );

14. Non-runnable or partial graphs
    Creating and combining open shapes
    In Akka Streams, a partial graph is a reusable, incomplete blueprint for stream processing that can be used as a component (source, flow, or sink) of larger graphs, but it's not yet a complete, runnable stream itself. It represents a connected set of flows, junctions, and ports that are not necessarily fully connected into a single source, flow, or sink. You construct these partial graphs using the GraphDSL and then materialize them as Source, Sink, or Flow shapes to build more complex streaming topologies.
    A partial graph in Akka Streams is a reusable, composable blueprint for a stream processing pipeline that is not yet fully connected. While a RunnableGraph has all its inputs and outputs wired, a partial graph exposes one or more unconnected ports, allowing it to be integrated into a larger stream.
    The concept of a partial graph is central to building modular and complex stream topologies, such as those with fan-in or fan-out patterns.


		Graph<SourceShape<Transaction>, NotUsed> sourcePartialGraph = GraphDSL.create(
                builder -> {
                    FanInShape2<Transaction, Integer, Transaction> assignTransactionIDs =
                            builder.add(ZipWith.create((trans, id) -> {
                                trans.setUniqueId(id);
                                return trans;
                            }));

                    builder.from(builder.add(source))
                            .via(builder.add(generateTransfer.alsoTo(transferLogger)))
                            .via(builder.add(getTransactionsFromTransfer))
                            .toInlet(assignTransactionIDs.in0());

                    builder.from(builder.add(transactionIDsSource))
                            .toInlet(assignTransactionIDs.in1());

                    return SourceShape.of(assignTransactionIDs.out());
                }
        );

        Graph<SinkShape<Transaction>, CompletionStage<Done>> sinkPartialGraph = GraphDSL.create(
                Sink.foreach(System.out::println), (builder, out) -> {
                    FlowShape<Transaction, Transaction> entryFlow =
                            builder.add(Flow.of(Transaction.class)
                                    .divertTo(rejectedTransactionsSink, trans -> {
                                        Account account = accounts.get(trans.getAccountNumber());
                                        BigDecimal forecastBalance = account.getBalance().add(trans.getAmount());
                                        return (forecastBalance.compareTo(BigDecimal.ZERO) < 0);
                                    }));
                    builder.from(entryFlow)
                            .via(builder.add(applyTransactionsToAccounts))
                            .to(out);
                    return SinkShape.of(entryFlow.in());
                }
        );

        RunnableGraph<CompletionStage<Done>> graph = RunnableGraph.fromGraph(
                GraphDSL.create(sinkPartialGraph, (builder, out) -> {
                    builder.from(builder.add(sourcePartialGraph))
                            .to(out);
                    return ClosedShape.getInstance();
                })
        );

    Using open shapes outside the graphDSL
    	Source<Transaction, NotUsed> newSource = Source.fromGraph(sourcePartialGraph);
        newSource.to(sinkPartialGraph).run(actorSystem);

15. Using actors in graphs
    Adding an actor to our project
    See AccountManager class in transactionmanager project

    Using actors as flows
    See Main class in transactionmanager project

    Using actors as sinks
    See Main class in transactionmanager project

16. Advanced backpressure
    Conflate
    In Akka Streams, conflate is a backpressure-aware operator used to handle situations where a fast-producing upstream component overwhelms a slower downstream component. It aggregates or combines incoming elements into a single element when the downstream is applying backpressure, ensuring the stream's buffer does not fill up and that the stream does not crash.

    	Akka provides two primary variants of the conflate operator:
    	. conflate(aggregate: (U, T) => U): This is the simpler version. It takes a single function to combine a new element with the aggregated value. The initial state of the aggregation is the first element itself.
    		Flow<Integer, Integer, NotUsed> conflateFlow = Flow.of(Integer.class)
                .conflate((accumulator, element) -> {
                   return accumulator + element;
                });

    	. conflateWithSeed(seed: T => U, aggregate: (U, T) => U): This more powerful version is useful when the aggregated type (U) is different from the incoming element type (T).

    		Flow<Integer, List, NotUsed> conflateWithSeedFlow = Flow.of(Integer.class)
                .conflateWithSeed(
                        a -> {
                            List<Integer> list = new ArrayList<>();
                            list.add(a);
                            return list;
                        },
                        (list, a) -> {
                            list.add(a);
                            return list;
                        }
                );

    Extrapolate and expand
    Extrapolate:
    The extrapolate operator in Akka Streams allows a slower upstream process to satisfy the demands of a faster downstream consumer. It does this by using the last element received from upstream to generate additional elements on demand. This mechanism prevents the downstream from having to wait for new data and helps to smooth out uneven data rates in a stream.
    Ex:
    Flow<String, String, NotUsed> extrapolateFlow = Flow.of(String.class)
    .extrapolate(x -> List.of(x).iterator());

    	expand:
    		The expand operator in Akka Streams is a powerful flow operator that transforms a single incoming element into a sequence of zero or more elements. Unlike map, which produces exactly one output element for each input, expand is used when you need to "expand" an incoming element into a stream of potentially infinite new elements. 
    		It is particularly useful for rate-limiting, backpressure handling, and recovering from slow upstream sources by emitting synthetic "fill-in" elements. 

17. The java flow package
    Java 9 reactive streams interfaces
    java.util.concurrent.Flow.*
    Publisher
    Subscriber
    Processor
    Subscription

18. Case Study - Blockchain mining
    Check Akka block chain project in akka streams.


