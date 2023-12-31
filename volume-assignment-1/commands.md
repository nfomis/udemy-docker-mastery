```bash
# create volume
docker volume create --name postgres-data
# che volume created
docker volume ls | grep postgres-data

# create container postgres 9.6.1
docker run -d --name postgres-961 -v postgres-data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=mysecretpassword -p 5432:5432 postgres:9.6.1

# upgrade container postgres 9.6.2
docker stop postgres-961
docker rm postgres-961
docker run -d --name postgres-962 -v postgres-data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=mysecretpassword -p 5432:5432 postgres:9.6.2

# clean up
docker stop postgres-962
docker rm -f $(docker container ls -a | grep postgres-9 | awk '{print $1}')
docker volume rm postgres-data
docker image rm -f $(docker image ls | grep '9\.6' | awk '{print $3}')
```
