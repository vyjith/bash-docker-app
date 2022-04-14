# bash-docker-app
```sh


# step 1: Check if the user is root or not.. If the shell is root user then run the coomamand if not exist from the console then print run this command as root user checking

####
dockercompose_root(){
if [ "$EUID"  == 0 ]
then
        dockercompose_os
else
        echo ""
        echo "Please run this command as the root user. If you want to switch the root user, if yes, please enter su - on your terminal"
        echo ""
        exit 1
fi
}
####

#step2: Need to check the os release.. If it is amzon liinux then run the rest of the command...

####
dockercompose_os(){
os=`hostnamectl | grep "Operating System" | awk '{print$3,$4,$5}'`
servername="Amazon Linux 2"

if [ "$os" = "$servername" ];

then
  echo ""
  docker_script
else
  echo "test is failed and exiting from here now"
  exit 1
fi
}

####

## step 3 :: checking the docker is installed or not on the server.....

dockercompose_requiredpackage(){

echo ""
echo "Checking the required docker packge is installed or not on your server. Please hold on..............................................................................."
echo ""
dockerstatus=`systemctl show -p ActiveState docker.service | sed 's/ActiveState=//g'`
if [ "$dockerstatus" = active ];
        then
        echo "The docker is already installed on teh server"
else
    echo "The docker version is not installed on the server."
        sudo yum install docker -y
        sudo systemctl start docker.service
        sudo systemctl enable docker.service

fi
}

# Step 4 : Checking the docker-compose is installed or not

dockercomposeversion(){

docker-compose --version || error_code=$?;

if [[ "$error_code" -eq 0 ]];

        then
        echo "The docker compose version is already installed on the server"
        else
        echo "The docker-compose not installed on the server. Installing the docker-compose version. Please hold a moment........."
        sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
                sudo chmod +x /usr/local/bin/docker-compose
fi
}

# Steps5 : Everything fine now, so that we are starting....

docker_script(){

dockercompose_requiredpackage
dockercomposeversion

mkdir /root/ip-api
mkdir /root/ip-api/nginx
echo ""
echo -n "Please let me know the domain name you would like to use for this : "
read domain

echo ""
echo -n "Please let me know the API key, you will get the API key by login here : https://app.ipgeolocation.io/auth/login : "
read API_KEY

echo -n "Please let me know the docker network, you would like to use : "
read mynet

docker network create $mynet

echo -n "Please let me know your server IP : "

read serverip

## Creating Nginx configration--------------------------------------------------
cat > /root/ip-api/nginx/proxy.conf << EOF
upstream loadbalancer {
server $serverip:8081;
server $serverip:8082;
server $serverip:8083;
}
server {
    listen 80;
    listen [::]:80;
    server_name $domain;
    server_tokens off;


    location / {
        return 301 https://$domain$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name  $domain;
    ssl_certificate /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;
    location / {
        proxy_pass http://loadbalancer;
    }
}
EOF
## Docker compose file creating here--------------------------------------------------

cat > /root/ip-api/docker-compose.yml << EOF
version: '3'
services:
  app:
    image: redis:latest
    container_name: ipgeolocation-caching-service
    networks:
      - $mynet
  api_servcie:
    container_name: ipgeolocation-api-service
    env_file: .env
    environment:
      - REDIS_PORT=6379
      - REDIS_HOST=ipgeolocation-caching-service
      - APP_PORT=8080
      - API_KEY=$API_KEY
    networks:
      - $mynet
    ports:
      - "8080:8080"
    image: fujikomalan/ipgeolocation-api-service:v1
  frontend1:
    container_name: ipgeolocation-frontend-service-1
    image: fujikomalan/ipgeolocation-frontend:v1
    env_file: .env
    environment:
      - API_SERVER=ipgeolocation-api-service
      - API_SERVER_PORT=8080
      - API_PATH=/api/v1/
      - APP_PORT=8080
    networks:
      - $mynet
    ports:
      - "8081:8080"
  frontend2:
    container_name: ipgeolocation-frontend-service-2
    image: fujikomalan/ipgeolocation-frontend:v1
    env_file: .env
    environment:
      - API_SERVER=ipgeolocation-api-service
      - API_SERVER_PORT=8080
      - API_PATH=/api/v1/
      - APP_PORT=8080
    networks:
      - $mynet
    ports:
      - "8082:8080"
  frontend3:
    container_name: ipgeolocation-frontend-service-3
    image: fujikomalan/ipgeolocation-frontend:v1
    env_file: .env
    environment:
      - API_SERVER=ipgeolocation-api-service
      - API_SERVER_PORT=8080
      - API_PATH=/api/v1/
      - APP_PORT=8080
    networks:
      - $mynet
    ports:
      - "8083:8080"
  webserver:
    container_name: nginx
    image: nginx:1.15.12-alpine
    env_file: .env
    volumes:
      - ./nginx/:/etc/nginx/conf.d/
      - ./data/certs:/etc/nginx/certs
    networks:
      - $mynet
    ports:
      - "80:80"
      - "443:443"
    restart: always

networks:
  $mynet:
    driver: bridge
EOF

cat > /root/ip-api/.env << EOF

# If you would like to any details of env file here
EOF

cd /root/ip-api/
docker-compose up -d

}

docker_main(){
  dockercompose_root
}
docker_main

```
