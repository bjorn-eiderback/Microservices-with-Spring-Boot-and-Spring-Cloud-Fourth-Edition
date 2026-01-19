Here’s what `microservices/product-service/Dockerfile` does, line by line:

1. FROM eclipse-temurin:24_36-jre-noble AS builder
   Uses a JRE 24 base image and names this stage builder for the multi-stage build.
2. WORKDIR /extracted
   Sets the working directory inside the builder stage.
3. ADD ./build/libs/*.jar app.jar
   Copies the built Spring Boot jar from the host into the image as /extracted/app.jar.
   This expects you already ran ./gradlew :microservices:product-service:bootJar (or root ./gradlew build) so build/libs/*.jar exists.
4. RUN java -Djarmode=layertools -jar app.jar extract
   Uses Spring Boot’s layer tools to extract the jar into layer directories.
   This creates /extracted/dependencies/, /extracted/spring-boot-loader/, /extracted/snapshot-dependencies/, /extracted/application/.
   Layering helps Docker cache dependencies separately from app code.
5. FROM eclipse-temurin:24_36-jre-noble
   Starts a fresh runtime image (second stage) so the final image doesn’t include build-time artifacts.
6. WORKDIR /application
   Sets the runtime working directory.
7. COPY --from=builder /extracted/dependencies/ ./
   Copies dependency layer into the final image.
8. COPY --from=builder /extracted/spring-boot-loader/ ./
   Copies Spring Boot loader layer.
9. COPY --from=builder /extracted/snapshot-dependencies/ ./
   Copies snapshot dependency layer.
10. COPY --from=builder /extracted//application/ ./
    Copies the application classes/resources layer.
    (Note the double / before application is harmless.)
11. EXPOSE 8080
    Documents that the app listens on port 8080.
12. ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
    Launches the app using Spring Boot’s loader, which knows how to run the layered contents.

Net effect: you get a smaller, cache-friendly image that rebuilds quickly when only app code changes, while dependencies remain cached.
