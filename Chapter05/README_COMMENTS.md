# Gradle build
Here’s how this repo defines what ./gradlew build builds and how, plus what docker compose build requires/does.

Gradle build configuration (what gets built and how)

- The root settings.gradle defines the multi-project build and included modules: :api, :util, and four services under :microservices:*. That’s
  what ./gradlew build will target from the repo root. See settings.gradle.
- Each included module has its own build.gradle that defines plugins, dependencies, Java toolchain (Java 24), repositories, and tests. See:
    - api/build.gradle (Java + dependency management; WebFlux + springdoc)
    - util/build.gradle (Java + dependency management; depends on :api)
    - microservices/product-service/build.gradle (Spring Boot + dependency management; depends on :api and :util)
    - microservices/recommendation-service/build.gradle
    - microservices/review-service/build.gradle
    - microservices/product-composite-service/build.gradle
- Running ./gradlew build from the root invokes the standard build task for each included project. For Java/Spring Boot projects, build includes
  assemble and check (tests). The services disable the plain jar task and use Spring Boot’s bootJar output instead (via jar { enabled = false } in
  each service build file), so the build produces Spring Boot executable jars in each service’s build/libs.
- Dependencies across modules matter for build order: services depend on :api and :util, and :util depends on :api. Those get compiled first.

## bootJar
Those jar task overrides are in each service’s build.gradle:

- microservices/product-service/build.gradle
- microservices/recommendation-service/build.gradle
- microservices/review-service/build.gradle
- microservices/product-composite-service/build.gradle

Each file contains:

`jar {
enabled = false
}`

That disables the standard jar task so the Spring Boot bootJar is the main artifact.

# Docker compose
docker compose build (requirements and what happens)

- Compose configuration is in `docker-compose.yml`. It defines four services (product, recommendation, review, product-composite), each with build:
  microservices/<service-name>. See docker-compose.yml.
- Requirements:
    - Docker Engine + Docker Compose plugin installed and running.
    - Each service’s Spring Boot jar already built in its local build/libs directory. The Dockerfiles do ADD ./build/libs/*.jar app.jar, so if you
      haven’t run ./gradlew build (or at least ./gradlew :microservices:<svc>:bootJar) first, docker compose build will fail because it can’t find
      the jar.
- What it does:
    - For each service, Compose runs a docker build in that service’s directory using its Dockerfile.
    - The Dockerfiles are multi-stage: they take the built jar, extract Spring Boot layers with java -Djarmode=layertools, then copy those layers
      into the final runtime image (eclipse-temurin:24_36-jre-noble) and set ENTRYPOINT to run the app. See microservices/*/Dockerfile.
    - docker compose build only builds images; it does not start containers. ports/environment in the compose file are used when you run docker
      compose up.

# What does it take to upgrade Java and Spring Boot?
Short version: you’d need to update the Java toolchain, the Spring Boot version in both plugin and BOM, the Docker base images, and ensure your
Gradle wrapper and dependencies support those versions.

Concrete changes in this repo:

- Java toolchain
  Update java { toolchain { languageVersion = JavaLanguageVersion.of(25) } } in:
    - api/build.gradle
    - util/build.gradle
    - microservices/product-service/build.gradle
    - microservices/recommendation-service/build.gradle
    - microservices/review-service/build.gradle
    - microservices/product-composite-service/build.gradle
- Spring Boot version
    - For services (they apply the Boot plugin): change id 'org.springframework.boot' version '3.5.0' to 4.0.1 in each service build.gradle.
    - For api and util: update springBootVersion = '3.5.0' to 4.0.1 so the BOM matches.
- Docker base images
  Update the FROM eclipse-temurin:24_36-jre-noble lines in each service Dockerfile to a Java 25 image tag (whatever you standardize on).
- Gradle wrapper
  Make sure the Gradle version supports Java 25 and Boot 4. If not, update the wrapper in gradle/wrapper/gradle-wrapper.properties. (Boot 4 and
  Java 25 will likely require a newer Gradle; confirm the exact minimum from the Gradle and Spring Boot 4 release notes.)
- Dependencies compatibility
  You’re using org.springdoc:springdoc-openapi-starter-* 2.8.6. Verify it supports Boot 4 (Spring Framework 7). If not, bump to a compatible
  version.
- Runtime/CI
  Ensure your local JDK and any CI images use JDK 25.

(codex says) If you want, I can make the version edits and update the Dockerfiles and wrapper for you.
