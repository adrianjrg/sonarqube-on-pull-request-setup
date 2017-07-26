# Running SonarQube on each pull request

# Installing SonarQube and integrating with jenkins

## Useful info
- Default SonarQube user: admin/admin
- Web interface: http://localhost:9000

## Prerequisites
1. Java installed
2. Jenkins up and running
3. MySQL installed

# Installing SonarQube
1. Download SonarQube from (LTS was used): https://www.sonarqube.org/downloads/
2. Extract

# Database connection
Using information from: http://www.jouvinio.net/wiki/index.php/Sonar_Configuration_MySQL
## Database setup
1. Create new schema for sonarqube:
```mysql
create database sonar CHARACTER SET UTF8;
```

2. Create user: (replace `<SONAR_USER>` and `<SONAR_PASSWORD>` with your own)
```shell
mysql> create user '<SONAR_USER>'@'localhost' identified by '<SONAR_PASSWORD>';
```

3. Grant rights of sonar user to sonar schema:
```shell
mysql> grant all on sonar.* to '<SONAR_USER>'@'localhost';
mysql> FLUSH PRIVILEGES;
```

## SonarQube settings
Open `sonarqube-5.6.6/conf/sonar.properties`

1. Under DATABASE, fill in and uncomment:
```properties
sonar.jdbc.username=<SONAR_USER>
sonar.jdbc.password=<SONAR_PASSWORD>
```
  And a bit lower uncomment
```properties
sonar.jdbc.url=jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance
```
2. Make sure the Embedded database (just above MySQL) is commented out
		
# Plugins:
There are a number of plugins for both SonarQube and Jenkins that are useful
## SonarQube plugins (put them in sonarqube-5.6.6/extensions/plugins/):

### GitHub plugin at https://docs.sonarqube.org/display/PLUG/GitHub+Plugin

Lets SonarQube put its analysis and comments on GitHub.

Needs when used from jenkins:
```
sonar.github.oauth=<github-oauth-token>
sonar.github.pullRequest=${ghprbPullId}
```
${ghprbPullId} is the pull request number gotten from jenkins
```
sonar.github.repository=<user>/<repository> or <organization>/<repository>
```

### Build Breaker plugin for SonarQube at https://github.com/SonarQubeCommunity/sonar-build-breaker

Lets SonarQube fail the jenkins build on certain conditions  
- When used from jenkins:  
   `sonar.buildbreaker.skip=(true|false)`  
   Default: false  
True should be used on pull requests as the failure will be reported directly by SonarQube using the GitHub plugin  

## Jenkins plugins:

1.	SonarQube Scanner for Jenkins at https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner+for+Jenkins

Easy integration of SonarQube into jenkins (gives you a build block that is used to start the scans)

2. 	GitHub Pull Request Builder plugin at https://wiki.jenkins.io/display/JENKINS/GitHub+pull+request+builder+plugin

Triggers the jenkins build on webhook or cron (default 5 min) and generates the needed environment variables
			
# Starting SonarQube:
1. Run `sonarqube-5.6.6/bin/linux-x86-64/sonar.sh { console | start | stop | restart | status | dump }`
2. Web interface at http://localhost:9000
3. Login admin/admin
	
# Jenkins build setup:
## Configuring plugins
  - Jenkins -> Manage Jenkins -> Configure System
  - Under SonarQube servers
	Add server information. 
	To get "Server authentication token":
		Go to SonarQube web interface
		Login as admin
		Go to Administation (top bar) -> Security (dropdown) -> Users -> Tokens (right side, per user) -> Generate

  - Under GitHub Pull Request Builder
	Add secret (optional)
	Add credentials (github-user/github-oauth-token)
		Recommended to run this from a seperate 'bot' account
		Needs 'repo' permissions, including 'repo:status', 'repo_deployment' and 'public_repo'
		Account needs at least pull access. Push access lets it put the results as 'tests' instead of comments, meaning the "Merge" button turns red on fail. 

## Setting up Freestyle Project for Pull Requests
- Jenkins -> New Item -> Freestyle project
  ### General
   - (Optional) GitHub project (adds link to the github repository on the build page in jenkins)
  ### Source Code Management
   - Git
   - Add repository and credentials
   - Under Advanced
	name: origin
	refspec: +refs/pull/*:refs/remotes/origin/pr/*
	alternate refspec:
		recommended: http://macoscope.com/blog/using-sonarqube-with-jenkins-continuous-integration-and-github-to-improve-code-review/
			refspec that worked on testing:		+refs/pull/*:refs/remotes/origin/pr/*
		recommended on: https://github.com/jenkinsci/ghprb-plugin#creating-a-job
			refspec that's recommended for pr: 	+refs/pull/${ghprbPullId}/*:refs/remotes/origin/pr/${ghprbPullId}/*
			refspec for pr + branch:			+refs/heads/*:refs/remotes/origin/* +refs/pull/${ghprbPullId}/*:refs/remotes/origin/pr/${ghprbPullId}/*

   - Set "Branches to build" to ${sha1} (set by GitHub Pull Request Builder)

  ### Build Triggers
   - Check GitHub Pull Request Builder
    - Select credentials (from configuring plugins step)
    - Under advanced set options
    Recommended:
     On "List of organizations" add: "<organization-name>" to prevent the "Request for testing phrase" (configure in plugin settings) to trigger
     Another option is to whitelist the users instead
    - If using webhook, point the webhook at http://localhost:8080/jenkins/ghprbhook/

  ### Build Environment
   - No specific requirements

  ### Build
   - Add build block 'Execute SonarQube Scanner'
    - Under "Analysis properties":
```
# SonarQube server URL
sonar.host.url=http://localhost:9000/

# must be unique in a given SonarQube instance, can be whatever you want
sonar.projectKey=com:<organization>:<repository>
# this is the name and version displayed in the SonarQube UI. Was mandatory prior to SonarQube 6.1.
sonar.projectName=<repository>
# Project version does not matter when in "preview" mode as nothing is saved to the db
# "preview" mode is used when triggered by pull requests
# projectVersion can be any string
sonar.projectVersion=${ghprbPullId}
#sonar.projectVersion=origin/develop

# Path is relative to the sonar-project.properties file. Replace "\" by "/" on Windows.
# This property is optional if sonar.modules is set. 
# "." results in every file being scanned
sonar.sources=.

# Encoding of the source code. Default is default system encoding. Important as MySQL is set up for this encoding
sonar.sourceEncoding=UTF-8

# Github comment settings
sonar.analysis.mode=preview
# Unneeded
#sonar.github.login=<user>
sonar.github.oauth=<github-oauth-token>
sonar.github.pullRequest=${ghprbPullId}
sonar.github.repository=<repository>

# Disable build breaker (no need in this job as SonarQube will report the issues directly to Github pull request as inline comment / regular comment)
sonar.buildbreaker.skip=true

# To change other default behaviour, find the behaviour in SonarQube -> Administation to find the keys.
# These keys can be added to this section
```
Under "Additional arguments":
- Add '-X' for full debugging, otherwise leave empty
