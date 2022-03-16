# intro

Docker compose group as a starting point, we created a dotnet point. For use as a starting point or practice. Written for people in automation practice.

# jenkins-docker-ci

We started with a CI stack in Docker which runs:

1. Jenkins
2. Selenium grid - selenium/hub:3.141.59-palladium
3. Chrome node - selenium/node-chrome:3.141.59-palladium
4. Firefox node - selenium/node-firefox:3.141.59-palladium

## Jenkins

The jenkins url is http://localhost:7000
The login details are user: `admin` and password: `admin`

Jenkins is run on a docker image that uses Ubuntu.

## C# Project

I used my own project to run this against: https://github.com/pfekete-ahq/code-camp-march-2022

We found that jenkins by itself was not able to run the .NET tests we were using. We were using MSTest for unit testing and using .NET 5.0.

## Dockerfile

Several packages needed to be installed inside the docker container in order to use Microsoft software products for Linux systems. Using apt we install wget and then use that to install packages-microsoft-prod.deb which is required to run .NET on Linux.

Included in the dockerfile are commands to install dotnet-sdk-5.0. This package is installed because it needs to match the version used in the Visual Studio project. If they do not match the tests will not run.

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

We found that docker will handle selenium-hub::4444 and connect to the selenium-hub container, but our local machine requires localhost:4444. We declare them based on whether the machine is running Windows or if it's running Linux. The assumption is developers are using Windows and docker will be running Linux. This is not an ideal solution. The intention moving forwards is to have the network connection setup in the docker-compose.yml file.