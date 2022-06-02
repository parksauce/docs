# Compose file to more quickly deploy Tabby
```bash
version: '3'
services:

  tabby:
    image: parksauce/tabby
    container_name: tabby
    ports:
      - 8010:80
    environment:
      - TZ=America/New_York
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

  #db:
  #  image: postgres
  #  container_name: tabby-db
  #  networks:
  #    - backend
  #  environment:
  #    - POSTGRES_DB=tabby
  #    - POSTGRES_USER=tabby
  #    - POSTGRES_PASSWORD=tabby
  #  volumes:
  #    - ./db:/var/lib/postgresql/data
  #  restart: unless-stopped
```