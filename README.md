# Running SonarQube on each pull request through jenkins

## Useful info
- Default SonarQube user: admin/admin
- Web interface: http://localhost:9000

## Prerequisites
1. Java installed
2. Jenkins up and running
3. MySQL installed

# Installing SonarQube
1. Download SonarQube from (LTS was used): https://www.sonarqube.org/downloads/
2. Extract. From here on refered to as `<sonarqube-dir>`.

# Database connection
Using information from: http://www.jouvinio.net/wiki/index.php/Sonar_Configuration_MySQL
## Database setup
1. Create new schema for sonarqube:
```mysql
mysql> create database sonar CHARACTER SET UTF8;
```

2. Create user: (replace `<SONAR_USER>` and `<SONAR_PASSWORD>` with your own)
```mysql
mysql> create user '<SONAR_USER>'@'localhost' identified by '<SONAR_PASSWORD>';
```

3. Grant rights of sonar user to sonar schema:
```mysql
mysql> grant all on sonar.* to '<SONAR_USER>'@'localhost';
mysql> FLUSH PRIVILEGES;
```

## SonarQube settings
Open `<sonarqube-dir>/conf/sonar.properties`  
Under DATABASE, fill in and uncomment:
```properties
sonar.jdbc.username=<SONAR_USER>
sonar.jdbc.password=<SONAR_PASSWORD>
```
  And a bit lower uncomment
```properties
sonar.jdbc.url=jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance
```
Make sure the Embedded database (just above MySQL) is commented out
		
# Plugins

## SonarQube plugins 
Put plugins in `<sonarqube-dir>/extensions/plugins/`.

### GitHub plugin ([download](https://docs.sonarqube.org/display/PLUG/GitHub+Plugin))

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
See below for full configuration.

### Build Breaker plugin for SonarQube ([download](https://github.com/SonarQubeCommunity/sonar-build-breaker))

Lets SonarQube fail the jenkins build on certain conditions.  
When used from jenkins:  
```
sonar.buildbreaker.skip=(true|false)
```  
Default is `false`.  
Should be set to `true` on pull requests as the failure will be reported directly by SonarQube using the GitHub plugin.

## Jenkins plugins

### SonarQube Scanner for Jenkins ([download](https://docs.sonarqube.org/display/SCAN/Analyzing+with+SonarQube+Scanner+for+Jenkins))

Easy integration of SonarQube into jenkins (gives you a build block that is used to start the scans).

### GitHub Pull Request Builder plugin ([download](https://wiki.jenkins.io/display/JENKINS/GitHub+pull+request+builder+plugin))

Triggers the jenkins build on webhook or cron (default 5 min) and generates the needed environment variables.
			
# Starting SonarQube:
1. Run `<sonarqube-dir>/bin/linux-x86-64/sonar.sh { console | start | stop | restart | status | dump }`
   - `console` to see what is happening.
   - `start`, `stop`, `restart`, `status` otherwise.
2. Web interface at `http://localhost:9000`
3. Login with `admin/admin`
	
# Jenkins build setup:
## Configuring plugins
1. Open jenkins configure screen
    - Jenkins -> Manage Jenkins -> Configure System
1. Under SonarQube servers ADD IMAGE HERE
    - Add server information.  
      - To get `Server authentication token`:  
		Go to SonarQube web interface  
		Login as admin  
		Go to Administation (top bar) -> Security (dropdown) -> Users -> Tokens (right side, per user) -> Generate

1. Under GitHub Pull Request Builder
	Add secret (optional)
	Add credentials (github-user/github-oauth-token)  
    - Recommended to run this from a seperate 'bot' account  
		Needs 'repo' permissions, including `repo:status`, `repo_deployment` and `public_repo`.  
		Account needs at least pull access. Push access lets it put the results as 'tests' instead of comments, meaning the "Merge" button turns red on fail. 

## Setting up Freestyle Project for Pull Requests ADD PICTURES
- Create a new freestyle project
- Jenkins -> New Item -> Freestyle project
### General
- (Optional) GitHub project (adds link to the github repository on the build page in jenkins)
### Source Code Management
- Git
  - Add repository and credentials
  - Under Advanced  
  name: `origin`  
  refspec: `+refs/pull/*:refs/remotes/origin/pr/*`  
  Other refspec configurations:  
  
  | refspec | use | recommened by |
  |---------|-----|---------------|
  |`+refs/pull/*:refs/remotes/origin/pr/*`| Get all pull requests | [link](http://macoscope.com/blog/using-sonarqube-with-jenkins-continuous-integration-and-github-to-improve-code-review/)|
  |`+refs/pull/${ghprbPullId}/*:refs/remotes/origin/pr/${ghprbPullId}/*`| Get only current pull request |[link](https://github.com/jenkinsci/ghprb-plugin#creating-a-job) |
  |`+refs/heads/*:refs/remotes/origin/* +refs/pull/${ghprbPullId}/*:refs/remotes/origin/pr/${ghprbPullId}/*`| Get branches and current pull request|[link](https://github.com/jenkinsci/ghprb-plugin#creating-a-job)

  - Set "Branches to build" to `${sha1}` (environment variable is set by GitHub Pull Request Builder)

### Build Triggers
- Check GitHub Pull Request Builder
  - Select credentials (from configuring plugins step)
  - Under advanced set options. Recommended to:  
    - On "List of organizations" add `<organization>` to prevent the "Request for testing phrase" (configured in plugin settings) to trigger. Another option is to whitelist the users instead.
  - If using webhook, point it at `http://localhost:8080/jenkins/ghprbhook/`

### Build Environment
- No specific requirements

### Build
- Add build block 'Execute SonarQube Scanner'
  - Under "Analysis properties" (replace `<organization>` and `<repository>` accordingly):
```properties
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

# Disable build breaker (set to true when triggering on pull requests, false for nightly)
sonar.buildbreaker.skip=true

# To change other default behaviour, find the behaviour in SonarQube -> Administation to find the keys.
# These keys can be added to this section
```
  - Under "Additional arguments":
    - Add `-X` for full debugging, otherwise leave empty.
