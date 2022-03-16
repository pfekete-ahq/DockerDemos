# jenkins-docker-ci

A CI stack in Docker which runs:

1. Jenkins
2. Selenium Grid
3. Chrome
4. Firefox

## Jenkins

The jenkins url is http://localhost:7000
The login details are user: `admin` and password: `admin`

## C# Project

I used my own project to run this against: https://github.com/pfekete-ahq/code-camp-march-2022

## Dockerfile

Included in the dockerfile are commands to install dotnet-sdk-5.0

We chose this because it matches the version of dotnet used in the project we are building.

## Run the docker setup

docker-compose build && docker-compose up 

NOTE: sometimes the volumes get into a state where it cannot be restarted due to an inability to write to the docker volume. In this case run: (need to be in root directory of this project)
docker-compose down --volumes 

When docker is up and running you should be able to open the following:
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