### 1. Submission Deliverable
This submission contains following files, which will be copied into the created docker image:
* json-rest-api-1.0.8.zip - the json-rest-api plugin for grails
* ant-libs.tgz - the libraries for apache ant
* jboss-service.xml - jboss configuration file, it's changed to enable OnlineReview and TC Direct to run together according to [TC Direct Setup Guide](http://apps.topcoder.com/wiki/display/docs/TC+Direct+Setup+Guide)
* server.xml - jboss configuration file, it's changed to enable SSL according to [TC Direct Setup Guide](http://apps.topcoder.com/wiki/display/docs/TC+Direct+Setup+Guide)
* myserver.keystore - the keystore for SSL
* run.conf - jboss configuration file, it's changed to avoid OutOfMemoryError according to [TC Direct Setup Guide](http://apps.topcoder.com/wiki/display/docs/TC+Direct+Setup+Guide)
* token.properties - the token properties file for tc direct
* topcoder_global.properties - the global properties file for tc direct

The Dockerfile is used to build the docker image. 

### 2. Database Setup
Launch the database server using the topcoder's informix image as below:
```sh
docker run -itd --name tc-informix -p 2021:2021 appiriodevops/informix:1.0
```
At this point you should be able to access informix at localhost:2021. You can use tools like ServerStudio to connect to it and manage it.

Note that the image above may not contain latest db schema, follow the steps below to update it. 

You need to log into the tc-informix container first:
```sh
docker exec -it tc-informix bash
```

Then execute the following commands:
```sh
# go into db trunk directory
cd /home/informix/trunk

# update the db shema files, you may need to enter your 
# svn username/password to the db schema repository
svn update

# drop the databases
ant drop_db

# create the databases
ant setup_db
```

Now the db schema should be up to date.

### 3. Build TC Direct Docker Image

In the direct-app-local directory, execute the following commands to build the docker image first:
```sh
docker build -t direct .
```

Then launch container from the image built above:
```sh
docker run -itd --name tc-direct --link tc-informix -p 8180:8180 -p 443:443 direct bash
```

The container has a link to the tc-informix container launched earlier, so we can access the database server using tc-informix as the hostname. 
And if the jboss inside the container is launched, we will be able to access it on the 8180 (http) & 443 (https) ports.

### 4. TC Direct Setup
##### 4.1 Deploy TC Direct
Now log into the tc-direct container as below:
```sh
docker exec -it tc-direct bash
```

Then execute the following commands to download and deploy the direct app:
```sh
# download source code
git clone https://github.com/appirio-tech/direct-app /root/direct

# create directories configured in properties file
mkdir /root/temp_files
mkdir -p /root/studiofiles/submissions

# copy properties files
cp /root/token.properties /root/direct
cp /root/topcoder_global.properties /root/direct

# build and deploy
cd /root/direct
ant first_deploy

# start the jboss
cd /root/jboss-4.2.3.GA/bin
./run.sh -b 0.0.0.0
```

Then configure the following entry in your hosts file of your computer (NOT the docker container):
```
192.168.99.100 tc.cloud.topcoder.com
```

Where 192.168.99.100 is the ip address of my docker machine (launched via docker toolbox for windows or mac).
If you are using linux, you can simply use 127.0.0.1 here.

Wait until the jboss server is ready, and then open the following link in your browser:
http://tc.cloud.topcoder.com:8180/direct

It will redirect you to https://tc.cloud.topcoder.com/direct, login with heffan/password. 
Note that before redirecting to the https, the browser will warn you about the security certificate (as it's generated by me), just ignore and proceed. 

And as the jboss-cache is not configured, so you will see a bunch of exceptions in jboss log.

##### 4.2 Extra Notes
Note that for the first time, the jboss does not contain the needed dependencies and configuration for TC Direct application, so we should first run 'ant first_deploy'.
For later deployment you can simply run 'ant deploy' to build and deploy the latest EAR tarball.

The direct app's build.xml defines the following main targets.

* *first_deploy* It is used if the jboss is not setup yet for deploying TC Direct. It will copy all dependencies to your local JBoss folder, build and deploy the EAR file by using the already built libs. deploy It will build and deploy EAR file using the already built libs.
* *deploy-internal* this target deploys the configurations files to jboss.
* *deploy-static-files* this target deploys the static files to jboss.
* *deploy-jboss-files* this target deploys the dependent ears to jboss.
* *dist-backend* this target will rebuilt all the components from the source code.
* *ear* this target will build the EAR tarball file. 
* *deploy* this target will deploy the EAR tarball to jboss
