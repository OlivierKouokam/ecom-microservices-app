# ecom-microservices-app — README

**Stack :** Java 21 · Spring Boot 3.5.7 · Spring Cloud 2025.0.0 · H2 in-memory · Eureka · Spring Cloud Gateway · Spring Cloud Config

---

## Architecture & ports

| Service             | Port | Config client | Rôle                       |
|---------------------|------|---------------|----------------------------|
| `config-service`    | 8888 | —             | Spring Cloud Config Server |
| `discovery-service` | 8761 | ✗             | Eureka Server              |
| `gateway-service`   | 9999 | ✗             | Spring Cloud Gateway       |
| `customer-service`  | 8081 | ✓             | API REST clients           |
| `inventory-service` | 8082 | ✓             | API REST produits          |

**Ordre de démarrage obligatoire :** `config` → `discovery` → `gateway` → `customer` → `inventory`

---

## Prérequis — Java 21

```bash
sudo apt update && sudo apt install -y openjdk-21-jdk
echo 'export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64' >> ~/.bashrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
java -version   # openjdk 21
```

---

## Corrections obligatoires avant build/run

### 1. Activer le config client dans customer & inventory

`spring.cloud.config.enabled=false` est défini dans les deux services → la config centralisée ne sera **pas lue**.

```bash
# customer-service
sed -i 's/spring.cloud.config.enabled=false/spring.cloud.config.enabled=true/' \
  ~/TRAINING/ecom-microservices-app/customer-service/src/main/resources/application.properties

# inventory-service
sed -i 's/spring.cloud.config.enabled=false/spring.cloud.config.enabled=true/' \
  ~/TRAINING/ecom-microservices-app/inventory-service/src/main/resources/application.properties
```

### 2. Ajouter le config client au gateway-service

Sans `spring-cloud-starter-config`, le gateway ne lit pas `gateway-service.yml` du config-repo → routes absentes → **404 sur `/customers` et `/products`**.

**`gateway-service/pom.xml`** — ajouter dans `<dependencies>` :
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

**`gateway-service/src/main/resources/application.properties`** — ajouter :
```properties
spring.config.import=optional:configserver:http://localhost:8888
```

### 3. Port gateway (déjà corrigé)

```bash
# Vérifier
grep "server.port" ~/TRAINING/ecom-microservices-app/gateway-service/src/main/resources/application.properties
# → server.port=9999
```

### 3. Initialiser ecom-config-repo comme repo git local

```bash
cd ~/TRAINING/ecom-config-repo
git init
git add .
git commit -m "init config"
```

---

## config-service (à créer)

### Arborescence

```
config-service/
├── pom.xml
├── mvnw  (.mvn/ mvnw.cmd)
└── src/main/
    ├── java/net/youssfi/configservice/
    │   └── ConfigServiceApplication.java
    └── resources/
        └── application.properties
```

### Initialisation

```bash
cd ~/TRAINING/ecom-microservices-app

mkdir -p config-service/src/main/java/net/youssfi/configservice
mkdir -p config-service/src/main/resources

cp customer-service/mvnw     config-service/mvnw
cp customer-service/mvnw.cmd config-service/mvnw.cmd
cp -r customer-service/.mvn  config-service/.mvn
chmod +x config-service/mvnw
```

### `config-service/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.5.7</version>
    <relativePath/>
  </parent>
  <groupId>net.youssfi</groupId>
  <artifactId>config-service</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <properties>
    <java.version>21</java.version>
    <spring-cloud.version>2025.0.0</spring-cloud.version>
  </properties>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
  </dependencies>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>${spring-cloud.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

### `ConfigServiceApplication.java`

```java
package net.youssfi.configservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServiceApplication.class, args);
    }
}
```

### `application.properties`

```properties
server.port=8888
spring.application.name=config-service

# Repo local (recommandé en VM)
spring.cloud.config.server.git.uri=file:///home/vagrant/TRAINING/ecom-config-repo
spring.cloud.config.server.git.default-label=main
spring.cloud.config.server.git.clone-on-start=true

# Alternative repo GitHub distant
# spring.cloud.config.server.git.uri=https://github.com/OlivierKouokam/ecom-config-repo.git

management.endpoints.web.exposure.include=*
```

---

## Build

```bash
cd ~/TRAINING/ecom-microservices-app

for svc in config-service discovery-service gateway-service customer-service inventory-service; do
  echo "=== $svc ==="
  cd $svc && ./mvnw clean package -DskipTests && cd ..
done
```

---

## Run — Option A : `java -jar` (5 terminaux)

```bash
# T1
java -jar ~/TRAINING/ecom-microservices-app/config-service/target/config-service-0.0.1-SNAPSHOT.jar

# T2 — attendre que :8888 soit UP
java -jar ~/TRAINING/ecom-microservices-app/discovery-service/target/discovery-service-0.0.1-SNAPSHOT.jar

# T3
java -jar ~/TRAINING/ecom-microservices-app/gateway-service/target/gateway-service-0.0.1-SNAPSHOT.jar

# T4
java -jar ~/TRAINING/ecom-microservices-app/customer-service/target/customer-service-0.0.1-SNAPSHOT.jar

# T5
java -jar ~/TRAINING/ecom-microservices-app/inventory-service/target/inventory-service-0.0.1-SNAPSHOT.jar
```

**Avec profil dev** (charge `*-dev.properties` → location=France) :

```bash
java -jar target/customer-service-0.0.1-SNAPSHOT.jar --spring.profiles.active=dev
java -jar target/inventory-service-0.0.1-SNAPSHOT.jar --spring.profiles.active=dev
```

---

## Run — Option B : systemd

### Créer les 5 unités

```bash
BASE=/home/vagrant/TRAINING/ecom-microservices-app

for svc in config-service discovery-service gateway-service customer-service inventory-service; do
sudo tee /etc/systemd/system/${svc}.service > /dev/null <<UNIT
[Unit]
Description=${svc}
After=network.target

[Service]
User=vagrant
WorkingDirectory=${BASE}/${svc}
ExecStart=/usr/bin/java -jar ${BASE}/${svc}/target/${svc}-0.0.1-SNAPSHOT.jar
SuccessExitStatus=143
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
UNIT
done

sudo systemctl daemon-reload
```

### Démarrer dans l'ordre

```bash
sudo systemctl enable --now config-service    && sleep 20
sudo systemctl enable --now discovery-service && sleep 20
sudo systemctl enable --now gateway-service   && sleep 10
sudo systemctl enable --now customer-service
sudo systemctl enable --now inventory-service
```

### Commandes utiles

```bash
sudo systemctl status <service>
journalctl -u <service> -f
sudo systemctl restart <service>
sudo systemctl stop <service>
```

---

## Config centralisée — ce que chaque service lit depuis le config-server

| Fichier dans ecom-config-repo       | Consommé par         | Profil    |
|-------------------------------------|----------------------|-----------|
| `application.properties`            | tous les services    | default   |
| `customer-service.properties`       | customer-service     | default   |
| `customer-service-dev.properties`   | customer-service     | dev       |
| `customer-service-prod.properties`  | customer-service     | prod      |
| `inventory-service.properties`      | inventory-service    | default   |
| `inventory-service-dev.properties`  | inventory-service    | dev       |
| `inventory-service-prod.properties` | inventory-service    | prod      |
| `gateway-service.yml`               | gateway-service (*)  | default   |
| `gateway-service-dev.yml`           | gateway-service (*)  | dev       |

> (*) `gateway-service` lit ses routes depuis `gateway-service.yml` du config-repo via le config-server.

---

## Vérifier la config servie (navigateur ou curl)

```bash
# Syntaxe : http://localhost:8888/{application}/{profile}
curl http://localhost:8888/customer-service/default
curl http://localhost:8888/customer-service/dev
curl http://localhost:8888/inventory-service/default
curl http://localhost:8888/gateway-service/default
```

---

## Tests — Navigateur

| URL | But |
|-----|-----|
| `http://localhost:8761` | **Eureka Dashboard** — vérifier les 4 services enregistrés |
| `http://localhost:8888/customer-service/default` | Config brute customer (profil default) |
| `http://localhost:8888/customer-service/dev` | Config brute customer (profil dev) |
| `http://localhost:8888/inventory-service/default` | Config brute inventory |
| `http://localhost:8081/h2-console` | H2 customer · JDBC URL : `jdbc:h2:mem:customers-db` |
| `http://localhost:8082/h2-console` | H2 inventory · JDBC URL : `jdbc:h2:mem:products-db` |
| `http://localhost:9999/customers` | Customers via Gateway |
| `http://localhost:9999/products` | Products via Gateway |

---

## Tests — Postman

### customer-service (via Gateway :9999)

```
GET    http://localhost:9999/customers
GET    http://localhost:9999/customers/{id}

POST   http://localhost:9999/customers
       Content-Type: application/json
       Body: {"name": "Alice", "email": "alice@mail.com"}

PUT    http://localhost:9999/customers/{id}
       Body: {"name": "Alice Updated", "email": "alice@mail.com"}

DELETE http://localhost:9999/customers/{id}
```

### inventory-service (via Gateway :9999)

```
GET    http://localhost:9999/products
GET    http://localhost:9999/products/{id}

POST   http://localhost:9999/products
       Content-Type: application/json
       Body: {"name": "Laptop", "price": 999.99, "quantity": 10}

PUT    http://localhost:9999/products/{id}
DELETE http://localhost:9999/products/{id}
```

### Accès direct (bypass Gateway)

```
GET http://localhost:8081/customers
GET http://localhost:8082/products
```

> Vérifier les chemins exacts avant de tester :
> ```bash
> grep -r "@RestController\|@RequestMapping\|@GetMapping" \
>   ~/TRAINING/ecom-microservices-app/customer-service/src/
> grep -r "@RestController\|@RequestMapping\|@GetMapping" \
>   ~/TRAINING/ecom-microservices-app/inventory-service/src/
> ```

---

## Récapitulatif des actions obligatoires

| # | Action | Statut |
|---|--------|--------|
| 1 | Corriger port gateway `8888` → `9999` | ✅ fait |
| 2 | Activer `spring.cloud.config.enabled=true` dans customer & inventory | ✅ fait |
| 3 | Ajouter `spring-cloud-starter-config` + `spring.config.import` au gateway | ✅ fait |
| 4 | Créer `config-service` (pom + classe + properties) | ✅ fait |
| 5 | `git init && git commit` dans `ecom-config-repo` | ✅ fait |
| 6 | Rebuilder après corrections 1-5 | ✅ fait |