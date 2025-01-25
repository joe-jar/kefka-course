# Guide Complet : Kafka avec Spring Boot

## Introduction à Kafka

Apache Kafka est une plateforme de messagerie distribuée conçue pour gérer des flux de données en temps réel. Il est utilisé pour la communication entre différentes applications ou microservices. Voici quelques concepts clés de Kafka :

- **Broker** : Serveur Kafka où les messages sont stockés.
- **Topic** : Canal de communication où les messages sont publiés.
- **Partition** : Division logique d'un topic pour améliorer la scalabilité.
- **Producer** : Application qui envoie des messages vers Kafka.
- **Consumer** : Application qui lit des messages depuis Kafka.
- **Consumer Group** : Groupe de consommateurs qui collaborent pour lire les partitions d'un topic.

---

## Objectif

Nous allons créer un système composé de **2 producteurs** et **2 consommateurs** en utilisant **4 microservices** Spring Boot. Chaque microservice interagira avec Kafka pour publier ou consommer des messages. 

### Architecture

1. **Order Service** (producteur) : Publie des commandes dans le topic Kafka.
2. **Payment Service** (producteur) : Publie des paiements dans un autre topic Kafka.
3. **Inventory Service** (consommateur) : Consomme des messages sur les commandes pour mettre à jour l'inventaire.
4. **Notification Service** (consommateur) : Consomme des messages sur les paiements pour notifier l'utilisateur.

---

## Configuration du Kafka dans `application.properties`

Dans chaque application Spring Boot (producteur et consommateur), configurez Kafka comme suit :

```properties
# Configuration commune à tous les microservices
spring.kafka.bootstrap-servers=localhost:9092  # Adresse du broker Kafka

# Configuration Producteur
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer

# Configuration Consommateur
spring.kafka.consumer.group-id=order-consumer-group  # Identifiant du groupe de consommateurs
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
```

---

## Création des Topics

Créez les topics nécessaires à l'aide de la commande Kafka CLI :

1. Pour les commandes :
   ```bash
   kafka-topics.sh --create --topic orders --bootstrap-server localhost:9092 --partitions 3 --replication-factor 1
   ```
2. Pour les paiements :
   ```bash
   kafka-topics.sh --create --topic payments --bootstrap-server localhost:9092 --partitions 2 --replication-factor 1
   ```

---

## Implémentation des Microservices

### 1. Producteur : `Order Service`

Ce service publie des commandes dans le topic `orders` :

```java
@Service
public class OrderProducer {
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    public OrderProducer(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendOrder(String orderId, String orderDetails) {
        kafkaTemplate.send("orders", orderId, orderDetails);
    }
}
```

### 2. Producteur : `Payment Service`

Ce service publie des paiements dans le topic `payments` :

```java
@Service
public class PaymentProducer {
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Autowired
    public PaymentProducer(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendPayment(String paymentId, String paymentDetails) {
        kafkaTemplate.send("payments", paymentId, paymentDetails);
    }
}
```

### 3. Consommateur : `Inventory Service`

Ce service consomme les messages du topic `orders` :

```java
@Service
public class InventoryConsumer {

    @KafkaListener(topics = "orders", groupId = "order-consumer-group")
    public void consumeOrder(String message) {
        System.out.println("Commande reçue pour mise à jour d'inventaire : " + message);
    }
}
```

### 4. Consommateur : `Notification Service`

Ce service consomme les messages du topic `payments` :

```java
@Service
public class NotificationConsumer {

    @KafkaListener(topics = "payments", groupId = "notification-consumer-group")
    public void consumePayment(String message) {
        System.out.println("Paiement reçu pour notification : " + message);
    }
}
```

---

## Partitions et Clés

### Partitions

- **Définition** : Une partition est une division logique d'un topic qui permet de distribuer les messages sur plusieurs brokers.
- **Scalabilité** : Les partitions permettent de répartir la charge entre plusieurs consommateurs.

### Clés

- **Pourquoi les utiliser ?** Les clés garantissent que les messages ayant la même clé vont dans la même partition, préservant ainsi leur ordre.
- **Exemple avec clé :**

```java
kafkaTemplate.send("orders", orderId, "order details");
```

---

## Tableau Comparatif : Kafka vs RabbitMQ

| **Aspect**              | **Kafka**                                           | **RabbitMQ**                               |
|-------------------------|----------------------------------------------------|-------------------------------------------|
| **Modèle**              | Pub/Sub (orienté log)                              | Pub/Sub ou Point-à-Point                  |
| **Stockage**            | Messages stockés durablement                      | Messages supprimés après consommation     |
| **Ordre des messages**  | Garanti par partition                              | Pas garanti entre files                   |
| **Performance**         | Haute performance, adapté aux gros volumes        | Bonne performance pour petits volumes     |
| **Cas d'utilisation**   | Big Data, Analytics, Event Sourcing               | Workflow, microservices classiques        |

---

## Conclusion

- Apache Kafka est idéal pour des systèmes nécessitant une gestion efficace de flux de données massifs.
- Utilisez des **partitions** pour améliorer la scalabilité et des **clés** pour préserver l'ordre.
- Configurez les producteurs et consommateurs dans des microservices séparés pour une architecture claire et modulaire.

---

## Ressources Supplémentaires

- [Documentation Apache Kafka](https://kafka.apache.org/documentation/)
- [Spring Kafka Documentation](https://docs.spring.io/spring-kafka/docs/current/reference/html/)
- [Guide de l'API Kafka CLI](https://kafka.apache.org/quickstart)
