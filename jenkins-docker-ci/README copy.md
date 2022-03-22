# intro

The aim of this blog is to exlain a method for running a Jenkins pipeline which uses Selenium Grid, doing both in docker containers. 

We started with a docker compose group for running jenkins to use selenium grid for running automation tests. We wanted to modify this in order to use .NET. Out of the box, the docker group is unable to run .NET unit tests using MSTest.

We modified the docker compose YAML and the dockerfile to add the required .NET to the jenkins container. The intention is that these instructions should be a working starting point for anybody to get the docker deployment up and running, and to be able to practice with. 

Written for people in automation practice.

# jenkins-docker-ci

We started with a common CI stack in Docker which runs Jenkins and Selenium Grid:

1. Jenkins
2. Selenium grid - selenium/hub:3.141.59-palladium
3. Chrome node - selenium/node-chrome:3.141.59-palladium
4. Firefox node - selenium/node-firefox:3.141.59-palladium

## Jenkins

Once Jenkins is up and running in Docker it can be connected to on the host machine in a browser by navigating to http://localhost:7000. The default login details for username and password are both 'admin'. This can be changed
The login details are user: `admin` and password: `admin`

Jenkins is run on a docker image that uses Ubuntu.

## C# Project

I used my own project to run this against: https://github.com/pfekete-ahq/code-camp-march-2022

We found that jenkins by itself was not able to run the .NET tests we were using. We were using MSTest for unit testing and using .NET 5.0.

## Dockerfile

Several packages needed to be installed inside the docker container in order to use Microsoft software products for Linux systems. Using apt we install wget and then use that to install packages-microsoft-prod.deb which is required to run .NET on Linux.

Included in the dockerfile are commands to install dotnet-sdk-5.0. This package is installed because it needs to match the version used in the Visual Studio project. If they do not match the tests will not run.

We do this with following code in the Dockerfile:

    # Install needed tools and upgrade installed packages
    RUN apt-get update
    RUN apt-get install wget
    RUN wget https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
    RUN dpkg -i packages-microsoft-prod.deb
    RUN rm packages-microsoft-prod.deb

    # install certificates
    RUN apt-get update
    RUN apt-get install ca-certificates

    # Install .NET SDK 5.0
    RUN apt-get update
    RUN apt-get install -y apt-transport-https
    RUN apt-get update
    RUN apt-get install -y dotnet-sdk-5.0
    RUN dotnet --version

These lines are run as root in order to have maximum permissions. 

Are this the root user add the jenkins user to the /etc/sudoers file. This way the jenkins user will be able to build and execute using the dotnet command.

    RUN echo "jenkins ALL=NOPASSWD: ALL" >> /etc/sudoers

The dockerfile then switches to the jenkins user and performs a set of tasks needed to use jenkins.

    # config
    COPY ./config.xml /usr/share/jenkins/ref/config.xml

This will copy the config.xml from the jenkins-docker-ci repository into the jenkins container.

    # initial admin user
    COPY security.groovy /usr/share/jenkins/ref/init.groovy.d/security.groovy

This will copy the security.groovy file in the jenkins container. Jenkins, when initialized, will execute all the Groovy scripts located in that directory. If you create additional setup scripts, that is the place you should place them. The security.groovy script creates the admin user with password admin. The problem with this is the credentials are hard-coded. This can be changed by leveraging Docker secrets. This requires the container be changed to a swarm services. We will not cover this here as it is (complicated) tale for another time.

    # plugins
    COPY ./plugins.txt /usr/share/jenkins/ref/plugins.txt
    RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

This will copy the plugins.txt file from the repository. This file is a list of plugins that the install-plugins.sh executable will install into the jenkins container.

As with security.groovy and config.xml this means they can be version controlled, and maintained without having to do it in the container. Insead they are copied into the container and automatically applied.

    # prevent banner prompt for pipeline plugins
    RUN echo 2.164.2 > /usr/share/jenkins/ref/jenkins.install.UpgradeWizard.state
    RUN echo 2.164.2 > /usr/share/jenkins/ref/jenkins.install.InstallUtil.lastExecVersion

## Run the docker setup

docker-compose build && docker-compose up 

NOTE: sometimes the volumes get into a state where it cannot be restarted due to an inability to write to the docker volume. In this case run: (need to be in root directory of this project)
docker-compose down --volumes 

When docker is up and running you should be able to open the following in the browser of the host machine (ie the computer you are running the docker containers on):
localhost:7000 <-- this will open jenkins
localhost:4444/grid/console <-- this will open the Selenium Hub grid console

# Setup the pipeline in jennkins

Open jenkins (localhost:7000, user=admin,password=admin).

Click 'New Item'

Enter an item name eg. web-playground.
Select 'Freestyle project'
Click OK

Under "Source Code Management" click the 'Git' radio button. This will expand a form for configuring the git repository for the pipeline to run from.

In 'Repository URL' enter the git repository url ie https://github.com/pfekete-ahq/code-camp-march-2022

Credentials are needed for jenkins to connect to the git repo. Under the 'Credentials' section, click on the 'Add' drop down and select the 'Jenkins' option. This should open a popup form "Jenkins Credentials Provider: Jenkins".

In this enter a username which can be anything to identify the user. This is significant to jenkins as an identifier and does not impact on the github repository. 

For the password you need to enter a personal access token which needs to be retrieved from the git repo. In the case of using github:
1. Login to the github account
2. Click on the user icon in the top right
3. Click on 'Settings'
4. In the settings page, click on 'Developer settings' in left hand side menu.
5. Click on 'Personal access tokens'
6. Click 'Generate new token' - you will probably be asked for your password.
7. The 'New personal access token' form should open. Enter a note and select an expiration period for the token.
8. In the 'Select scopes' you can define the access for the personal tokens. Tick the repo box. This is required to allow jenkins to pull the code when the pipeline runs. 
9. Copy the personal access token and copy it into the password field in the pipeline configuration.

Once the credentials are set up, add a build step. 
1. Click 'Add build step' drop down and select 'Execute shell'
2. In the command text box enter 'dotnet test'

In the page for the pipeline, click on 'Build Now' in the left hand menu. The a new pipeline run will appear in the 'Build history'. If you click on the build number this should open a console view where you can see the output of the pipeline.


# Change to the code

One problem we found is accessing the selenium-hub from within the docker container. When trying to connect to selenium-hub using localhost:4444 we encounter an error saying the connection has been refused. In order to deal with this we included the following code to instantiate the web driver:

if (OperatingSystem.IsWindows())
{
    driver = new RemoteWebDriver(remoteAddress: new Uri("http://localhost:4444/wd/hub"), options);
}
else if (OperatingSystem.IsOSPlatform("Linux"))
{
    driver = new RemoteWebDriver(remoteAddress: new Uri("http://selenium-hub:4444/wd/hub"), options);
}

We found that docker will handle selenium-hub::4444 and connect to the selenium-hub container, but running on a local machine requires localhost:4444. We declare them based on whether the machine is running Windows or if it's running Linux. The assumption is developers are using Windows and docker will be running Linux. This is not an ideal solution. The intention moving forwards is to have the network connection setup in the docker-compose.yml file.

Alternatively, we are able to use a different conditional. Instead of checking the OS, we can check to see if a '.dockerenv' file is in the root directory. Docker creates this file at the top of the container's directory tree. This is far more reliable than the previous conditional.

if (DockerHelper.IsRunningInDocker())
{
    driver = new RemoteWebDriver(remoteAddress: new Uri("http://selenium-hub:4444/wd/hub"), options);
}
else
{
    driver = new RemoteWebDriver(remoteAddress: new Uri("http://localhost:4444/wd/hub"), options);
}

This conditional uses the following method in the DockerHelper class:

public static bool IsRunningInDocker()
{
    return File.Exists(@"/.dockerenv");
}

If the file exists then the code should be satisfied that it is running in a docker container. The only drawback is if a developer has the '.dockerenv' file in their root directory. However, this is very unlikely and therefore safer to use than checking which OS is being run. (The developer could be running their code on Linux or the code would have to accomodate for other OS ie Mac).


# Troubleshooting

docker.errors.InvalidArgument: "host" network_mode is incompatible with port_bindings

This occurred when executing 'docker-compose up' on the command line. 