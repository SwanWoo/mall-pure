FROM eclipse-temurin:17-jdk
ADD target/mall-portal-1.0-SNAPSHOT.jar /mall-portal-1.0-SNAPSHOT.jar
EXPOSE 8085
ENTRYPOINT ["java", "-jar", "-Dspring.profiles.active=prod", "/mall-portal-1.0-SNAPSHOT.jar"]
