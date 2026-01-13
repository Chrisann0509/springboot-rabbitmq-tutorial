## Install and Setup RabbitMQ Using Docker

### 1. Pull RabbitMQ Image (with Management UI)

Use the official RabbitMQ image that includes the management plugin:

```bash
docker pull rabbitmq:3-management
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
