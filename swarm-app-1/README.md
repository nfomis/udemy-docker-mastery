# Assignment: Create A Multi-Service Multi-Node Web App

## Goal: create networks, volumes, and services for a web-based "cats vs. dogs" voting app

Here is a basic diagram of how the 5 services will work:

![diagram](./architecture.png)

- All images are on Docker Hub, so you should use editor to craft your commands locally,
  then paste them into swarm shell (at least that's how I'd do it)
- a `backend` and `frontend` overlay network are needed.
  Nothing different about them other than that backend will help protect database from the voting web app.
  (similar to how a VLAN setup might be in traditional architecture)
- The database server should use a named volume for preserving data.
  Use the new `--mount` format to do this: `--mount type=volume,source=db-data,target=/var/lib/postgresql/data`

### Services (names below should be service names)

- vote

  - bretfisher/examplevotingapp_vote
  - web frontend for users to vote dog/cat
  - ideally published on TCP 80. Container listens on 80
  - on frontend network
  - 2+ replicas of this container

- redis

  - redis:3.2
  - key-value storage for incoming votes
  - no public ports
  - on frontend network
  - 1 replica NOTE VIDEO SAYS TWO BUT ONLY ONE NEEDED

- worker

  - bretfisher/examplevotingapp_worker
  - backend processor of redis and storing results in postgres
  - no public ports
  - on frontend and backend networks
  - 1 replica

- db

  - postgres:9.4
  - one named volume needed, pointing to /var/lib/postgresql/data
  - on backend network
  - 1 replica
  - remember set env for password-less connections -e POSTGRES_HOST_AUTH_METHOD=trust

- result
  - bretfisher/examplevotingapp_result
  - web app that shows results
  - runs on high port since just for admins (lets imagine)
  - so run on a high port of your choosing (I choose 5001), container listens on 80
  - on backend network
  - 1 replica

### Commands

- create the swarm cluster:
  - `docker swarm init --advertise-addr <ip>` on the manager node
  - `docker swarm join --token <token> <ip>:2377` on the worker nodes
  - check the swarm status:
    - `docker node ls`
    ```bash
    ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
    sf4qi2zdv26h5n8pjapkxn90q *   node1      Ready     Active         Leader           24.0.7
    t93whjwczei22cn7o7x2i8tnb     node2      Ready     Active                          24.0.7
    4oh66ur3mxv767fyry159kmt3     node3      Ready     Active                          24.0.7
    ```
- create the overlay networks:
  - `docker network create -d overlay frontend`
  - `docker network create -d overlay backend`
  - check the networks:
    - `docker network ls`
    ```bash
    NETWORK ID     NAME              DRIVER    SCOPE
    dgw49cn43j3z   backend           overlay   swarm
    b3b65d42c75c   bridge            bridge    local
    a037e4d2fba3   docker_gwbridge   bridge    local
    tljqnl1vyns4   frontend          overlay   swarm
    8c9656be5427   host              host      local
    k2pmikyke7on   ingress           overlay   swarm
    cc6b03a07275   none              null      local
    ```
- create the services:
  - vote: `docker service create --name vote -p 80:80 --network frontend --replicas 2 bretfisher/examplevotingapp_vote`
    - check the vote service is up:
      ```bash
      >> docker service ls
      ID             NAME      MODE         REPLICAS   IMAGE                                     PORTS
      kwkcrxfums1t   vote      replicated   2/2        bretfisher/examplevotingapp_vote:latest   *:80->80/tcp
      >> docker service ps vote
      ID             NAME      IMAGE                                     NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
      2q2tkmg8wwsu   vote.1    bretfisher/examplevotingapp_vote:latest   node2     Running         Running 2 minutes ago
      uvvpxpktdjxh   vote.2    bretfisher/examplevotingapp_vote:latest   node1     Running         Running 2 minutes ago
      ```
  - redis: `docker service create --name redis --network frontend redis:3.2`
  - worker: `docker service create --name worker --network frontend --network backend bretfisher/examplevotingapp_worker`
  - db: `docker service create --name db --network backend --mount type=volume,source=db-data,target=/var/lib/postgresql/data -e POSTGRES_HOST_AUTH_METHOD=trust postgres:9.4`
    - check the volume is created:
      ```bash
      >> docker volume ls
      DRIVER    VOLUME NAME
      local     db-data
      ```
  - result: `docker service create --name result --network backend -p 5001:80 bretfisher/examplevotingapp_result`
- check all services are up and running:
  ```bash
  >> docker service ls
  ID             NAME      MODE         REPLICAS   IMAGE                                       PORTS
  2i4j917n3tdm   db        replicated   1/1        postgres:9.4
  mwza03pdsey3   redis     replicated   1/1        redis:3.2
  poc9iv0vy59l   result    replicated   1/1        bretfisher/examplevotingapp_result:latest   *:5001->80/tcp
  kwkcrxfums1t   vote      replicated   2/2        bretfisher/examplevotingapp_vote:latest     *:80->80/tcp
  c8xm5luy7i5y   worker    replicated   1/1        bretfisher/examplevotingapp_worker:latest
  ```
