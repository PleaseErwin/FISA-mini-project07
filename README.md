# 도커 컨테이너 관리 및 백업 자동화 

## 2개 이상의 spring app에서 mysql db 컨테이너에 접속


<br><br>

# mysql 컨테이너 이중화를 통한 고가용성 구현

### 2. mysql 컨테이너 개별 구동

```
docker run --name mysqldb --network 06dockercompose_spring-mysql-net --volume 06dockercompose_db_data:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=root \
    -e MYSQL_DATABASE=fisa \
    -e MYSQL_USER=user01 \
    -e MYSQL_PASSWORD=user01 \
    -d -p 3307:3306 mysql:8.0
```

<br>

### 2. docker compose로 mysql 컨테이너 2개를 한번에 구동 

```
#docker-compose.yml



```

<br>

두 경우 모두 mysql 컨테이너의 볼륨을 똑같이 설정했더니 TroubleShooting-문제1 발생

<br>

# TroubleShooting

## 문제1

> mysqldb1과 mysqldb2 컨테이너가 동시에 구동되지 않는 문제<br>
> mysqldb 컨테이너가 하나 구동중이라면 docker compose 실행 오류로 mysqldb2뿐만이 아닌 springapp 또한 구동 실패<br>
> 동시에 mysql 컨테이너들이 켜지더라도 파일 잠금 오류가 발생해 spring app에서 mysql db에 접근 불가

### 원인
docker-compose.yml에서 mysqldb1과 mysqldb2 컨테이너가 동시에 같은 볼륨을 공유하여 충돌 - 파일 잠금 오류(ibdata1 error:11) 발생

<br>

## 문제2

> docker-compose.yml 파일에서 볼륨 이름을 db_data로 설정했는데 docker compose up으로 실행시켰을 때 만들어진 볼륨 이름이 06dockercompose_db_data 가 되는 문제<br>

* docker compose up을 실행시킨 위치의 절대경로는 /home/ubuntu/06.docker_compose

### 원인
Docker Compose는 볼륨, 네트워크 등의 리소스를 관리할 때 프로젝트 디렉토리 이름을 자동으로 prefix 추가하여 고유한 이름을 생성

<br><br>


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


