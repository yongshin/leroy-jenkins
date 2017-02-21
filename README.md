# leroy-jenkins

This repo contains is a set of instructions to get you started on running Jenkins in a Container and building and deploying applications with Docker DataCenter for Engine 1.13.

## Provision node to run Jenkins on

#### Install CS Engine on Node
```
curl -fsSL https://packages.docker.com/1.13/install.sh | repo=testing sh
```

#### Join Node to Docker Swarm
```
docker swarm join --token ${SWARM_TOKEN} ${SWARM_MANAGER}:2377
```

#### Create Jenkins directory on Node
```
mkdir jenkins
```

#### Create Node label on Docker Engine
```
docker node update --label-add jenkins
```

#### Install DTR CA on Node as well as all Nodes inside of UCP Swarm (if using self-signed certs)
```
export DTR_IPADDR=$(cat /vagrant/dtr-vancouver-node1-ipaddr)
openssl s_client -connect ${DTR_IPADDR}:443 -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM | sudo tee /usr/local/share/ca-certificates/${DTR_IPADDR}.crt
sudo update-ca-certificates
sudo service docker restart
```

## Build application using Jenkins

### Setup Jenkins

#### Build from Dockerfile on Github (Optional, otherwise just pull from DockerHub)
```
docker build -t yongshin/leroy-jenkins .
```

#### Download UCP Client bundle from ucp-bundle-admin and unzip in `ucp-bundle-admin` folder
```
cp -r /vagrant/ucp-bundle-admin/ .
cd ucp-bundle-admin
source env.sh
```

#### Copy scripts folder that includes `trust-dtr.sh` script, provided by [vagrant-vancouver](https://github.com/yongshin/vagrant-vancouver)
```
cp -r /vagrant/scripts/ /home/ubuntu/scripts
```

#### Start Jenkins by mapping workspace, expose Docker socket and Docker compose to container:

```
docker service create --name leroy-jenkins --network ucp-hrm --publish 8080:8080 \
  --mount type=bind,source=/home/ubuntu/jenkins,destination=/var/jenkins_home \
  --mount type=bind,source=/var/run/docker.sock,destination=/var/run/docker.sock \
  --mount type=bind,source=/home/ubuntu/ucp-bundle-admin,destination=/home/jenkins/ucp-bundle-admin \
  --mount type=bind,source=/home/ubuntu/scripts,destination=/home/jenkins/scripts \
  --mount type=bind,source=/home/ubuntu/notary,destination=/usr/local/bin/notary \
  --label com.docker.ucp.mesh.http.8080=external_route=http://jenkins.local,internal_port=8080 \
  --constraint 'node.labels.jenkins == true' yongshin/leroy-jenkins
```

#### Have Jenkins trust the DTR CA (if using self-signed certs)
Run this inside of Jenkins container, mounted from a volume as shown above, the contents of the file are here: [trust-dtr.sh](https://github.com/yongshin/vagrant-vancouver/blob/master/scripts/trust-dtr.sh)
```
./scripts/trust-dtr.sh
```

#### Copy password from jenkins folder on Node
Run this inside of node
```
sudo more jenkins/secrets/initialAdminPassword
```

### Setup Docker Build and Push to DTR Jenkins Job

#### Create repo in DTR to push images. Otherwise authentication to DTR will fail on build.
![Repo](images/repo.png?raw=true)

#### Initialize notary on repository
Run `notary init` on the newly created repo `docker-node-app`
```
ubuntu@worker-node2:~$ docker ps
CONTAINER ID        IMAGE                                                                                            COMMAND                  CREATED             STATUS              PORTS                     NAMES
09a07f72010d        yongshin/leroy-jenkins@sha256:6bc8aeff905bb504de40ac9da15b5842108c01ab02e8c4b56064902af376b473   "/bin/tini -- /usr..."   22 hours ago        Up 22 hours         8080/tcp, 50000/tcp       leroy-jenkins.1.vuiocbnlz8cvi9925vsy54i5q
20b8b64d6a3f        docker/ucp-agent@sha256:a428de44a9059f31a59237a5881c2d2cffa93757d99026156e4ea544577ab7f3         "/bin/ucp-agent agent"   23 hours ago        Up 23 hours         2376/tcp                  ucp-agent.mkf4p9818iydomuokv9o8ztyv.zmh24nsty2epkne4dukk3m0di
3bcfa136e99b        docker/ucp-agent:2.1.0                                                                           "/bin/ucp-agent proxy"   24 hours ago        Up 23 hours         0.0.0.0:12376->2376/tcp   ucp-proxy

ubuntu@worker-node2:~$ docker exec -it 09a07f72010d bash
root@09a07f72010d:/# notary -s https://172.28.128.4 init 172.28.128.4/engineering/docker-node-app
Root key found, using: c47333b8b15fe43a6abc59dcb29f4e60dee1807919dfc05f6e57dbfc57553d88
Enter passphrase for root key with ID c47333b:
Enter passphrase for new targets key with ID 8e7009a (172.28.128.4/engineering/docker-node-app):
Repeat passphrase for new targets key with ID 8e7009a (172.28.128.4/engineering/docker-node-app):
Enter passphrase for new snapshot key with ID 16827df (172.28.128.4/engineering/docker-node-app):
Repeat passphrase for new snapshot key with ID 16827df (172.28.128.4/engineering/docker-node-app):
Enter username: admin
Enter password:

root@09a07f72010d:/#
```

#### Add delegation for targets/releases
```
root@6ddfb62a5b8d:/home/jenkins/ucp-bundle-admin# notary -s https://172.28.128.4 delegation add -p 172.28.128.4/engineering/docker-node-app targets/releases --all-paths cert.pem --debug

DEBU[0000] Configuration file not found, using defaults
DEBU[0000] Using the following trust directory: /root/.notary
DEBU[0000] No yubikey found, using alternative key storage: no library found
DEBU[0000] Making dir path: /root/.notary/tuf/172.28.128.4/engineering/docker-node-app/changelist
DEBU[0000] Adding delegation "targets/releases" with threshold 1, and 1 keys\n
DEBU[0000] Making dir path: /root/.notary/tuf/172.28.128.4/engineering/docker-node-app/changelist
DEBU[0000] Adding [] paths to delegation targets/releases\n
Addition of delegation role targets/releases with keys [8f3d39a5265a1245a958dd3123a008588d1ea93684166ec0019cb312346370cf], with paths ["" <all paths>], to repository "172.28.128.4/engineering/docker-node-app" staged for next publish.
DEBU[0000] No yubikey found, using alternative key storage: no library found
Auto-publishing changes to 172.28.128.4/engineering/docker-node-app
DEBU[0000] Making dir path: /root/.notary/tuf/172.28.128.4/engineering/docker-node-app/changelist
DEBU[0000] entered ValidateRoot with dns: 172.28.128.4/engineering/docker-node-app
DEBU[0000] found the following root keys: [dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1]
DEBU[0000] found 1 valid leaf certificates for 172.28.128.4/engineering/docker-node-app: dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1
DEBU[0000] found 1 leaf certs, of which 1 are valid leaf certs for 172.28.128.4/engineering/docker-node-app
DEBU[0000] checking root against trust_pinning config%!(EXTRA string=172.28.128.4/engineering/docker-node-app)
DEBU[0000] checking trust-pinning for cert: dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1
DEBU[0000]  role has key IDs: dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1
DEBU[0000] verifying signature for key ID: dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1
DEBU[0000] root validation succeeded for 172.28.128.4/engineering/docker-node-app
DEBU[0000] entered ValidateRoot with dns: 172.28.128.4/engineering/docker-node-app
DEBU[0000] found the following root keys: [dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1]
DEBU[0000] found 1 valid leaf certificates for 172.28.128.4/engineering/docker-node-app: dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1
DEBU[0000] found 1 leaf certs, of which 1 are valid leaf certs for 172.28.128.4/engineering/docker-node-app
DEBU[0000] checking root against trust_pinning config%!(EXTRA string=172.28.128.4/engineering/docker-node-app)
DEBU[0000] checking trust-pinning for cert: dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1
DEBU[0000]  role has key IDs: dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1
DEBU[0000] verifying signature for key ID: dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1
DEBU[0000] root validation succeeded for 172.28.128.4/engineering/docker-node-app
Enter username: admin
Enter password:
DEBU[0009] received HTTP status 404 when requesting root.
DEBU[0009] Loading trusted collection.                  
DEBU[0009] entered ValidateRoot with dns: 172.28.128.4/engineering/docker-node-app
DEBU[0009] found the following root keys: [dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1]
DEBU[0009] found 1 valid leaf certificates for 172.28.128.4/engineering/docker-node-app: dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1
DEBU[0009] found 1 leaf certs, of which 1 are valid leaf certs for 172.28.128.4/engineering/docker-node-app
DEBU[0009] checking root against trust_pinning config%!(EXTRA string=172.28.128.4/engineering/docker-node-app)
DEBU[0009] checking trust-pinning for cert: dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1
DEBU[0009]  role has key IDs: dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1
DEBU[0009] verifying signature for key ID: dfb9d4de98113f6e4088b9611550f0b01c84260baf51deb1dfb3b75c48bc9fa1
DEBU[0009] root validation succeeded for 172.28.128.4/engineering/docker-node-app
DEBU[0009] targets role has key IDs: 6f2823e832eeb072559b88bac12cf877ef7c75e7d002648d42d846daa1b60ed1
DEBU[0009] verifying signature for key ID: 6f2823e832eeb072559b88bac12cf877ef7c75e7d002648d42d846daa1b60ed1
DEBU[0009] snapshot role has key IDs: 80b729fecdccdd31172d5c0b389b510c42367a291fcc4b0ff306833a2e234a3d
DEBU[0009] verifying signature for key ID: 80b729fecdccdd31172d5c0b389b510c42367a291fcc4b0ff306833a2e234a3d
DEBU[0009] No yubikey found, using alternative key storage: no library found
Enter passphrase for targets key with ID 6f2823e:
DEBU[0013] role targets/releases with no Paths will never be able to publish content until one or more are added
DEBU[0013] No yubikey found, using alternative key storage: no library found
DEBU[0013] No yubikey found, using alternative key storage: no library found
DEBU[0013] No yubikey found, using alternative key storage: no library found
DEBU[0013] applied 2 change(s)                          
DEBU[0013] sign targets called for role targets         
DEBU[0013] sign called with 1/1 required keys           
DEBU[0013] No yubikey found, using alternative key storage: no library found
DEBU[0013] sign called with 0/0 required keys           
DEBU[0013] signing snapshot...                          
DEBU[0013] sign called with 1/1 required keys           
DEBU[0013] No yubikey found, using alternative key storage: no library found
Enter passphrase for snapshot key with ID 80b729f:
DEBU[0016] sign called with 0/0 required keys           
Successfully published changes for repository 172.28.128.4/engineering/docker-node-app
```

#### Create 'docker build and push' Free-Style Jenkins Job
![Jenkins Job](images/jenkins-create-job.png?raw=true)

#### Source Code Management -> Git - set repository to the repository to check out source
```
https://github.com/yongshin/docker-node-app.git
```

#### Set Build Triggers -> Poll SCM
```
* * * * *
```

#### Add Build Step -> Execute Shell
```
#!/bin/bash
export DTR_IPADDR=172.28.128.11
export DOCKER_CONTENT_TRUST=1 DOCKER_CONTENT_TRUST_ROOT_PASSPHRASE=docker123 DOCKER_CONTENT_TRUST_REPOSITORY_PASSPHRASE=docker123
docker build -t ${DTR_IPADDR}/engineering/docker-node-app:1.${BUILD_NUMBER} .
docker tag ${DTR_IPADDR}/engineering/docker-node-app:1.${BUILD_NUMBER} ${DTR_IPADDR}/engineering/docker-node-app:latest
docker login -u admin -p dockeradmin ${DTR_IPADDR}
docker push ${DTR_IPADDR}/engineering/docker-node-app:1.${BUILD_NUMBER}
docker push ${DTR_IPADDR}/engineering/docker-node-app:latest
```

### Setup Docker Deploy Jenkins Job

#### Create Docker Deploy Application Free-Style Jenkins Job
![Docker Create Job](images/jenkins-create-job-deploy.png?raw=true)

#### Source Code Management -> Git - set repository to the repository to check out source
```
https://github.com/yongshin/docker-node-app.git
```

#### Add Jenkins build trigger to run deploy after image build job is complete
![Jenkins Build Trigger](images/jenkins-build-trigger.png?raw=true)

#### Add Build Step -> Execute Shell
```
#!/bin/bash
export DTR_IPADDR=172.28.128.11
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH="/home/jenkins/ucp-bundle-admin"
export DOCKER_HOST=tcp://172.28.128.5:443
docker login -u admin -p dockeradmin ${DTR_IPADDR}
docker pull ${DTR_IPADDR}/engineering/docker-node-app:latest
docker pull clusterhq/mongodb
docker stack rm nodeapp
docker stack deploy -c docker-compose.yml nodeapp
```
