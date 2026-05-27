# ==========================================
# ÉTAPE 1 : Compilation et Build avec Maven
# ==========================================
FROM maven:3.9.6-eclipse-temurin-21-alpine AS builder

WORKDIR /build

# Copie du fichier de configuration Maven (pom.xml) du microservice
COPY pom.xml .

# Téléchargement des dépendances pour mettre en cache cette couche Docker
RUN mvn dependency:go-offline -B

# Copie du code source
COPY src ./src

# Compilation et package de l'application (en sautant les tests pour accélérer le build Docker)
RUN mvn clean package -DskipTests

# ==========================================
# ÉTAPE 2 : Image d'exécution (Runtime) minimale
# ==========================================
FROM eclipse-temurin:21-jre-alpine

# Ajout d'un utilisateur non-root pour des raisons de sécurité évidentes
RUN addgroup -S spring && adduser -S spring -G spring
USER spring

WORKDIR /app

# Récupération de l'artefact généré à l'étape précédente
COPY --from=builder /build/target/gateway-service-0.0.1-SNAPSHOT.jar app.jar

# Port exposé par ce service spécifique
EXPOSE 9999

# Optimisations JVM pour les environnements conteneurisés
ENTRYPOINT ["java", "-jar", "-Djava.security.egd=file:/dev/./urandom", "app.jar"]