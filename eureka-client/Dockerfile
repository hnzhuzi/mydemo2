FROM harbor.k8s.maimaiti.site/library/openjdk:8-jre-alpine-time
VOLUME /tmp
COPY target/*.jar /app.jar
EXPOSE 8101
CMD ["sh","-c","java ${JVM_OPTS} -jar /app.jar"]
