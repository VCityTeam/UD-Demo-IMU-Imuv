## Redirection of port outside the ud-demo-imu_imuv network

```mermaid
graph RL
    subgraph localhost
    subgraph ud-demo-imu_imuv network
    imuv-->|port|imuvport(8000/tcp)
    parse[parse-server]-->|port|parseport(1337/tcp)
    mdb[(mongodb)]-->|port|mdbport(27017/tcp)
    wbo[wbo]-->|port|wbop(5001/tcp)
    end
    imuvport-->|redirect|rdrimuvport(0.0.0.0:IMUV_PORT)
    parseport-->|redirect|rdrparseport(0.0.0.0:PARSE_PORT)
    mdbport-->|redirect|rdrmongoport(0.0.0.0:MONGO_PORT)
    wbop-->|redirect|rdrwbop(0.0.0.0:WBO_PORT)
    end
```

There is four services :

- imuv : game
- parse : authentification
- mongo : database
- whitebophir : whiteboard

There are in a docker network and can communicate within that network.
We can redirect the communication ports when we want to serve them in the machine (localhost).

## Sequence diagram of an user authentication

```mermaid
sequenceDiagram
    actor user
    user->>imuv : https://imuv_domain_name
    imuv->>+parse-server: http://parse-server:1337/parse
    parse-server->>+mongodb:mongodb://usr:pwd&#64;mongodb:27017
    mongodb->>-parse-server:mongodb://usr:pwd&#64;mongodb:27017
    parse-server->>-imuv: http://parse-server:1337/parse
    imuv->>user : https://imuv_domain_name
```

Here is the classic path of how information flows to receive data or change data in the user database via the imuv client.

## Running the demo for the first time

The only pre-requisite is to have a host with a running [docker daemon](https://docs.docker.com/) (on the CLI try running `docker ps`) and the [docker compose](https://docs.docker.com/compose/) command installed.

## Setup the environment variables

Copy the [`env-default`](env-default) to `.env` file docker-compose environment
file and customize it to fit your needs.

| VARIABLES                  | DESCRIPTION                                                                                                                    |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| MONGO_INITDB_ROOT_USERNAME | The username of the root user of the mongo database                                                                            |
| MONGO_INITDB_ROOT_PASSWORD | The password of the root user of the mongo database                                                                            |
| MONGO_PORT                 | The port of the mongo database                                                                                                 |
| MONGO_HOST                 | Hostname of the mongo database URI                                                                                             |
| PARSE_PORT                 | The port of the parse-server                                                                                                   |
| IMUV_PORT                  | The port of the imuv game                                                                                                      |
| WBO_PORT                   | The port of the wbo service (whiteboard)                                                                                       |
| PARSE_APP_ID               | The application id of the parse-server. Used by the [IMUV](https://github.com/VCityTeam/UD-Imuv#set-the-environment-variables) |
| PARSE_MASTER_KEY           | The master key of the parse-server. Used by the [IMUV](https://github.com/VCityTeam/UD-Imuv#set-the-environment-variables)     |
| PARSE_MASTER_KEY_IPS       | Restrict masterKey to be used by only these ips.                                                                               |
| JITSI_PUBLIC_URL           | Used by the [IMUV](https://github.com/VCityTeam/UD-Imuv#set-the-environment-variables)                                         |
| WBO_PUBLIC_URL             | Used by the [IMUV](https://github.com/VCityTeam/UD-Imuv#set-the-environment-variables).                                        |
| PARSE_SERVER_URL           | Used by the [IMUV](https://github.com/VCityTeam/UD-Imuv#set-the-environment-variables)                                         |

> Note : Unlike IMUV, we use our own wbo service. So WBO_PUBLIC_URL point to localhost (not dev server).

## Start the demo

Clean volumes (optional):

```bash
rm -fr mongo-data/[A-z]*
rm -fr wbo-boards/[A-z]*
```

> Otherwise, if the `mongo-data` directory happens to have a data base image
> (stored in a previous run and hidden by the `.gitignore`) with a different
> pair of `username/password` as the one provided in the customized `.env` file,
> the mongo database won't complain on still launch smoothly.
> Yet the `username/password` won't be the ones provided in the `.env` file
> ... but the ones encountered in the saved image of the database.
> And this will most often bite you down the road. :-/

Create a folder wich contains the whiteboards json saved with theses commands:

```bash
mkdir wbo-boards # Create a directory that will contain your whiteboards
chown -R 1000:1000 wbo-boards # Make this directory accessible to WBO
```

Then start the demo with the following command (-d is for detached mode):

```bash
docker compose up -d
```

If you want run only certain services, you can use the following command:

```bash
docker compose up -d name_service_1 name_service_2 ...
```

<p id="run_parse_mongo">For example running parse-server and mongodb:</p>

```bash
docker compose up -d parse-server mongodb
```

## Checking the installation

Once docker-compose is running, one can follow the parse-server documentation
(refer to the
["saving an object" section](https://hub.docker.com/r/parseplatform/parse-server/#saving-an-object))
and try creating an entry in the parse server with the following command

```bash
curl -X POST -H "X-Parse-Application-Id: my_parse_app_id" \
     -H "Content-Type: application/json" \
     -d '{"score":1337,"playerName":"John Doe","cheatMode":false}' \
     http://localhost:1337/parse/classes/GameScore
```

that should provide you with an answer of the form

```bash
{"objectId":"8A0YN6GkJ7","createdAt":"2022-04-26T16:33:59.551Z"}
```

You can then assert that writing went all the way by reading with

```bash
curl -X GET \
  -H "X-Parse-Application-Id: my_parse_app_id" \
  http://localhost:1337/parse/classes/GameScore/8A0YN6GkJ7
```

## Trouble shooting

### Inspecting the mongodb content

```bash
docker run -it --rm  mongo:4.4.7 mongo --host <FQDN>:1338 -u mygogodancer -p mygogodummypass
> use test
> db.GameScore.find()
```

### Running individual docker commands

In case some manual troubleshooting is required, containers can be
run from cli with the following commands

```bash
# Make sure we start with a clean slate database.
# WARNING: if you forget this clean-up and your database
#   was set up with different username/password, mongodb won't
#   complain on launch. Yet the password won't be the ones
#   provided as environment variables...but the ones encountered
# in the saved db.
# And this will burn you quite some time down the road. :-/
rm -fr mongo-data/[A-z]*
docker network create imuv-net
docker run --rm -d --network imuv-net --name my-mongo \
   -e MONGO_INITDB_ROOT_USERNAME=mygogo -e MONGO_INITDB_ROOT_PASSWORD=mydummypass \
   -v $(pwd)/mongo-data:/data/db mongo
# Optionnal: look about possible complaints of database
docker logs --follow --until=10s my-mongo
# Make sure one can connect to the database (exit with quit)
docker run -it --rm --network imuv-net mongo mongo --host my-mongo -u mygogo -p mydummypass
# Run the parse server
docker run --network imuv-net --rm --name my-parse-server \
  -p 1337:1337 parseplatform/parse-server \
  --appId APPLICATION_ID \
  --masterKey MASTER_KEY \
  --databaseURI mongodb://mygogo:mydummypass@my-mongo:27017
```
