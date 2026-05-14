FROM maven:3.9-eclipse-temurin-17 AS builder

WORKDIR /app

COPY pom.xml .

RUN mvn dependency:go-offline

COPY src ./src

RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

RUN addgroup -S devops && adduser -S devops -G devops

COPY --from=builder /app/target/*.jar app.jar

RUN chown devops:devops app.jar

USER devops

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]