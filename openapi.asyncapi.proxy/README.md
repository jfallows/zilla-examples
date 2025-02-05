# openapi.asyncapi.proxy
## Running locally

This example uses `docker compose`.

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
curl -X PUT --location 'http://localhost:7114/v1/budget' \
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
"1-72201d10107d556ec88869760cdd7df0"
```
Use correlation identifier to produce response.
```
echo '{"budgetId": "budgetId1", "msgId": "Msg001", accept: "yes", total: 100}' | \
    kcat -P \
         -b localhost:29092 \
         -t responses \
         -k "1-72201d10107d556ec88869760cdd7df0" \
         -H ":status=200" \
         -H "zilla:correlation-id=1-72201d10107d556ec88869760cdd7df0"
```

#### Reserve Budget
When sending an HTTP request via Kafka request topic, the request message is sent by Zilla and then Zilla awaits the correlated response message on the response topic to send the HTTP response back to the client.

The following command will wait for the HTTP response.
```bash
curl -X PUT --location 'http://localhost:7114/v1/reservation' \
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
"1-75b393b1a523731cf030c0489eaa17cc"
```
Use correlation identifier to produce response.
```
echo '{"budgetId": "budgetId1", "msgId": "Msg001", accept: "yes", total: 100}' | \
    kcat -P \
         -b localhost:29092 \
         -t responses \
         -k "1-75b393b1a523731cf030c0489eaa17cc" \
         -H ":status=200" \
         -H "zilla:correlation-id=1-75b393b1a523731cf030c0489eaa17cc"
```
Then `curl` request completes successfully.

### Test via kafka console client

Install kafka client 3.5.1.
```
wget https://archive.apache.org/dist/kafka/3.5.1/kafka_2.13-3.5.1.tgz
tar xvzf kafka_2.13-3.5.1.tgz
```

Check `requests` topic for correlation identifier.
```
kafka_2.13-3.5.1/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:29092 \
  --topic requests --from-beginning \
  --property print.key=true \
  --property print.headers=true
```
```
:scheme:http,:method:PUT,:path:/v1/budget,:authority:localhost:7114,user-agent:curl/8.5.0,accept:*/*,content-type:application/json,idempotency-key:1,zilla:reply-to:responses,zilla:correlation-id:1-72201d10107d556ec88869760cdd7df0    1    {"budgetId": "budgetId1", "msgId": "Msg001", "amount": 100}
```
The correlation identifier in this case is `1-72201d10107d556ec88869760cdd7df0`.

Use the correlation identifier to produce response.
```
echo ':status=200,zilla:correlation-id=1-72201d10107d556ec88869760cdd7df0\t1-72201d10107d556ec88869760cdd7df0\t{"budgetId": "budgetId1", "msgId": "Msg001", accept: "yes", total: 100}' |
kafka_2.13-3.5.1/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:29092 \
  --topic responses \
  --property parse.key=true \
  --property parse.headers=true \
  --property headers.key.separator="="
```
Then `curl` request completes successfully.