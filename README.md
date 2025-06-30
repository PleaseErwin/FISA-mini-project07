# 도커 컨테이너 관리 및 백업 자동화 

## 2개 이상의 spring app에서 mysql db 컨테이너에 접속



<br>

<b>DB 접속</b>
```
curl localhost:8080/one # 조회
curl -X POST localhost:8080/save # 저장
```

## 자동 실행 및 백업

<b>실행 스크립트 start_docker.sh</b>
```
#!/bin/bash

docker-compose up -d
```
<br>

<b>백업 스크립트 backup_db.sh</b>
```
#!/bin/bash

# 방법 1 : copy
docker cp mysqldb:/var/lib/mysql/fisa ~/06.dockerCompose/backup/backup_$(date +%F)_$(date +%T)

# 방법 2 : mysqldump
docker exec mysqldb sh -c "exec mysqldump -u'root' -p'root' --databases fisa" > ~/06.dockerCompose/backup_dump/backdump_$(date +%F)_$(date +%T).sql
```
<br>

<b>백업 Crontab</b>
```
# Crontab 편집
crontab -e

* * * * * /home/ubuntu/06.dockerCompose/backup_db.sh >> /home/ubuntu/06.dockerCompose/cron.log 2>&1
```

<br><br>

# mysql 컨테이너 이중화를 통한 고가용성 구현

### 1. mysql 컨테이너 개별 구동

```
docker run --name mysqldb --network 06dockercompose_spring-mysql-net --volume db_data:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=root \
    -e MYSQL_DATABASE=fisa \
    -e MYSQL_USER=user01 \
    -e MYSQL_PASSWORD=user01 \
    -d -p 3306:3306 mysql:8.0
```


### 2. docker compose로 mysql 컨테이너 2개를 한번에 구동 

```
#docker-compose.yml

version: "3.8"

services:
  mysqldb1:
    image: mysql:8.0
    container_name: mysqldb1
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    ports:
      - "3307:3306"
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - spring-mysql-net

  mysqldb2:
    image: mysql:8.0
    container_name: mysqldb2
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    ports:
      - "3308:3306"
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - spring-mysql-net

volumes:
  db_data:

networks:
  spring-mysql-net:
    external: true

```

<br>

두 경우 모두 mysql 컨테이너들의 볼륨을 똑같이 설정했더니 TroubleShooting-문제1 발생

<br>

# TroubleShooting

## 문제1

> mysqldb1과 mysqldb2 컨테이너가 동시에 구동되지 않는 문제<br>
> mysqldb1 컨테이너가 구동중이라면 docker compose 실행 오류로 mysqldb2뿐만이 아닌 springapp 또한 구동 실패<br>
> 동시에 mysql 컨테이너들이 켜지더라도 파일 잠금 오류가 발생해 spring app에서 mysql db에 접근 불가

### 원인
docker-compose.yml에서 mysqldb1과 mysqldb2 컨테이너가 동시에 같은 볼륨을 공유하여 충돌 - 파일 잠금 오류(ibdata1 error:11) 발생<br>
```
app:
    container_name: springbootapp
         ...
         ...
         ...
    depends_on:
      db:
        condition: service_healthy
```
Docker Compose는 컨테이너 간의 의존성을 정의함 - depends_on에 지정된 db 서비스가 먼저 실행된 후 app을 실행
<br><br>
이미 mysql 컨테이너를 하나 실행시켰을 경우 docker compose로 또다른 mysql 컨테이너와 springapp을 실행시키려고 시도하면 볼륨 충돌로 인해 mysql 컨테이너 실행에 실패하고, depends_on에 지정된 springapp 또한 실패한다.


<br>

## 문제2

> docker-compose.yml 파일에서 볼륨 이름을 db_data로 설정했는데 docker compose up으로 실행시켰을 때 만들어진 볼륨 이름이 06dockercompose_db_data 가 되는 문제<br>

* docker compose up을 실행시킨 위치의 절대경로는 /home/ubuntu/06.docker_compose

### 원인
Docker Compose는 볼륨, 네트워크 등의 리소스를 관리할 때 프로젝트 디렉토리 이름을 자동으로 prefix 추가하여 고유한 이름을 생성

| 생성 방법                      | 네트워크 이름                        |
|------------------------------------------------|--------------------------------------|
| docker network create spring-mysql-net         | prefix 안 붙음                        |
| docker-compose up (external: false)            | prefix 붙음                           |
| docker-compose up (external: true)             | 기존 네트워크 그대로 사용             |

<br>

<b>방법1 : volumes에서 name 속성을 사용하여 이름 고정</b>
```
#docker-compose.yml 

volumes:
  db_data:
    name: db_data
```

<br>

<b>방법2 : docker-compose.yml과 같은 디렉토리에 .env 파일을 만들고 COMPOSE_PROJECT_NAME 설정 </b>
```
COMPOSE_PROJECT_NAME=
```
비워두면 네임스페이스가 붙지 않고 docker-compose.yml의 설정 그대로 생성됨
