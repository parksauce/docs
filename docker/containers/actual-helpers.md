---
title: Actual Helpers
description: 
published: true
date: 2024-10-08T00:48:47.278Z
tags: containers, docker
editor: markdown
dateCreated: 2024-10-04T20:21:05.265Z
---

# Actual Helpers - Dockerized
## Credits
All credit for the project go to the developers over at [psybers/actual-helpers](https://github.com/psybers/actual-helpers). I simply package the scripts into an easy to deploy and customize container.

## Table of Contents
- [Deployment](#deployment)
    - [Quick Start](#quick-start)
    - [Environment Variables](#environment-variables)
    - [Docker Compose](#docker-compose) 

## Deployment

### Quick Start 
Deploying the container is a fairly simple process, there are three required variables and this can vary depending on the scripts you'd like to use those are `ACTUAL_SERVER_URL`, `ACTUAL_SERVER_PASSWORD`, and `ACTUAL_SYNC_ID`, see the [environment variables](#environment-variables) section for more information. By default this container is configured to run the `sync-banks.js` script. If you'd like to change this set the `SCRIPT` varaible to the name of the script you'd like to run. 

To run the container you can use the following `docker run` command.
```shell
docker run --name=actual-helpers -e ACTUAL_SERVER_URL="https://budget.example.com" -e ACTUAL_SERVER_PASSWORD="1234" -e ACTUAL_SYNC_ID="1234" -e SCRIPT="sync-banks.js parksauce/actual-helpers:latest"
```

The container will run to completion and stop, to keep it lightweight it doesn't have a cron mechanism built into it. If you'd like to run it on a schedule then you'll have to configure a CronJob to run the command on a schedule there's plenty of guides out there on this already so I won't delve into this.

### Environment Variables
|  Variable | Description | Default | Optional |
|:---------:|:-----------:|:-------:|:--------:|
| `SCRIPT` | Sets the script to run in the container | `sync-banks.js` | &#x2713; |
| `ACTUAL_SERVER_URL` | The address of your actual-server instance |  | |
| `ACTUAL_SERVER_PASSWORD` | The password for your actual-server instance |  | |
| `ACTUAL_SYNC_ID` | The sync ID for your actual-server budget. You can get your sync ID from the advanced settings in your actual server. |  | |
| `NODE_TLS_REJECT_UNAUTHORIZED` | Allow self-signed SSL certificates | 0 | &#x2713; |
| `ACTUAL_FILE_PASSWORD` | Sets the password for encrypted files |  | &#x2713; |
| `ACTUAL_CACHE_DIR` | Allows you to change the cache directory | `./cache` | &#x2713; |
| `INTEREST_PAYEE_NAME` | Name of the payee for added interest transactions |  | &#x2713; |
| `INVESTMENT_PAYEE_NAME` | Name of the payee for added interest transactions |  | &#x2713; |
| `INVESTMENT_CATEGORY_GROUP_NAME` | Name of the category group for added investment tracking |  | &#x2713; |
| `INVESTMENT_CATEGORY_NAME` | Name of the category for added investment tracking |  | &#x2713; |
| `SIMPLEFIN_CREDENTIALS` | For logging into SimpleFin. Credentials, not the setup token! |  | &#x2713; |
| `BITCOIN_PRICE_URL` | Host for retrieving Bitcoin price |  | &#x2713; |
| `BITCOIN_PRICE_JSON_PATH` | JSON path for retrieving Bitcoin price |  | &#x2713; |
| `BITCOIN_PAYEE_NAME` | Name of the payee for Bitcoin price changes |  | &#x2713; |

### Docker Compose
Theres an example compose file available in the github, you can pull it to your local machine using the command below.
```shell
wget https://raw.githubusercontent.com/realjoshparker/docker_actual-helpers/refs/heads/master/docker-compose.yaml
```

Once downloaded go ahead and modify the values to suit your needs, for more information on the different environment variables see [here](#environment-variables). I'd advise against putting the `ACTUAL_SERVER_PASSWORD` in the compose file directly, instead place them in a `.env` file and set the permissions to `600`. Your `.env` file can look like below.
```env
ACTUAL_SERVER_URL=https://budget.example.com
ACTUAL_SERVER_PASSWORD=myPassword
ACTUAL_SYNC_ID=< sync ID >
```

To start the container you can run the command below.
```shell
docker compose up
```