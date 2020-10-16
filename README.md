
# Kafka project 



## 2.create Kafka topic

    kafka-topics --create --bootstrap-server ip-172-31-94-165.ec2.internal:9092 --replication-factor 2 --partitions 2 --topic rsvp_ivan

## 3.create Kafka data source

create producer

    kafka-console-producer --broker-list ip-172-31-94-165.ec2.internal:9092 ip-172-31-91-232.ec2.internal:9092 ip-172-31-89-11.ec2.internal:9092 --topic rsvp_ivan

create consumer

    kafka-console-consumer --bootstrap-server ip-172-31-94-165.ec2.internal:9092 --topic rsvp_ivan --group group-ivan

### method 1

load RSVP data

    curl https://stream.meetup.com/2/rsvps | kafka-console-producer --broker-list ip-172-31-94-165.ec2.internal:9092, ip-172-31-91-232.ec2.internal:9092, ip-172-31-89-11.ec2.internal:9092 --topic rsvp_ivan

### method 2

    ./myscript.sh | kafka-console-producer --broker-list ip-172-31-94-165.ec2.internal:9092, ip-172-31-91-232.ec2.internal:9092, ip-172-31-89-11.ec2.internal:9092 --topic rsvp_ivan

myscript.sh [click here](~/kafka_project/myscript.sh)
