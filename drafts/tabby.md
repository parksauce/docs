## Overview
Tabby is a debt tracker, it allows you to manage debts and remind others about it. This write-up will cover basic configuration and usage of `parksauce/tabby`. We will use two different methods to deploy the container Docker CLI and Docker Compose.

## Usage
The container is super simple to use, its only requirement is a database; you can use tabby supports both postgres and mysql.

### Docker CLI
First create a network called `tabby-backend`

```bash
docker network create tabby
```
Simply run the following command to set up the Tabby container.

```bash
docker run -d \
--name=tabby \
--network=tabby \
-p 8090:80 \
--restart unless-stopped \
thealpaka/tabby
```

Then run this command to setup the database, once done you can visit `http://<HOST_IP>:8090` to finish setup for Tabby.

```bash
docker run -d \
  --name=mariadb \
  --network=tabby \
  -e PUID=1000 # Run 'id' in your terminal to get this value \
  -e PGID=1000 # Run 'id' in your terminal to get this value \
  -e MYSQL_ROOT_PASSWORD=ROOT_ACCESS_PASSWORD \
  -e TZ=America/New_York \
  -e MYSQL_DATABASE=tabby \
  -e MYSQL_USER=tabby \
  -e MYSQL_PASSWORD=tabby \
  -p 3306:3306 \
  -v path_to_data:/config \
  --restart unless-stopped \
  linuxserver/mariadb
```

### Docker Compose
Create a file named `docker-compose.yml`, then run `docker-compose pull && docker-compose up -d`.

```yaml
version: '3'
services:
  tabby:
    image: thealpaka/tabby
    container_name: tabby
    ports:
      - 8090:80
    restart: unless-stopped
  db:
    image: linuxserver/mariadb
    container_name: tabby-db
    environment:
      - PUID=1000 # Run 'id' in your terminal to get this value
      - PGID=1000 # Run 'id' in your terminal to get this value
      - MYSQL_ROOT_PASSWORD=ROOT_ACCESS_PASSWORD
      - TZ=America/New_York
      - MYSQL_DATABASE=tabby
      - MYSQL_USER=tabby
      - MYSQL_PASSWORD=tabby
    volumes:
      - ./db:/config
    restart: unless-stopped
```

## License
This project is licensed under the AGPL license - see the [LICENSE](https://github.com/parksauce/docker_tabby/blob/main/LICENSE) file for details.

## Credits
All credits to Tabby goes to the devs at [https://github.com/bertvandepoel/tabby](https://github.com/bertvandepoel/tabby). 
Credits for the docker base image go to the devs over at [Linuxserver.io](https://linuxserver.io).
