FROM eclipse-temurin:21-jre-jammy

WORKDIR /reports_service

COPY reports_service-0.0.1-SNAPSHOT.jar reports_service.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "reports_service.jar"]