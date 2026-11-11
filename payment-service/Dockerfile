#payment-service/Dockerfile

# Step 1: Use Java 17 as base image
FROM eclipse-temurin:17-jre-alpine

# Step 2: Set working directory inside container
WORKDIR /app

# Step 3: Copy the build jar file
COPY  target/payment-service-0.0.1-SNAPSHOT.jar app.jar

# Step 4: Expose port
EXPOSE 8083

# Step 5: Run the jar
ENTRYPOINT ["java", "-jar", "app.jar"]
