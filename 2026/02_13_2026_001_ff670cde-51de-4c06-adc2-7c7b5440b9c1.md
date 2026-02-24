# Series 001

- Initially designed the deserializer and mapper to instantiate an object for every `adv` field due to 'semantic' data format reasonings

- however, after confirming with my manager, we discussed constraints and the fact that we are going to be serving 400+ devices concurrently will not be a good design for the system because it will be 400+ object instances in memory per 1-2s.

# Series 002

- There was also an extra requirement in caching the device battery information (derived from the mqtt broker) into redis.

- However, the MQTT broker and listeners are already complicated via dynamic configuration and polymorphism + the fact that the SOSListener is also performing business-specific logic on top of mqtt message deserialization and filtering. The design was already too coupled (i.e. MQTT client management, message handling and deserialization, and sos business logic handling), that if we added a caching logic that we would have a coupled 'God' Listener.

- I suggested to leverage Spring Event capability since we were already using it in some parts of the code. We would decouple the mqtt logic (client management, subscription, and deserializing) with the business logic (i.e. caching + sos management). In that way, we have the mqtt service/infra which sole purpose is to do mqtt client management and message deserialization, and we could have the observer be correctly part of the business domain logic space.

- Sidenote: I researched asynchronous programming and Spring proxies. Initially, I thought our application already had asynchronous programming implemented because we were using Spring Events for a different feature. We had event listeners annotated with `@Async` that I thought were working.

  Through my research, I realized that implementing this feature required a dedicated thread pool for the MQTT asynchronous listeners to avoid exhausting the default pool. I added a configuration class with `@EnableAsync`, but the application failed to compile. This led me to investigate how to actually implement asynchronous programming in Spring.

  I discovered that `@EnableAsync` is what actually enables Spring's asynchronous engine. It was strange because the previous code appeared to be working, or at least it was working silently. Further research revealed that without the `@EnableAsync` annotation, Spring defaults to synchronous events. Essentially, our application had been processing events synchronously in the same thread where they were published.

  Once the asynchronous engine is enabled, Spring creates a proxy class for any bean with an `@Async` annotation. A proxy allows Spring to intercept code execution. This is necessary because the application container runs on a main thread; a function cannot simply decide to run on a different thread by itself. The proxy intercepts the call and handles the thread hand-off.

  Spring proxies operate in two ways. The first is the JDK proxy, which is the default for any class implementing an interface; those interface methods define what can be proxied. The second is the CGLIB proxy, where Spring generates a subclass of the annotated class. While CGLIB doesn't require an interface, the subclassing approach means any dependencies in the constructor are effectively called twice. If you have complex dependencies, like a repository expecting a single connection, this double-instantiation can cause issues. Consequently, I decided to use JDK proxies instead.

- Diving deep --> As I was implementing the code, one of the requirements for caching was to only write new values past the expiry threshold (i.e. we set once, wait for it to expire, and then update value). However, I got to thinking about the cost of having to "reject" each public message within that period which was minimal because it was singular boolean condition check. Then I went further deeper and asked if there was a performance cost in using Spring Events, especially with our expected throughput (2000 msg/sec). From research, it seems that the Spring Bus was light enough to handle message pub and sub. So I went even deeper, and checked to see if our deserialization code could handle the expected throughput. From research, we are far optimized from it because we are using JSON Node Parsing (which is inherently expensive because you have to build a hashmap). I went to profile it and it cost us **497ms** in execution time for 400 msgs/s.

- Sidenote --> We implemented the deserialization using Jackson via the `readTree()` functuion which has to build a tree structure (DOM) of the mqtt message first. Each primitive field is wrapped with a JSON Node equivalent and each object is wrapped with a LinkedHashMap. This makes the memory usage huge and also more processing time due to having to initialize the node wrappers and building the tree.

- To optimize the deserialization function I reviewed first how I was deserialzing things in the first place. Because I initially implemented the Data POJOs to be polymorphic, that influenced the way I deserialized the messages as well. I created a JSON node from root first using `readTree()` which was expensive. Then, I would map the correct concrete class depending on the `type` field of the message and if not present, map it to a default class. However, because I did not know the performance hit before the implementation, I was using `convertValue()` (which is also expensive) to do the mapping. I had liberally used `convertValue` to do the mapping without knowing about the performance cost. Essentially I had the following: `CPU COST = Cost(readTree()) + Cost(2*convertValue()) + Cost(N*convertValue())` per message where N is the number of elements in the `adv` array. My mind was set on readability and maintanibility over performance.

```java
private List<AbstractMqttMessage<?>> deserializeMessage(MqttMessage msg) {
        List<AbstractMqttMessage<?>> parsedMessages = new ArrayList<>();
        try {
            JsonNode rootNode = objectMapper.readTree(new String(msg.getPayload()));

            // validate
            if (!rootNode.has("adv")) return parsedMessages;

            //map json node to object
            JsonNode advNode = rootNode.get("adv");
            if (advNode.isEmpty() || !advNode.isArray()) {
                DefaultConcreteMessageClass gwMsg = objectMapper.convertValue(rootNode, DefaultConcreteMessageClass.class);
                parsedMessages.add(gwMsg);
            }
            else {
                for (JsonNode node : advNode) {
                    String type = node.path("type").asText();
                    ObjectNode objectNode = (ObjectNode) rootNode;
                    objectNode.set("adv", node);

                    if (MQTTConstant.IBEACON_MESSAGE_TYPE.equals(type)) {
                        ConcreteMessageClassA wrapper = objectMapper.convertValue(objectNode, ConcreteMessageA.class);
                        parsedMessages.add(wrapper);
                    }
                    else if (MQTTConstant.DEVICE_MESSAGE_TYPE.equals(type)) {
                        ConcreteMessageClassB wrapper = objectMapper.convertValue(objectNode, ConcreteMessageClassB.class);
                        parsedMessages.add(wrapper);
                    }
                }
            }
        } catch (Exception e) {
            log.error("Error deserializing message: {}", e.getMessage(), e);
        }
        return parsedMessages;
    }
```

- To optimize the deserialization, I went first to clean up unnecessary `convertValue()` calls. In the first iteration, I avoided manually creating and setting classes (even though I knew manually creating and setting it was faster), but it seems it is unavoidable in this case. Now, with the new requirement, I had to manually extract the common fields and manually instantiate the classes. I ended up having to call `readTree()` once but I converted some of the `convertValue()` to `treeTovalue()` (which is a more optimized approach) for readability purposes. I still wanted to keep some readability/jackson capability over performance because mapping everything manually would defeat the purpose of using Jackson's automatic mapping.

```java
private List<AbstractMqttMessage<?>> deserializeMessage(MqttMessage msg) {
    List<AbstractMqttMessage<?>> parsedMessages = new ArrayList<>(2);
    try {
        JsonNode rootNode = objectMapper.readTree(msg.getPayload());

        // Validation
        JsonNode advNode = rootNode.path("adv");
        if (advNode.isMissingNode() || !advNode.isArray() || advNode.isEmpty()) {
            return parsedMessages;
        }

        // Prepare Header fields manually
        String tm = rootNode.path("tm").asText();
        String gw = rootNode.path("gw").asText();
        Long seq = rootNode.path("seq").asLong();

        List<PayloadA> payloadListA = new ArrayList<>();
        List<PayloadB> payloadListB = new ArrayList<>();

        for (JsonNode node : advNode) {
            String type = node.path("type").asText();

            if (MQTTConstant.PAYLOAD_A_TYPE.equals(type)) {
                // treeToValue is more efficient for Node -> POJO
                payloadListA.add(objectMapper.treeToValue(node, PayloadA.class));
            }
            else if (MQTTConstant.PAYLOAD_B_TYPE.equals(type)) {
                payloadListB.add(objectMapper.treeToValue(node, PayloadB.class));
            }
        }

        if (!payloadListA.isEmpty()) {
            ConcreteMessageA concreteMsgA = new ConcreteMessageA();
            concreteMsgA.setTm(tm);
            concreteMsgA.setGw(gw);
            concreteMsgA.setSeq(seq);
            concreteMsgA.setAdv(payloadListA);
            parsedMessages.add(concreteMsgA);
        }

        if (!PayloadB.isEmpty()) {
            ConcreteMessageB concreteMsgB = new ConcreteMessageB();
            concreteMsgB.setTm(tm);
            concreteMsgB.setGw(gw);
            concreteMsgB.setSeq(seq);
            concreteMsgB.setAdv(payloadListB);
            parsedMessages.add(concreteMsgB);
        }

    } catch (Exception e) {
        log.error("Error deserializing message", e);
    }
    return parsedMessages;
}
```

- The optimization resulted at **361ms** execution time for 400 msg/s. This was a significant jump meaning we could handle the expected throughput without any bottlenecks. This was X faster in throughput with a processing reduction of X (Insert calculations here)

# Series 003

- After confirming with my manager again, he gave me a thumbs up for the credits for factor, but he was very concerned about the performance of the deserialize function so he asked me to keep working on it. He had further clarify that we had an expected throughput of 4000 MQT messages per second. And so I went to review the optimized profile again and I saw that the serialized function was also taking up 18% of the total flame graph.

- I then investigated how to further optimize the deserialization function. My first step was to transition Jackson from DOM parsing to a more efficient approach. My research into Jackson deserialization revealed two primary methods: `treeToValue()`, which builds a full DOM tree, and `readValue()`, which parses the byte stream directly into tokens and binds to a class via Jackson annotations.

  Moving toward the latter was my first step. With this implementation, the `readValue()` function requires an input such as an array of bytes or a byte stream. In this case, the MQTT message that the Eclipse Paho library provides already gives us a byte array via the `getPayload()` function.

  All we would need to do is annotate the Plain Old Java Objects (POJOs) with the correct Jackson annotations so that Jackson knows how to instantiate the class properly. Essentially, how it works is that after Jackson tokenizes the raw input, it goes through the tokens one by one and uses the annotated POJOs to bind the raw fields into the POJO fields.

  When it came to the `adv` field, we needed to tell Jackson that when it reaches the `type` field within the `adv` object, it should bind to a specific class. Since the POJO is essentially polymorphic, it was a bit difficult to get the annotations right. The first step I needed to do was to create another POJO that essentially models the parent fields of the incoming raw MQTT message. Then, for the `adv` field, I needed to annotate the subclasses properly by implementing an interface, which is allowed when doing Jackson serialization.

```java
@JsonTypeInfo(
        use = JsonTypeInfo.Id.NAME,
        include = JsonTypeInfo.As.EXISTING_PROPERTY,
        property = "type",
        visible = true,
        defaultImpl = Void.class
)
@JsonSubTypes({
        @JsonSubTypes.Type(value = DataPayloadA.class, name = "type_a"),
        @JsonSubTypes.Type(value = DataPayloadB.class, name = "type_b")
})
public interface AdvPayload {
    String type();
}

@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
public record DataPayloadA(
        String type,
        String uuid,
        int major,
        int minor,
        int rssiAtXm,
        int rssi,
        String tm,
        String mac
) implements AdvPayload {}

@Data
public class RawMqttMessageWrapper {
    private String tm;
    private String gw;
    private Long seq;
    private List<AdvPayload> adv;
}

private List<BaseMqttMessage<?>> deserializeMqttMessage(MqttMessage msg) {
    List<BaseMqttMessage<?>> parsedMessages = new ArrayList<>();

    try {
        // Standard MQTT payload extraction
        RawMqttMessageWrapper rawMsg = rawMessageReader.readValue(msg.getPayload());

        // Validation
        if (rawMsg.getAdv() == null || rawMsg.getAdv().isEmpty()) {
            return parsedMessages;
        }

        List<DataPayloadA> groupAList = new ArrayList<>();
        List<DataPayloadB> groupBList = new ArrayList<>();

        // Group the payloads by their resolved concrete type
        for (AdvPayload payload : rawMsg.getAdv()) {
            if (payload instanceof DataPayloadA concreteA) {
                groupAList.add(concreteA);
            } else if (payload instanceof DataPayloadB concreteB) {
                groupBList.add(concreteB);
            }
        }

        // Construct final MQTT message objects using the ConcreteMessage wrapper
        if (!groupAList.isEmpty()) {
            ConcreteMessageA mqttMsgA = new ConcreteMessageA(rawMsg.getTm(), rawMsg.getGw(), rawMsg.getSeq());
            mqttMsgA.setAdv(groupAList);
            parsedMessages.add(mqttMsgA);
        }

        if (!groupBList.isEmpty()) {
            ConcreteMessageB mqttMsgB = new ConcreteMessageB(rawMsg.getTm(), rawMsg.getGw(), rawMsg.getSeq());
            mqttMsgB.setAdv(groupBList);
            parsedMessages.add(mqttMsgB);
        }

    } catch (Exception e) {
        log.error("Error deserializing MQTT message", e);
    }

    return parsedMessages;
}
```

- for this optimization, I managed to get the execution time down to 200ms representing 12.46% of the flame graph. {insert imporved metrics here}.

- At this point, I was still concerned because 400 messages per second were taking up 12.46% of the flame graph. I wanted to see if there was something else I could improve. My next research revealed that we could use the Jackson Streaming API, which processes the raw byte stream directly.

  The difference with this approach is that while the `readValue()` function uses Java reflection to find fields for you, the Jackson Streaming API requires you to manually find and map each field. I had initially avoided this because I knew that while manual mapping is the fastest, it's often the least maintainable. It’s very finicky; if you have a strict structure, a highly optimized streaming API works well, but if the JSON structure is volatile or the shape changes, the deserialization logic will likely break.

  I decided to proceed because I confirmed with my manager that we have a strict JSON structure. I made two primary assumptions: first, that we would always have a known set of fields, and second, that although we know the expected field order, I shouldn't rely on it. Since we are dealing with live IoT devices sending high throughput over the network, fields can arrive jumbled, and I wanted the code to be resilient.

  My idea was to create a buffer for all the fields (or instantiate the variables) as we pass through an entire object. We would find the tokens and work around the known field structure. For primitive fields, we check if the field name is part of our known set and store it. When we reach the `adv` array, we iterate through each entry, buffering the primitive fields within those nested objects and creating the payload objects accordingly.

  ```java
  private List<BaseMqttMessage<?>> deserializeMessage(MqttMessage msg) {
        List<BaseMqttMessage<?>> parsedMessages = new ArrayList<>();

        try (JsonParser parser = jsonFactory.createParser(msg.getPayload())) {

            // Ensure we start with an object, return empty list otherwise
            if (parser.nextToken() != JsonToken.START_OBJECT) {
                return parsedMessages;
            }

            String tm = null;
            String gw = null;
            Long seq = null;

            List<DataPayloadA> payloadAList = new ArrayList<>();
            List<DataPayloadB> payloadBList = new ArrayList<>();

            while (parser.nextToken() != JsonToken.END_OBJECT) {
                String fieldName = parser.getCurrentName();
                parser.nextToken(); // move to the value

                if (MQTTFieldConstant.TM.equals(fieldName)) {
                    tm = parser.getText();
                } else if (MQTTFieldConstant.GW.equals(fieldName)) {
                    gw = parser.getText();
                } else if (MQTTFieldConstant.SEQ.equals(fieldName)) {
                    seq = parser.getLongValue();
                } else if (MQTTFieldConstant.ADV.equals(fieldName)) {

                    // If adv is null or not an array, break out to return the default wrapper
                    if (parser.currentToken() != JsonToken.START_ARRAY) {
                        break;
                    }

                    // Iterate through the array
                    while (parser.nextToken() != JsonToken.END_ARRAY) {
                        if (parser.currentToken() == JsonToken.START_OBJECT) {

                            // Local variables to hold data until we know the type
                            String type = null, uuid = null, mac = null, advTm = null, name = null;
                            int major = 0, minor = 0, rssiAtXm = 0, rssi = 0, battery = 0;

                            // Read the inner object
                            while (parser.nextToken() != JsonToken.END_OBJECT) {
                                String advFieldName = parser.getCurrentName();
                                parser.nextToken(); // move to value

                                switch (advFieldName) {
                                    case MQTTFieldConstant.TYPE -> type = parser.getText();
                                    case MQTTFieldConstant.UUID -> uuid = parser.getText();
                                    case MQTTFieldConstant.MAC -> mac = parser.getText();
                                    case MQTTFieldConstant.TM -> advTm = parser.getText();
                                    case MQTTFieldConstant.NAME -> name = parser.getText();
                                    case MQTTFieldConstant.MAJOR -> major = parser.getIntValue();
                                    case MQTTFieldConstant.MINOR -> minor = parser.getIntValue();
                                    case MQTTFieldConstant.RSSI_AT_XM -> rssiAtXm = parser.getIntValue();
                                    case MQTTFieldConstant.RSSI -> rssi = parser.getIntValue();
                                    case MQTTFieldConstant.BATTERY -> battery = parser.getIntValue();
                                }
                            }

                            // instantiate the appropriate record
                            if (MQTTConstant.TYPE_A_IDENTIFIER.equals(type)) {
                                payloadAList.add(new DataPayloadA(type, uuid, major, minor, rssiAtXm, rssi, advTm, mac));
                            } else if (MQTTConstant.TYPE_B_IDENTIFIER.equals(type)) {
                                payloadBList.add(new DataPayloadB(type, battery, name, rssi, advTm, mac));
                            }
                        }
                    }
                }
            }

            // Construct final messages
            if (!payloadAList.isEmpty()) {
                ConcreteMessageA msgA = new ConcreteMessageA(tm, gw, seq);
                msgA.setAdv(payloadAList);
                parsedMessages.add(msgA);
            }

            if (!payloadBList.isEmpty()) {
                ConcreteMessageB msgB = new ConcreteMessageB(tm, gw, seq);
                msgB.setAdv(payloadBList);
                parsedMessages.add(msgB);
            }

            // Handle empty adv array or unknown types
            if (parsedMessages.isEmpty()) {
                DefaultConcreteMessage defaultMsg = new DefaultConcreteMessage(tm, gw, seq);
                defaultMsg.setAdv(new ArrayList<>());
                parsedMessages.add(defaultMsg);
            }

        } catch (Exception e) {
            log.error("Error streaming message payload", e);
        }

        return parsedMessages;
    }
  ```

  Profiling this optimization showed a massive jump: we achieved 6.55% of the flame graph with 110ms of execution time. This was almost a 2X optimization over the Jackson `readValue` binding approach. However, I wanted to get the execution under 5%.

  I looked into external libraries, thinking Jackson might be at its limit since we were already using token-based mapping. My research led me to DSL-JSON, which is often cited as the fastest JSON library for Java. I followed the API guide and implemented it; it was surprisingly readable and the API was developer-friendly without adding too much complexity.

  DSL-JSON’s advantage over Jackson is that it works similarly to compiled libraries—it uses annotation processing to compile your POJOs, making execution much faster. Instead of relying on reflection at runtime, it uses the compiled mapping code. With DSL-JSON, I got the execution time down to 142ms, representing 8% of the total graph.

  This was a surprise; I expected DSL-JSON to beat my manual Jackson streaming implementation. I investigated why DSL-JSON was slower and discovered that at these deserialization speeds, "CPU churn" (the actual logic processing) matters less because it's a relatively lightweight task.

  The real bottleneck wasn't the CPU processing, but the CPU cycles spent on memory management. The Jackson Streaming API optimization was so fast because it allowed us to utilize stack space more effectively than heap space. This was a great refresher on Java fundamentals.

  At these speeds, the garbage collector becomes much more active, consuming CPU cycles to manage the intermediary DTO objects created by DSL-JSON or other high-level approaches. With the streaming API approach, we buffered primitive variables in stack space, which is blazingly fast compared to heap space. The garbage collector didn't have to destroy unused objects or dereference them, avoiding the overhead present in the other approaches.

Shown is the following graph that compares the execution time per message and single core cpu utilization rate per 400, 4000 and 10,000 mqtt messages/s
![alt text](./images/mqtt_optimization_graph.png)

- after some research a sentence had stuck with me:

  > Designing for high throughput almost always forces you to write code that caters to how the machine wants to read data, rather than how a human wants to read the code.

- Sidenote -- after exhausting the optimized approaches, I was still concerned that my best optimization route used up 6.55% of the flame graph for only 400 messages. Since our expected throughput was 4000 messages per second, I was worried. Looking back, I realized I had misinterpreted the percentage metrics. I thought 6.55% represented the total CPU usage percentage, but I needed to verify if I was reading the profile correctly.

  What actually happened was that I was interpreting the values wrong. The IntelliJ profiler only captures the active time the CPU is being used. Consequently, I had to redo all the profiles. For readability, I have updated the values mentioned above to reflect the correct profiling methodology. In essence, profiling in IntelliJ needs to be treated like any other test: you must control the environment and variables. One of those variables was total execution time, which I had been "winging" during my first few attempts.

  I discovered that the percentage shown wasn't the total CPU utilization of the system; it was just the percentage of active CPU usage during the profiling window. The IntelliJ profiling system wakes up every millisecond or so to capture metrics from the application, extending to hardware-level execution time and memory. The flame graph only shows the active state of the threads. I had assumed that for my 60-second test, the Spring Boot server would be constantly running.

  However, the application is optimized to sleep while waiting for network requests. Since the MQTT service and the deserialization function were the only features being tested, they were the only active workloads in that entire 60-second duration. This gave me a skewed idea of what that percentage meant. It wasn't the CPU utilization rate; it was the percentage of active work being done by the deserialization function within that 60-second period. For example, if the total active execution time was only 1.6 seconds (1600ms) out of the entire profiling duration, the most efficient implementation only took 6% of that active time.

  This meant the function was extremely fast and didn't actually need such aggressive optimization. Even the baseline Jackson DOM approach would have supported tens of thousands of MQTT messages per second. Although the function could already handle the throughput and I had arguably "over-optimized" it, it was still a great learning experience. I gained in-depth knowledge of how the Spring profiler works, the internals of Jackson deserialization, and high-throughput data processing.

  I also improved the system's overall architecture by decoupling the MQTT listener from the business logic and moving to an event-driven architecture. I learned not to make assumptions too quickly; I was initially overly confident in my POJO design, but fortunately, I confirmed the requirements with my manager before going too deep. Had I continued, we would have been instantiating close to 4000 POJOs in memory every second. Even though the performance is now beyond what we strictly need, I still believe this is the best approach—especially regarding garbage collector efficiency. Any approach that prioritizes readability by creating intermediary objects would increase the GC overhead. Moving forward, I’ve learned that in real-time systems, every single line of code matters.
