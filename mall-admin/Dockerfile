FROM eclipse-temurin:17-jdk
ADD target/mall-admin-1.0-SNAPSHOT.jar /mall-admin-1.0-SNAPSHOT.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "-Dspring.profiles.active=prod", "/mall-admin-1.0-SNAPSHOT.jar"]
