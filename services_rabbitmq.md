```
  +--------------+         +--------------+         +--------------+
   |  service1    |         |   RabbitMQ   |         |  service2    |
   +--------------+         +--------------+         +--------------+
           |                        |                        |
           |    Sends message       |                        |
           |----------------------->|                        |
           |                        |                        |
           |                        |                        |
           |                        |                        |
           |                        |    Message received    |
           |                        |<-----------------------|
           |                        |                        |
   +--------------+         +--------------+         +--------------+
   |   Python     |         |   RabbitMQ   |         |   Python     |
   |   script     |         |              |         |   script     |
   +--------------+         +--------------+         +--------------+
 ```

# Dockerfiles

### Dockerfile for `service1`:

```Dockerfile
# Use an official Python runtime as a parent image
FROM python:3.9

# Set the working directory in the container
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install any needed packages specified in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Run service1.py when the container launches
CMD ["python", "service1.py"]
```

### Dockerfile for `service2`:

```Dockerfile
# Use an official Python runtime as a parent image
FROM python:3.9

# Set the working directory in the container
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install any needed packages specified in requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Run service2.py when the container launches
CMD ["python", "service2.py"]
```

In both Dockerfiles, `requirements.txt` should contain the necessary Python packages for your services, including `pika` for RabbitMQ communication.

### `service1/requirements.txt`:

```plaintext
pika==1.2.0
```

### `service2/requirements.txt`:

```plaintext
pika==1.2.0
```

These files specify the version of the `pika` package required for RabbitMQ communication. You may need to adjust the version number based on your project requirements or any specific compatibility considerations. To add more dependencies for your actual services, you can list them in these `requirements.txt` files, and they will be installed when building the Docker images.

Assuming you have `service1.py` and `service2.py` as the entry points for your services, you need to modify the CMD accordingly:

### service1.py:

```python
import pika

# Establish a connection to RabbitMQ server
connection = pika.BlockingConnection(pika.ConnectionParameters('rabbitmq'))
channel = connection.channel()

# Declare a queue named 'hello'
channel.queue_declare(queue='hello')

# Send a test message to 'hello' queue
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello from Service 1!')

print(" [x] Sent 'Hello from Service 1!'")

# Close the connection
connection.close()
```

### service2.py:

```python
import pika

# Establish a connection to RabbitMQ server
connection = pika.BlockingConnection(pika.ConnectionParameters('rabbitmq'))
channel = connection.channel()

# Declare a queue named 'hello'
channel.queue_declare(queue='hello')

# Define a callback function to process incoming messages
def callback(ch, method, properties, body):
    print(f" [x] Received {body}")

# Set up the consumer and start consuming messages from 'hello' queue
channel.basic_consume(queue='hello',
                      on_message_callback=callback,
                      auto_ack=True)

print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
```

Make sure to adapt these scripts to fit the logic and functionality of your actual services. Once the Docker images are built and the containers are running, `service2` should receive the message sent by `service1` through RabbitMQ.

# Docker Compose File (`docker-compose.yml`):

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

# Directory Structure:

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

# Verification:

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


