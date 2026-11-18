start the local git server for the testing purpose only. We will start the git server within the docker container. 
 # docker run -d --name gitea   -p 8085:3000 -p 222:22   -e GITEA__security__INSTALL_LOCK=true   gitea/gitea:latest

 Access the user interface for gitea application.
 # http://localhost:8085

 Register to the application with the testuser and password & create a testrepo for the testing purpose with argocd only. 

 clone the repo.
 # adminuser@IN-CS7MR24:~/temp_work$ git clone http://localhost:8085/testuser/testrepo.git
Cloning into 'testrepo'...
Username for 'http://localhost:8085': testuser
Password for 'http://testuser@localhost:8085':
warning: You appear to have cloned an empty repository.
adminuser@IN-CS7MR24:~/temp_work$

Add the nginx deployment and service.

adminuser@IN-CS7MR24:~/temp_work/testrepo$ ll
total 20
drwxr-xr-x 3 adminuser adminuser 4096 Nov 18 15:20 ./
drwxr-xr-x 3 adminuser adminuser 4096 Nov 18 15:17 ../
drwxr-xr-x 8 adminuser adminuser 4096 Nov 18 15:20 .git/
-rw-r--r-- 1 adminuser adminuser  335 Nov 18 15:19 nginx-deployment.yaml
-rw-r--r-- 1 adminuser adminuser  218 Nov 18 15:20 nginx-svc.yaml

adminuser@IN-CS7MR24:~/argocd/private_repositories$ hostname -I | awk '{print $1}'

 dminuser@IN-CS7MR24:~/argocd/private_repositories$  argocd repo add http://172.31.86.15:8085/testuser/testrepo.git --username testuser --password testPassword@123
Handling connection for 8080
Handling connection for 8080
Repository 'http://172.31.86.15:8085/testuser/testrepo.git' added

2. Repo Add using access token.
adminuser@IN-CS7MR24:~/argocd/private_repositories$ argocd repo add https://github.com/neerajkori/argocd.git --username neeraj --password pat_token_you_created
Handling connection for 8080
Handling connection for 8080
Repository 'https://github.com/neerajkori/argocd.git' added

3. TLS Authentication with private git server. 
    We are using self signed certificate for the testing purpose only. 
    adminuser@IN-CS7MR24:~/temp_work/testrepo$ wget -O gitea https://dl.gitea.com/gitea/28.4.0/gitea-28.4.0-linux-amd64
    chmod +x gitea

    openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout /path/to/key.pem \
  -out /path/to/cert.pem \
  -subj "/CN=172.31.86.15" \
  -addext "subjectAltName=IP:172.31.86.15"

  or

  gitea cert --host 172.31.86.15

-- 

cert creation inside the docker container.
adminuser@IN-CS7MR24:~/temp_work/testrepo/certs$ docker exec -it gitea mkdir -p /data/gitea/certs
adminuser@IN-CS7MR24:~/temp_work/testrepo/certs$ docker exec -it -w /data/gitea/certs gitea gitea cert --host 172.31.86.15
Written cert to cert.pem
Written key to key.pem

4. SSH Private Key Credential
adminuser@IN-CS7MR24:~/argocd/private_repositories$ ssh-keyscan -p 222 172.31.86.15 | argocd cert add-ssh --batch
adminuser@IN-CS7MR24:~/argocd/private_repositories$ argocd repo add ssh://git@172.31.86.15:222/testuser/testsshrepo.git   --ssh-private-key-path /home/adminuser/temp_work/testrepo/gitea-ssh-keys/argocd_gitea   --insecure-ignore-host-key
Handling connection for 8080
Handling connection for 8080
E1118 16:25:34.036924  513606 portforward.go:489] "Unhandled Error" err="error copying from remote stream to local connection: readfrom tcp4 127.0.0.1:8080->127.0.0.1:34178: write tcp4 127.0.0.1:8080->127.0.0.1:34178: write: broken pipe"
Repository 'ssh://git@172.31.86.15:222/testuser/testsshrepo.git' added
