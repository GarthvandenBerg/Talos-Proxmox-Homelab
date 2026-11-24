I was going to look at using Gatus but I found I don't need many services to be managed, might move once there becomes too many things to monitor and manual updating of files gets too much 

```
mkdir -p monitor
mkdir -p monitor/data

sudo chown -R 1000:1000 monitor/data

cd monitor

# Copy the docker compose file
vi docker-compose.yaml

# Start container
docker compose up -d
```