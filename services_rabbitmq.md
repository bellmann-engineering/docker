   +--------------+         +--------------+         +--------------+
   |  service1    |         |   RabbitMQ   |         |  service2    |
   +--------------+         +--------------+         +--------------+
           |                        |                        |
           |    Sends message       |                        |
           |----------------------->|                        |
           |                        |                        |
           |                        |                        |
           |                        |                        |
           |                        |    Message received   |
           |                        |<-----------------------|
           |                        |                        |
   +--------------+         +--------------+         +--------------+
   |   Python     |         |   RabbitMQ   |         |   Python     |
   |   script     |         |              |         |   script     |
   +--------------+         +--------------+         +--------------+

### Docker Compose File (`docker-compose.yml`):

```yaml
version: '3'

services:
  service1:
    build:
      context: ./service1
    environment:
      - AMQP_HOST=rabbitmq
      - AMQP_PORT=5672
      - AMQP_USERNAME=guest
      - AMQP_PASSWORD=guest
    depends_on:
      - rabbitmq

  service2:
    build:
      context: ./service2
    environment:
      - AMQP_HOST=rabbitmq
      - AMQP_PORT=5672
      - AMQP_USERNAME=guest
      - AMQP_PASSWORD=guest
    depends_on:
      - rabbitmq

  rabbitmq:
    image: "rabbitmq:3-management"
    environment:
      - RABBITMQ_DEFAULT_USER=guest
      - RABBITMQ_DEFAULT_PASS=guest
    ports:
      - "5672:5672"
      - "15672:15672"
```

### Directory Structure:

```
project/
│
├── docker-compose.yml
│
├── service1/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── service1.py
│
└── service2/
    ├── Dockerfile
    ├── requirements.txt
    └── service2.py
```

### Verification:

1. Save the Docker Compose file and the Dockerfiles in the appropriate directories (`service1` and `service2`).
2. In each service directory, create a `requirements.txt` file with `pika` listed as a dependency.
3. Build the Docker images and start the containers:

   ```bash
   docker-compose up --build -d
   ```

4. Check the logs of `service2` to see if it received the message:

   ```bash
   docker-compose logs service2
   ```

   You should see an output similar to:

   ```
   service2_1  |  [*] Waiting for messages. To exit press CTRL+C
   service2_1  |  [x] Received b'Hello from Service 1!'
   ```

   This indicates that `service2` successfully received the message sent by `service1`.

5. If you want to stop the containers, run:

   ```bash
   docker-compose down
   ```

Remember to adapt the code and configurations based on your actual service logic and requirements. This example is a starting point, and in a production environment, you may need to handle errors, implement retries, and add more features to ensure robust communication between services.
