FROM eclipse-temurin:17-jdk
ADD target/mall-search-1.0-SNAPSHOT.jar /mall-search-1.0-SNAPSHOT.jar
EXPOSE 8081
ENTRYPOINT ["java", "-jar", "-Dspring.profiles.active=prod", "/mall-search-1.0-SNAPSHOT.jar"]
