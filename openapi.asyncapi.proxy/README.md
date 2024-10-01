# openapi.asyncapi.proxy
## Running locally

This example runs using Docker compose. You will find the setup scripts in the [compose](./docker/compose) folder.

### Install kcat client

Requires Kafka client, such as `kcat`.

```bash
brew install kcat
```

### Setup

The `setup.sh` script will:

- Configure Zilla instance

```bash
./compose/setup.sh
```

### Start Kafka Broker

Run the `setup.sh` script in `kafka.broker` [compose](../kafka.broker/docker/compose) folder.

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

### Teardown

The `teardown.sh` script will remove any resources created.

```bash
./compose/teardown.sh
```
