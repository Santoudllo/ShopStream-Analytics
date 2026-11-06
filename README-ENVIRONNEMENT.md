# ShopStream Analytics — environnement Docker

## Prérequis

- Docker Desktop (Windows/macOS) ou Docker Engine + Docker Compose (Linux)
- Au moins 4 Go de RAM disponibles pour Docker ; 6 Go ou plus recommandés
- Ports locaux disponibles : 29092, 7077, 8080 et 8081

## Démarrage

Dans le dossier contenant `docker-compose.yml` :

```bash
docker compose up -d
docker compose ps
```

Attendre que Kafka soit indiqué comme `healthy` :

```bash
docker compose ps
docker compose logs -f kafka
```

## Interfaces

- Kafka : accessible depuis les conteneurs à `kafka:9092`
- Kafka : accessible depuis l'ordinateur hôte à `localhost:29092`
- Spark Master UI : http://localhost:8080
- Spark Worker UI : http://localhost:8081

## Créer le Topic

Depuis le dossier du projet :

```bash
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server kafka:9092 \
  --create --if-not-exists \
  --topic shopstream-products \
  --partitions 3 \
  --replication-factor 1
```

Vérifier le Topic :

```bash
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server kafka:9092 --list
```

## Tester Kafka

Ouvrir un terminal consommateur :

```bash
docker compose exec kafka /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server kafka:9092 \
  --topic shopstream-products \
  --from-beginning
```

Dans un deuxième terminal, publier un message :

```bash
docker compose exec kafka /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server kafka:9092 \
  --topic shopstream-products
```

Saisir un message JSON, puis appuyer sur Entrée :

```json
{"id":1,"title":"Product example","category":"beauty","price":9.99,"rating":4.5,"stock":50}
```

Le message doit apparaître dans le terminal consommateur.

## Utiliser Spark

Les applications et résultats sont partagés avec les conteneurs via :

- `./app` → `/opt/spark-apps`
- `./output` → `/opt/spark-output`

Créer ces dossiers avant de lancer le projet si nécessaire :

```bash
mkdir -p app output
```

Pour lancer une application Spark depuis le conteneur Master, le connecteur Kafka doit correspondre exactement à la version de Spark et à Scala. Pour Spark 3.5.6 / Scala 2.12 :

```bash
docker compose exec spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.5.6 \
  /opt/spark-apps/spark_streaming.py
```

Dans le code Spark, utiliser `kafka:9092` comme adresse du broker lorsque l'application s'exécute dans le conteneur Spark.

## Arrêt

Arrêter les services sans supprimer les données Kafka :

```bash
docker compose down
```

Réinitialiser complètement l'environnement et supprimer les données persistées de Kafka :

```bash
docker compose down -v
```

## Important

- Cette configuration est un environnement pédagogique à un seul broker Kafka et un seul Worker Spark ; elle ne constitue pas un cluster hautement disponible.
- Le facteur de réplication Kafka est fixé à 1 car il n'y a qu'un broker.
- Depuis un script exécuté sur l'ordinateur hôte, utiliser `localhost:29092`. Depuis les conteneurs, utiliser `kafka:9092`.
- Le téléchargement initial des images Docker et du connecteur Spark nécessite une connexion Internet.
