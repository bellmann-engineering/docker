```
   +--------------+         +--------------+         +--------------+
   |  Dienst1     |         |   RabbitMQ   |         |  Dienst2     |
   +--------------+         +--------------+         +--------------+
           |                        |                        |
           |    Sendet Nachricht    |                        |
           |----------------------->|                        |
           |                        |                        |
           |                        |                        |
           |                        |                        |
           |                        |    Nachricht erhalten  |
           |                        |<-----------------------|
           |                        |                        |
   +--------------+         +--------------+         +--------------+
   |   Python     |         |   RabbitMQ   |         |   Python     |
   |   Skript     |         |              |         |   Skript     |
   +--------------+         +--------------+         +--------------+

```
# Dockerfiles

### Dockerfile für `service1`:

```Dockerfile
FROM python:3.9
WORKDIR /app
COPY . /app

# Installiere alle benötigten Pakete, die in der requirements.txt angegeben sind
RUN pip install --no-cache-dir -r requirements.txt

# Führe service1.py aus, wenn der Container gestartet wird
CMD ["python", "service1.py"]
```

### Dockerfile für `service2`:

```Dockerfile
FROM python:3.9
WORKDIR /app
COPY . /app

# Installiere alle benötigten Pakete, die in der requirements.txt angegeben sind
RUN pip install --no-cache-dir -r requirements.txt

# Führe service2.py aus, wenn der Container gestartet wird
CMD ["python", "service2.py"]
```

In beiden Dockerfiles sollte `requirements.txt` die erforderlichen Python-Pakete für deine Dienste enthalten, einschließlich `pika` für die RabbitMQ-Kommunikation.

### `service1/requirements.txt`:

```plaintext
pika==1.2.0
```

### `service2/requirements.txt`:

```plaintext
pika==1.2.0
```

Diese Dateien geben die Version des `pika`-Pakets an, das für die RabbitMQ-Kommunikation erforderlich ist. Du musst möglicherweise die Versionsnummer entsprechend deiner Projektanforderungen oder spezifischer Kompatibilitätsüberlegungen anpassen. Um weitere Abhängigkeiten für deine tatsächlichen Dienste hinzuzufügen, kannst du sie in diesen `requirements.txt`-Dateien auflisten, und sie werden bei der Erstellung der Docker-Images installiert.

Angenommen, du hast `service1.py` und `service2.py` als Einstiegspunkte für deine Dienste, musst du den CMD entsprechend anpassen:

### service1.py:

```python
import pika

# Stelle eine Verbindung zum RabbitMQ-Server her
connection = pika.BlockingConnection(pika.ConnectionParameters('rabbitmq'))
channel = connection.channel()

# Deklariere eine Queue namens 'hello'
channel.queue_declare(queue='hello')

# Sende eine Testnachricht an die 'hello'-Queue
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hallo von Dienst 1!')

print(" [x] Sent 'Hallo von Dienst 1!'")

# Schließe die Verbindung
connection.close()
```

### service2.py:

```python
import pika

# Stelle eine Verbindung zum RabbitMQ-Server her
connection = pika.BlockingConnection(pika.ConnectionParameters('rabbitmq'))
channel = connection.channel()

# Deklariere eine Queue namens 'hello'
channel.queue_declare(queue='hello')

# Definiere eine Callback Methode zum Verarbeiten eingehender Nachrichten
def callback(ch, method, properties, body):
    print(f" [x] Received {body}")

# Richte den Consumer ein und beginne mit dem Empfangen von Nachrichten aus der 'hello'-Warteschlange
channel.basic_consume(queue='hello',
                      on_message_callback=callback,
                      auto_ack=True)

print(' [*] Warte auf Nachrichten. Zum Beenden drücke CTRL+C')
channel.start_consuming()
```

Stelle sicher, dass du diese Skripte anpasst, um die Logik und Funktionalität deiner tatsächlichen Dienste zu berücksichtigen. Sobald die Docker-Images erstellt sind und die Container ausgeführt werden, sollte `service2` die von `service1` über RabbitMQ gesendete Nachricht empfangen.

# Docker Compose-Datei (`docker-compose.yml`):

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

# Verzeichnisstruktur:

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

# Test:

1. Speichere die Docker Compose-Datei und die Dockerfiles in den entsprechenden Verzeichnissen (`service1` und `service2`).
2. Erstelle in jedem Dienstverzeichnis eine `requirements.txt`-Datei mit `pika` als Abhängigkeit.
3. Erstelle die Docker-Images und starte die Container:

   ```bash
   docker-compose up --build -d
   ```

4. Überprüfe die Protokolle von `service2`, um festzustellen, ob die Nachricht empfangen wurde:

   ```bash
   docker-compose logs service2
   ```

   Du solltest eine Ausgabe ähnlich wie folgt sehen:

   ```
   service2_1  |  [*] Warte auf Nachrichten. Zum Beenden drücke CTRL+C
   service2_1  |  [x] Received b'Hallo von Dienst 1!'
   ```

   Dies zeigt an, dass `service2` die von `service1` gesendete Nachricht erfolgreich erhalten hat.

5. Um die Contrainer zu stoppen:

   ```bash
   docker-compose down
   ```

# Troubleshooting

## Um auf den RabbitMq Service zu warten:

Eigenes Dockerfile für rabbitmq basierend auf demselben Base-Image:

```
FROM rabbitmq:3-management

# für den Healthcheck
RUN apt-get update
RUN apt-get install -y curl 
```

dann im compose File auf das neue Dockerfile verweisen und einen Healthcheck hinzufügen:
```
  rabbitmq:
    build:
      context: ./rabbitmq
    # image: "rabbitmq:3-management"
    # hostname: rabbitmq
...
    healthcheck:
        test: ["CMD", "curl", "-f", "http://localhost:15672"]
        interval: 30s
        timeout: 10s
        retries: 5
```

In der `docker-compose.yaml` der _beiden_ Services nun `depends_on` um diesen erweitern:

```
  service2:
    container_name: service2
...
    depends_on:
      rabbitmq:
        condition: service_healthy

```
Danach `docker-compose up --build -d`
Mit `docker-compose ps` überprüfen, ob alles startet.

## Um auf der Log-Konsole Ausgaben zu bekommen

Um mittels `docker-compose logs service1` bzw. `docker-compose logs service2` Ausgaben zu bekommen für Python die Option _unbuffered_ aktivieren:

Dockerfile(s):
```
CMD ["python", "-u", "service1.py"]
```
