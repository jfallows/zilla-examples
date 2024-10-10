# openapi.asyncapi.proxy
## Running locally

This example runs using Docker compose. You will find the setup scripts in the [compose](./docker/compose) folder.

### Install kcat client

Requires Kafka client, such as `kcat`.

```bash
brew install kcat
```

### Setup

- Start local Kafka broker
- Create requied Kafka topics
- Start local Kafka UI
- Configure Zilla instance

```bash
docker compose up 
```

### Test

#### Create Budget
When sending an HTTP request via Kafka request topic, the request message is sent by Zilla and then Zilla awaits the correlated response message on the response topic to send the HTTP response back to the client.

The following command will wait for the HTTP response.
```bash
curl -X PUT --location 'http://localhost:7114/budget' \
     --header 'Content-Type: application/json' \
     --header 'Idempotency-Key: 1' \
     --data '{"budgetId": "budgetId1", "msgId": "Msg001", "amount": 100}'
```
Now, we need to produce the correlated response so first we need the correlation identifier.
```
kcat -C -b localhost:29092 -t requests -J -u | jq '.headers[-2,-1]'
```
outputs
```
"zilla:correlation-id",
"1-8d8f9e3bef21555a0f8e36619b5abe7c"
```
Use correlation identifier to produce response.
```
echo '{"budgetId": "budgetId1", "msgId": "Msg001", accept: "yes", total: 100}' | \
    kcat -P \
         -b localhost:29092 \
         -t responses \
         -k "1-8d8f9e3bef21555a0f8e36619b5abe7c" \
         -H ":status=200" \
         -H "zilla:correlation-id=1-8d8f9e3bef21555a0f8e36619b5abe7c"
```

#### Reserve Budget
When sending an HTTP request via Kafka request topic, the request message is sent by Zilla and then Zilla awaits the correlated response message on the response topic to send the HTTP response back to the client.

The following command will wait for the HTTP response.
```bash
curl -X PUT --location 'http://localhost:7114/reservation' \
     --header 'Content-Type: application/json' \
     --header 'Idempotency-Key: 1' \
     --data '{"budgetId": "budgetId1", "msgId": "Msg001", "reservations": [ "amount": 60, "amount": 40 ]}'
```
Now, we need to produce the correlated response so first we need the correlation identifier.
```
kcat -C -b localhost:29092 -t requests -J -u | jq '.headers[-2,-1]'
```
outputs
```
"zilla:correlation-id",
"1-46108a33f0174340c245b71274f2d9bb"
```
Use correlation identifier to produce response.
```
echo '{"budgetId": "budgetId1", "msgId": "Msg001", accept: "yes", total: 100}' | \
    kcat -P \
         -b localhost:29092 \
         -t responses \
         -k "1-46108a33f0174340c245b71274f2d9bb" \
         -H ":status=200" \
         -H "zilla:correlation-id=1-46108a33f0174340c245b71274f2d9bb"
```
Then `curl` request completes successfully.

### Test via kafka console client

Install kafka client 3.5.1.
```
wget https://archive.apache.org/dist/kafka/3.5.1/kafka_2.13-3.5.1.tgz
tar xvzf kafka_2.13-3.5.1.tgz
```

Update `client.properties` as needed with `security.protocol`, SASL `username` and `password`.

Check `requests` topic for correlation identifier.
```
bin/kafka-console-consumer.sh --consumer.config ../client.properties \
  --bootstrap-server ${KAFKA_BOOTSTRAP_SERVER} \
  --topic requests --from-beginning \
  --property print.key=true \
  --property print.headers=true
```
```
:scheme:http,:method:PUT,:path:/budget,:authority:localhost:7114,user-agent:curl/8.5.0,accept:*/*,content-type:application/json,idempotency-key:1,zilla:reply-to:responses,zilla:correlation-id:1-c0ec40dc9423e97639d6ace8d215f695    1    {"budgetId": "budgetId1", "msgId": "Msg001", "amount": 100}
```
The correlation identifier in this case is `1-c0ec40dc9423e97639d6ace8d215f695`.

Use the correlation identifier to produce response.
```
echo ':status=200,zilla:correlation-id=1-c0ec40dc9423e97639d6ace8d215f695\t1-c0ec40dc9423e97639d6ace8d215f695\t{"budgetId": "budgetId1", "msgId": "Msg001", accept: "yes", total: 100}' |
bin/kafka-console-producer.sh --producer.config ../client.properties \
  --bootstrap-server ${KAFKA_BOOTSTRAP_SERVER} \
  --topic responses \
  --property parse.key=true \
  --property parse.headers=true \
  --property key.separator="\t" \
  --property headers.key.separator="=" \
  --property headers.separator="," \
  --property headers.delimiter="\t"
```
Then `curl` request completes successfully.