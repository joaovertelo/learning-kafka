FROM confluentinc/cp-kafka-connect:6.0.1

RUN confluent-hub install confluentinc/kafka-connect-jdbc:latest --no-prompt
RUN confluent-hub install debezium/debezium-connector-postgresql:latest --no-prompt

