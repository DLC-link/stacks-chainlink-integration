# Stacks - Chainlink Integration (DRM)

This repo contains all the parts for integrating a [Chainlink](https://chain.link/) node with the [Stacks chain](https://www.stacks.co/) with the Direct Request Model (DRM).

DRM means that you will be able to request data from your smart contract from any API via the Chainlink Oracles and get it back in a callback. This readme describes how the Chainlink Node and Stacks integration works and how to run your own node.

If you are interested in how to request data in your smart contracts check out [this repository](https://github.com/DLC-link/stacks-chainlink-documentation).

## Requirements

To run the integration you need:
 - Docker v19.X
 - NodeJS v12.X

This guide focuses on running the code on Linux or MacOS. If you are using Windows you may need further configuration.

## Run with Docker

### Oracle listener

The easiest way to get started is to run all the required components with the provided Docker environment.

1. Install the correct version of Docker and NodeJS (with NPM).
2. Open `.env` in the project root and set the environment variable `CREATE_SAMPLE_JOBS=true` to create sample get and post jobs.
3. `npm run docker:start --enable_oracle_listener`

This will start a Chainlink Node, the Event Observer server and a Postgres database. In oracle listener mode the event observer listens for calls on the stacks testnet API. This is the recommended way on testnet.

### With Stacks node

If you don't pass the `--enable_oracle_listener` flag then additional containers for a stack node are started alongside the rest. This way you need to wait until stacks node syncs itself which can take hours.

If you use this mode then set `STACKS_CORE_API_URL=http://localhost:3999` in the `.env` file.

## Managing the Chainlink node

If you want to create your custom job you can start the Chainlink node using the `(a)` command and go to `localhost:6688/signin`. Login credentials: `test@tintash.com // 12345678`.

Click on the `Jobs` tab and then hit `New Job` button to create the job. When you create your job, you'll get the job id. You have to paste the job id in `.env` as `CHAINLINK_GET_JOB_ID` or `CHAINLINK_POST_JOB_ID` depending on whether you create job with `get` or `post` task.

## Contract calls

Contracts are deployed on following addresses on stacks testnet
```
stxlink-token:  ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.stxlink-token 
oracle:         ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.oracle
---
direct-request: ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM.direct-request
```

The `direct-request` contract is an example consumer contract that you can use to test the oracle.

### Making a contract-call to `create-request` function of  `direct-request.clar` contract
You need to pass the `jod-spec-id buffer` , and `data-buffer` in `create-request` function of the `direct-request` contract.

The `direct-request` contract will make a `transfer-and-call` call to the oracle contract using the `stxlink-token`. The event emitted by the oracle contract is captured by our Event Observer Server, which will initiate our chainlink-job. On successful run of the job, `request fulfillment` is created and the expected data is received back in the `direct-request` contract.

To create a request with test data for getting ether price use following command  
`curl -X GET 'http://localhost:3501/consumer-test/?id=0'`

This will return the transaction id which you can track on [stacks explorer](https://explorer.stacks.co/?chain=testnet).To check the direct request result you should wait for transaction confirmation.

### Reading the response

You can call the `read-data-value` function to get the response that is stored in the `data-value` variable in the `direct-request` contract. 

```
curl -X POST https://stacks-node-api.testnet.stacks.co/v2/contracts/call-read/ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM/direct-request/read-data-value \
--header 'Content-Type: application/json' \
--data-raw '{
"sender": "ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM",
"arguments":[] }'
```

This will give you a hex encoded response.

## Testing DRM On Mocknet

You will need [Clarinet](https://docs.hiro.so/smart-contracts/clarinet) v0.15.X to deploy contracts on your mocknet.

In order to run the system on mocknet, you have to set the following variables in the `.env` file:
`STACKS_NETWORK=0`. Then run the command `npm run docker:start --stacks_network=mocknet`.  

To enable the oracle listener mode you need to pass the `--enable_oracle_listener` flag as well.

Once you&#39;ve followed the above steps, you need to deploy the smart contracts on to your local testnet. In the cloned repo navigate to `contracts/clarity` folder and deploy the contracts by running: `clarinet publish --devnet`

### Making a contract-call to `create-request` function of  `direct-request.clar` contract

You need to pass the `jod-spec-id buffer` , and `data-buffer` in `create-request` function of the `direct-request` contract.

The `direct-request` contract will make a `transfer-and-call` call to the oracle contract using the `stxlink-token`. The event emitted by the oracle contract is captured by our Event Observer Server, which will initiate our chainlink-job. On successful run of the job, `request fulfillment` is created and the expected data is received back in the `direct-request` contract.

To create a request with test data for getting ether price use following command

`curl -X GET 'http://localhost:3501/consumer-test/?id=0'`

This will return the transaction id which you can track on http://localhost:3999/extended/v1/tx/{txid}

### Reading the response

You can call the `read-data-value` function to get the response that is stored in the `data-value` variable in the `direct-request` contract.

```
curl -X POST http://localhost:3999/v2/contracts/call-read/ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM/direct-request/read-data-value \
--header 'Content-Type: application/json' \
--data-raw '{
"sender": "ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM",
"arguments":[] }'
```

This will give you a hex encoded response.

## Future work

This project enables the use of Chainlink Oracles on Stacks but it only supports the Direct Request Model.
To enable additional request models (Decentrailzed Data Mode, Off-Chain reporting) you will need to update the Chainlink Node to the latest version. 

This will require porting the jobs from V1 to V2 (toml) spec. Note that direct conversion is not possible as TOML jobs require specific URLs for jobs and the current JSON jobs are generic. Futhermore to support additional request models you will need to create smart contracts if you want to have on-chain aggregation. Or update the event observer server to handle additional job types.

You will also need to rewrite the communication between the event observer and the Chainlink node. Mainly around job creation and job usage.

An other important factor that is missing is payment to the node runners. We have STXLINK tokens on Stacks but currently there is no way to actually pay the Chainlink Nodes who run the requests.

## Addendum: Build From Source

Building from source involves running Chainlink node stacks node stacks Api event observer server and postgres on your machine in natively without containers.

### Setting up Chainlink node on your system

#### Chainlink node setup

Please clone [this](https://github.com/zeeshanakram27/chainlink/tree/feat/stacks-chainlink-integration-ui-changes) repo and checkout to `stacks-chainlink-integration-ui-changes` branch, and refer to instructions to setup the Chainlink node.

In the configuration (`.env`) of Chainlink node add the following flag to enable external-initiator feature.

`FEATURE_EXTERNAL_INITIATORS=true`

Also add your Stacks address, that will be used to create transactions by External Adapter(EA) and its `stx` balance will be visible on Chainlink node dashboard, and Stacks chain/node url to `.env` file.

`STACKS_ACCOUNT_ADDRESS=ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM`

`STACKS_NODE_URL=http://localhost:3999`

#### Setup POSTGRES database

To run Chainlink node you would require a running instance of POSTGRES server with a database named Chainlink already created in it.

Install Postgres

`brew install postgresql`

Login to Postgres server

`psql -U postgres -h localhost`

Create database

`create database chainlink;`

Add `DATABASE\_URL` to `.env` file

`DATABASE_URL=postgresql://{USERNAME}:{PASSWORD}@localhost:5432/chainlink?sslmode=disable`

### Running Chainlink node

Build the application.

`make install`

Run the Chainlink node.

`./chainlink local node --password password.txt --api apicredentials.txt`

Once you have set up the Chainlink node and tested it works fine, it&#39;s time to run the **stacks-blockchain-api** on our system.

### Running stacks-blockchain-api on our system

First clone the repository by using the following command.

`git clone https://github.com/blockstack/stacks-blockchain-api`

and build the application.

`cd stacks-blockchain-api`

##### `Stacks-dev.toml` file

In the `stacks-blockchain-api` directory add the following lines in the `Stacks-dev.toml` file.

``` 
[[mstx_balance]]
address = "ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM"
amount = 10000000000000000
# secret_key: 753b7cc01a1a2e86221266a154af739463fce51219d97e4f856cd7200c3bd2a601
# mnemonic: "twice kind fence tip hidden tilt action fragile skin nothing glory cousin green tomorrow spring wrist shed math olympic multiply hip blue scout claw"


[[events_observer]]
endpoint = "host.docker.internal:3501"
retry_count = 255
events_keys = ["*"]
```

#### Setup services

Build the services by using the following command.

`npm run devenv:build`

Then run `npm run devenv:deploy` which uses docker-compose to deploy the service dependencies (e.g. PostgreSQL, Blockstack core node, etc).

#### Running the server

Then run the server by using the following command.

`npm run dev`

### Setting up Event Observer Server

Clone the following repository.

[https://bitbucket.org/tintash/chainlink-integration/src/master/](https://bitbucket.org/tintash/chainlink-integration/src/master/)

#### Add Chainlink node related configuration

Add following configuration flags in `.env` file.

`CHAINLINK_EI_NAME = {EXTERNAL_INITIATOR_NAME}`

`CHAINLINK_EI_URL = http://localhost:3501`

`CHAINLINK_BRIDGE_NAME = {EXTERNAL_ADAPTER_NAME}`

`CHAINLINK_BRIDGE_URL = http://localhost:3501/adapter`

Set `CONFIGURE\_CHAINLINK` to `true` to automate the process of creating External Initiator(EI), and External Adapter(EA) of provided names if they don&#39;t exist already. Otherwise manually create them.

Set `CREATE\_SAMPLE\_JOBS` to true to create sample Get and Post jobs
```
CONFIGURE_CHAINLINK = false
CREATE_SAMPLE_JOBS = false
```

Then add following credential env vars, these will be automatically replaced by the credentials of newly created EI.
```
EI_IC_ACCESSKEY=""
EI_IC_SECRET=""
EI_CI_ACCESSKEY=""
EI_CI_SECRET=""
```

#### Running the server

Then run the server by using the following command.

`npm run start`

### Deploying the smart contracts and running the server

Once you&#39;ve followed the above steps, we need to deploy the smart contracts by using the following instructions.

In the Event Observer Server navigate to `contracts/clarity` folder and deploy the contracts:

`clarinet publish --devnet`

Start the Event Observer Server:

`npm run start`

(you may need to run `npm install` before the above command).

#### Making a contract-call to `create-request` function of  `direct-request.clar` contract

You need to pass the `jod-spec-id buffer` , and `data-buffer` in `create-request` function of the `direct-request` contract.

The `direct-request` contract will make a `transfer-and-call` call to the oracle contract using the `stxlink-token`. The event emitted by the oracle contract is captured by our Event Observer Server, which will initiate our chainlink-job. On successful run of the job, `request fulfillment` is created and the expected data is received back in the `direct-request` contract.

To create a request with test data for getting ether price use following command

`curl -X GET 'http://localhost:3501/consumer-test/?id=0'`

This will return the transaction id which you can track on http://localhost:3999/extended/v1/tx/{txid}

#### Reading the response

Once the create-request transaction you are tracking is confirmed.

You can call the `read-data-value` function to get the response that is stored in the `data-value` variable in the `direct-request` contract.

```
curl -X POST http://localhost:3999/v2/contracts/call-read/ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM/direct-request/read-data-value \
--header 'Content-Type: application/json' \
--data-raw '{
"sender": "ST1PQHQKV0RJXZFY1DGX8MNSNYVE3VGZJSRTPGZGM",
"arguments":[] }'
```

This command will give the response in the form of `hex`. You can decode it in the `string`


## Visual representation of complete DRM workflow

![](https://mermaid.ink/img/eyJjb2RlIjoic2VxdWVuY2VEaWFncmFtXG4gICAgYXV0b251bWJlclxuICAgIHBhcnRpY2lwYW50IEMgYXMgRGlyZWN0IFJlcXVlc3QgQ29udHJhY3RcbiAgICBwYXJ0aWNpcGFudCBMIGFzIFNUWC1MSU5LIFRva2VuIENvbnRyYWN0XG4gICAgcGFydGljaXBhbnQgTyBhcyBPcmFjbGUgQ29udHJhY3RcbiAgICBwYXJ0aWNpcGFudCBFSSBhcyBFeHRlcm5hbCBJbml0aWF0b3IoRUkpXG4gICAgcGFydGljaXBhbnQgRUEgYXMgRXh0ZXJuYWwgQWRhcHRlcihFQSlcbiAgICBwYXJ0aWNpcGFudCBOIGFzIENoYWlubGluayBOb2RlXG4gICAgTm90ZSBvdmVyIEVJLCBFQTogIzgxOTU7IzgxOTU7IzgxOTU7IzgxOTU7IEV2ZW50IE9ic2VydmVyIFNlcnZlciAjODE5NTsjODE5NTsjODE5NTsjODE5NTtcbiAgICBDLT4-QzogY3JlYXRlLXJlcXVlc3QgKGpvYi1zcGVjLWlkLCBzZW5kZXItaWQtYnVmZiwgZGF0YSwgY2FsbGJhY2spXG4gICAgTm90ZSBvdmVyIEM6IFwiY3JlYXRlLXJlcXVlc3RcIiBpbnRlcm5hbGx5IG1ha2VzIGNvbnRyYWN0IDxicj4gY2FsbCB0byBzdHgtbGluayAodHJhbnNmZXItYW5kLWNhbGwpIFxuICAgIHJlY3QgcmdiYSgyMjUsIDAsIDI1NSwgLjEpXG4gICAgTm90ZSBvdmVyIEMsTzogSU5URVJOQUwgVFJBTlNBQ1RJT05TXG4gICAgQy0-Pkw6IHRyYW5zZmVyLWFuZC1jYWxsIChqb2Itc3BlYy1pZCwgc2VuZGVyLWlkLWJ1ZmYsIGRhdGEsIGNhbGxiYWNrKVxuICAgIE5vdGUgb3ZlciBMOiBvbiBzdWNjZXNzZnVsIHRyYW5zZmVyIG1ha2VzIGNvbnRyYWN0IDxicj4gY2FsbCB0byBvcmFjbGUncyAob3JhY2xlLXJlcXVlc3QpXG4gICAgTC0-Pk86IG9yYWNsZS1yZXF1ZXN0IChzZW5kZXIsIHNwZWMtaWQsIHNlbmRlci1pZC1idWZmLCBjYWxsYmFjaywgcGF5bWVudCAuLi4pO1xuICAgIE5vdGUgb3ZlciBPOiBvcmFjbGUgcmVxdWVzdCBlbWl0cyBldmVudHMgdGhhdCBpcyBwaWNrZWQgPGJyPiB1cCBieSBleHRlcm5hbCBpbml0aWF0b3Igb2YgY2hhaW5saW5rIG5vZGVcbiAgICBlbmRcbiAgICBPLT4-RUk6ICM4MTk1OyM4MTk1OyBFbWl0cyBldmVudCAjODE5NTsjODE5NTtcbiAgICBFSS0-Pk46ICM4MTk1OyB2ZXJpZmllcyBzdHgtbGluayBwYXltZW50IGFnYWluc3Qgam9iIGNvc3QgYW5kIHBhc3NlcyByZXF1ZXN0IHRvIGNoYWlubGluay1ub2RlXG4gICAgTi0-PkVBOiAjODE5NTsgUnVucyBjb3JlIHRhc2tzIGFuZCBwYXNzIHRvIEVBXG4gICAgRUEtPj5POiAgZnVsbGZpbGwtb3JhY2xlLXJlcXVlc3QocmVxdWVzdC1pZCwgY2FsbGJhY2ssIGV4cGlyYXRpb24scmVxLWNvdW50IC4uLilcbiAgICByZWN0IHJnYmEoMCwgMjI1LCAyNTUsIC4xKVxuICAgIE5vdGUgb3ZlciBMOiBJTlRFUk5BTCBUUkFOU0FDVElPTlxuICAgIE8tPj5DOiBvcmFjbGUtY2FsbGJhY2staGFuZGxlcih2YWx1ZSlcbiAgICBlbmRcbiIsIm1lcm1haWQiOnsidGhlbWUiOiJkZWZhdWx0In0sInVwZGF0ZUVkaXRvciI6dHJ1ZSwiYXV0b1N5bmMiOnRydWUsInVwZGF0ZURpYWdyYW0iOnRydWV9)