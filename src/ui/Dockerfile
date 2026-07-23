###############################################################################
# Stage 1 - Build the Application
###############################################################################
#
# Use the official Maven image bundled with Eclipse Temurin JDK 21.
#
# Why?
# - Vendor-neutral (OpenJDK)
# - Includes Maven and Java out of the box
# - No need to manually install dependencies
#
FROM maven:3.9.11-eclipse-temurin-21 AS builder


###############################################################################
# Set the working directory inside the container.
###############################################################################
WORKDIR /workspace


###############################################################################
# Copy only files required to download dependencies.
#
# Docker caches layers. Since pom.xml changes less frequently than source code,
# dependencies will be reused across builds, making subsequent builds faster.
###############################################################################
COPY pom.xml .


###############################################################################
# Download project dependencies.
#
# This creates a dedicated cache layer for Maven dependencies.
###############################################################################
RUN mvn dependency:go-offline -B


###############################################################################
# Copy the application source code.
###############################################################################
COPY src ./src


###############################################################################
# Build the application.
#
# - package       : Generates the executable JAR
# -DskipTests     : Skip tests while building the container
#
###############################################################################
RUN mvn clean package -DskipTests


###############################################################################
# Stage 2 - Runtime Image
###############################################################################
#
# Use the Eclipse Temurin JRE instead of the full JDK.
#
# Benefits:
# - Smaller image
# - Reduced attack surface
# - Faster container startup
#
FROM eclipse-temurin:21-jre


###############################################################################
# OCI Image Metadata
#
# These labels help identify the image in container registries.
###############################################################################
LABEL org.opencontainers.image.title="UI Service"
LABEL org.opencontainers.image.description="Spring Boot UI Application"
LABEL org.opencontainers.image.vendor="Your Organization"
LABEL org.opencontainers.image.licenses="Apache-2.0"


###############################################################################
# Create a dedicated application user.
#
# Running as root inside containers is discouraged.
#
# Even if a container is compromised, the attacker receives only the
# permissions of this unprivileged user.
###############################################################################
RUN groupadd --gid 10001 appgroup && \
    useradd \
        --uid 10001 \
        --gid appgroup \
        --create-home \
        --shell /usr/sbin/nologin \
        appuser


###############################################################################
# Application directory.
###############################################################################
WORKDIR /app


###############################################################################
# Copy the JAR built in Stage 1.
#
# --chown ensures the application user owns the file.
###############################################################################
COPY --from=builder \
     --chown=appuser:appgroup \
     /workspace/target/*.jar \
     app.jar


###############################################################################
# Switch to the non-root user.
###############################################################################
USER appuser


###############################################################################
# Application Configuration
#
# JAVA_OPTS
#     Additional JVM arguments supplied at runtime.
#
# SPRING_PROFILES_ACTIVE
#     Default Spring profile.
#
###############################################################################
ENV JAVA_OPTS=""
ENV SPRING_PROFILES_ACTIVE=prod


###############################################################################
# Inform Docker that the application listens on port 8080.
#
# EXPOSE is documentation only.
# It does NOT publish the port.
###############################################################################
EXPOSE 8080


###############################################################################
# Health Check (Optional but Recommended)
#
# Kubernetes can perform its own probes, but this allows Docker
# to detect unhealthy containers as well.
#
# Requires Spring Boot Actuator.
###############################################################################
HEALTHCHECK --interval=30s \
             --timeout=5s \
             --start-period=60s \
             --retries=3 \
CMD wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1


###############################################################################
# Start the application.
#
# exec replaces the shell process with the JVM process,
# allowing Kubernetes and Docker to correctly handle signals
# such as SIGTERM during graceful shutdown.
###############################################################################
ENTRYPOINT ["sh","-c","exec java $JAVA_OPTS -jar app.jar"]