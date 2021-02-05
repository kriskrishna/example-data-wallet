# BigchainDB + React + Redux boilerplate 

## Clone
Clone or fork this repo

```bash
git clone git@github.com:bigchaindb/bigchaindb-react-redux-boilerplate.git my-bigchaindb-project 
```

and

```bash
cd my-bigchaindb-project
```

Now you can set your remotes to your local app and so forth

## Quickstart with Docker (Windows, OSX, lazy Linux)

> Supports BigchainDB Server v1.0

### Prequisites

You must have `docker`, `docker-compose` (and `make`) installed.
These versions or higher should work:

- `docker`: `v1.13.0`
- `docker-compose`: `v1.7.1`

### Make or docker-compose

To spin up the services, simple run the make command, which will orchestrate `docker-compose`

```bash
make
```

This might take a few minutes, perfect moment for a :coffee:!

Once docker-compose has built and launched all services, have a look:

```bash
docker-compose ps
```

```
            Name                          Command               State                        Ports                       
------------------------------------------------------------------------------------------------------------------------
mybigchaindbproject_bdb_1      bigchaindb start                 Up      0.0.0.0:49984->9984/tcp, 0.0.0.0:49985->9985/tcp 
mybigchaindbproject_client_1   npm start                        Up      0.0.0.0:3000->3000/tcp   
mybigchaindbproject_mdb_1      docker-entrypoint.sh mongo ...   Up      0.0.0.0:32797->27017/tcp                        
```

Which means that the internal docker port for the API is `9984` 
and the external one is `49984`.

The external ports might change, so for the following use the ports as indicated by `docker-compose ps`.

You can simply check if it's running by going to [`http://localhost:3000`](http://localhost:3000).

If you already built the images and want to `restart`:

```bash
make restart
```

Stop (and remove) the containers with

```bash
make stop
```

### Launch docker-compose services manually

No make? Launch the services manually:

Launch MongoDB:

```bash
docker-compose up -d mdb
```

Wait about 10 seconds and then launch the server & client:

```bash
docker-compose up -d bdb
docker-compose up -d client
```

## BigchainDB JavaScript Driver

see the [js-bigchaindb-driver](https://github.com/bigchaindb/js-bigchaindb-driver) for more details

KAFKA

    /**
     * Listen to transactions topic
     *
     * @param transactions JSON list of transactions
     * @param partition
     * @param topic
     * @param offset
     * @param ts
     */
    @KafkaListener(topics = "#{" + KafkaListenerConfig.CONSUMER_PROPS + ".topic}", groupId = "#{"
        + KafkaListenerConfig.CONSUMER_PROPS + ".group}",
        containerFactory = KafkaListenerConfig.CONSUMER_CONTAINER_FACTORY)
    public void listen(@Payload List<String> transactions,
            @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic, @Header(KafkaHeaders.OFFSET) Long offset,
            @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long ts) {
        logger.debug("Incoming records [{}] [{}:{}]", topic, partition, offset);

        if (CollectionUtils.isEmpty(transactions)) {
            logger.debug("Skipping blank transactions [{}] [{}:{}]", topic, partition, offset);
            return;
        }

        try {
            logger.debug("Start processing {} transaction.", transactions.size());

            transactions.parallelStream().forEach(
                this::processTransaction
            );

        } catch (RuntimeException e) {
            exceptionHandler.handleSystemError(e, null);
        }
    }

    /**
     * Process transaction and handle exceptions.
     *
     * @param jsonTrans
     * @param counter
     * @param transSize
     */
    @SuppressWarnings("squid:S1181") /* catching Error because of MongoDB dynamic class loading issues */
    private void processTransaction(final String jsonTrans) {
        String transactionId = null;
        Transaction transaction = null;

        try {
            transaction = modelJsonConverter.toTransaction(jsonTrans);
            transactionId = transaction.getId();
            logger.info("Bulk processing trans [{}] - started.", transactionId);

            ewsClientService.accountValidation(transaction);
            logger.info("Bulk processing trans [{}] - finished.", transactionId);

        } catch(ClientNotFoundException cnfe) {
            exceptionHandler.handle(cnfe, transaction);
        } catch(ClientAuthorizationException cae) {
            exceptionHandler.handle(cae, transaction);
        } catch(DataValidationException dve) {
            exceptionHandler.handle(dve, transaction);
        } catch (RuntimeException | Error e) { /* e.g. MongoDB NoClassDefFoundError */
            exceptionHandler.handleSystemError(e, transactionId);
        }
    }
