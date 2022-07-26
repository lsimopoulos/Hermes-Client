# Hermes-Client
The hermes chat client implemented in [VUE](https://vuejs.org/).

## Requirements ##
[Docker](https://docs.docker.com/get-docker/) 

## How to run ##

### Docker ###

Client
```
 docker run -it --name hermes-client -p 55555:443 leonidassim/hermes-client 
```

Server
```
 docker run -it --name hermes -p 5001:5001 -p 7001:7001 leonidassim/hermes-server 
```

#### Note ####
At this phase the hosting of both client and server need to run on the same pc/server. Since both client and server are using self signed certificates, the user need to make the following actions in order to work in rhe webbrowser:
*  In the webrowser navigate to [client](https://localhost:5555) and accept the certificate
* As for the server navigate to [localhost:7001](https://localhost:7001) and accept the certificate (in order the register and login functionality to work)


After both client and server are running , navigate  to [client](https://localhost:5555)
