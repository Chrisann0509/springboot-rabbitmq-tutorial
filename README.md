## Install and Setup RabbitMQ Using Docker

### 1. Pull RabbitMQ Image (with Management UI)

Use the official RabbitMQ image that includes the management plugin:

```bash
docker pull rabbitmq:4.2.2-management
```

### 2. Run RabbitMQ Container

Start RabbitMQ and expose the required ports:

```bash
docker run --rm -it -p 15672:15672 -p 5672:5672 rabbitmq:4.2.2-management
```

### 3. Verify RabbitMQ Is Running

Check running containers:

```bash
docker ps
```

You should see a container named `rabbitmq` with status **Up**.

### 4. Access RabbitMQ Management UI

Open in your browser:

```
http://localhost:15672
```

**Default credentials**

```
Username: guest
Password: guest
```

> Note: guest/guest is accessible only from localhost.
> 

### 5. (Optional) Create a Custom User

For better security, create an admin user:

```bash
dockerexec -it rabbitmq rabbitmqctl add_user admin admin123
dockerexec -it rabbitmq rabbitmqctl set_user_tags admin administrator
dockerexec -it rabbitmq rabbitmqctl set_permissions -p / admin".*"".*"".*"
```

### 6. (Optional) Enable Data Persistence

Persist RabbitMQ data across container restarts:

```bash
docker run -d --hostname rabbitmq-host --name rabbitmq -p 5672:5672 -p 15672:15672 -v rabbitmq_data:/var/lib/rabbitmq rabbitmq:4.2.2-management
```

### 7. Stop / Start RabbitMQ

```bash
docker stop rabbitmq
docker start rabbitmq
```
Remove container:

```bash
docker rm -f rabbitmq
```

# Spring Boot with RabbitMQ

### 1. Connection Between Spring Boot and RabbitMQ

Step 1: Add Dependency

Add the following dependency to your **`pom.xml`**:

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

Step 2: Configure Application Properties

Add RabbitMQ configuration to **`application.properties`**:

```
spring.application.name=springboot-rabbitmq-tutorial
spring.rabbitmq.host=localhost
spring.rabbitmq.port=5672
spring.rabbitmq.username=guest
spring.rabbitmq.password=guest
```

### Notes

- Port **5672** is the default RabbitMQ broker port
- Make sure RabbitMQ is running before starting the Spring Boot application
- `spring-boot-starter-amqp` provides support for:
    - RabbitTemplate
    - Message listeners
    - Exchange, queue, and binding abstractions

### 2. Queue & Exchange & Routing Key Configuration

Create a configuration class to define your queue, exchange, and bindings:

```java
@Configuration
public class RabbitMQConfig {

    @Value("${rabbitmq.exchange.name}")
    private String exchange;

    @Value("${rabbitmq.routing.key}")
    private String routingKey;

    @Value("${rabbitmq.json.queue.name}")
    private String jsonQueue;

    @Value("${rabbitmq.json.routing.key}")
    private String jsonRoutingKey;

    @Bean
    public Queue jsonQueue() {
        return new Queue(jsonQueue);
    }

    // spring bean for rabbitmq exchange
    @Bean
    public TopicExchange exchange() {
        return new TopicExchange(exchange);
    }

    @Bean
    public Binding jsonBinding() {
        return BindingBuilder.bind(jsonQueue())
                .to(exchange())
                .with(jsonRoutingKey);
    }
}

```

- This config ensures the queue is created on startup.
- You can add `DirectExchange`, `TopicExchange`, and `Binding` beans depending on routing needs.

### 3. Producer (Sending Messages)

Create a **sender service** that uses `RabbitTemplate`:

```java
@Service
public class RabbitMQJsonProducer {
    @Value("${rabbitmq.exchange.name}")
    private String exchange;
    @Value("${rabbitmq.json.routing.key}")
    private String routingJsonKey;

    private RabbitTemplate rabbitTemplate;
    private static final Logger LOGGER = LoggerFactory.getLogger(RabbitMQJsonProducer.class);

    public RabbitMQJsonProducer(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public void sendJsonMessage(User user) {
        LOGGER.info(String.format("Json message sent -> %s", user.toString()));
        rabbitTemplate.convertAndSend(exchange, routingJsonKey, user);
    }
}
```

- `RabbitTemplate` automatically connects to RabbitMQ using your properties.
- You can send any serializable POJO; make sure you configure a message converter for JSON (optional but recommended).

### 4. Listener (Receiving Messages)

Create a listener that processes incoming messages:

```java
@Service
public class RabbitMQJsonConsumer {
    private static final Logger LOGGER = LoggerFactory.getLogger(RabbitMQJsonProducer.class);

    @RabbitListener(queues = "${rabbitmq.json.queue.name}")
    public void consumerJsonMessage(User user) {
        LOGGER.info(String.format("Received Json message -> %s", user.toString()));
    }
}
```

- `@RabbitListener` registers the method as a message consumer.
- Messages arriving on the queue will be deserialized into the `Person` object.

### 5. Model Class

Example of a simple **Person** model:

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User {
    private int id;
    private String firstName;
    private String lastName;
}
```

- Ensure your producer and listener use the same model class for serialization/deserialization.

### 6. REST API to Send Message

Create a **REST controller** to send a message via HTTP POST:

```java
@RestController
@RequestMapping("/api/v1")
public class MessageJsonController {
    private RabbitMQJsonProducer jsonProducer;

    public MessageJsonController(RabbitMQJsonProducer jsonProducer) {
        this.jsonProducer = jsonProducer;
    }

    @PostMapping("/publish")
    public ResponseEntity<String> sendJsonMessage(@RequestBody User user) {
        jsonProducer.sendJsonMessage(user);
        return ResponseEntity.ok("Json messages sent to RabbitMQ.");
    }
}

```

### 7. (Optional) JSON Message Converter

To send/receive JSON messages instead of default Java serialization:

```java
@Configuration
public class RabbitMQConfig {

    @Bean
    public Jackson2JsonMessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public AmqpTemplate amqpTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMessageConverter(converter());
        return rabbitTemplate;
    }
}
```

- This allows automatic JSON conversion for payloads.

### 8. Running & Testing

1. **Start RabbitMQ** (via Docker or native).
2. **Run your Spring Boot app**.
3. **Send a message** via REST endpoint or main method invoking the sender.
4. **Observe the listener log** for received message output.

### 9. Tips & Best Practices

- Use the RabbitMQ **Management UI** (`http://localhost:15672`) to monitor queues, exchanges, and messages.
- For production, configure **security users**, persistent queues, and durable exchanges.
- Consider using **exchanges** (direct, topic, fanout) for more advanc
