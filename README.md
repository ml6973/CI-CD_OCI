CI/CD Documentation
The purpose of this documentation is to demonstrate how to implement a Continuous Integration/Continuous Deployment Pipeline utilizing Jenkins, Salt, and Selenium.

Overview
Daily Development

DailyDevelopment.png

This diagram explains the daily development cycle of a project.
1.) First you pull the integration branch. This is the branch where the current working code for the development version is.
2.) Create a local integration branch to make changes to.
3.) After you finish making changes, push your local integration branch to the remote test/* directory. Giving the branch a name of your choosing.
4.) When Jenkins detects a change in that git directory, it triggers the daily pipeline to run.
5.) It runs the basic smoke tests against the code that got pushed to test/*.
6.) If it passes the tests, it logs it and merges that branch with integration.
Nightly Testing and Merging

NightlyTestAndMerge.png

This diagram represents the integration/deployment process at night in the off peak hours. This would take place after a day of people working on the integration branch.
1.) A nightly trigger(usually a set cron time) initiates and Jenkins pulls the current integration branch.
2.) It runs the smoke Tests against it.
3.) If the smoke tests pass, then it builds and deploys the code.
4.5.6.) Uses the Salt tool to spin up cloud instance with the project running.
7.) This is where the extensive tests and UI tests are used.
6.) If all tests are passsed, it pushes the current integration back up to master as all the new code is working. Thus, becoming the next day's integration branch to be edited.
Release Deployment

ReleaseAndDeploy.png

This is the process of deploying a new release when the master branch is at a point to do so, as determined by the developers.
Salt/Jenkins Installation
(Ubuntu 14.04 Cloud Ready)

Create a cloud VM with a public IP


sudo add-apt-repository ppa:saltstack/salt

sudo apt-get update

apt-get install -y salt-api salt-cloud salt-master salt-minion salt-ssh salt-syndic 

MASTER_IP= <MASTER IP>
USERNAME= <USERNAME>
PASSWORD= <PASSWORD>
TENANT= <TENANT NAME>

cat <<EOF | tee -a /etc/salt/cloud.providers.d/chameleon.conf
chameleon-config:
  # Set the location of the salt-master
  #
  minion:
    master: $MASTER_IP

  # Configure the OpenStack driver
  #
  identity_url: https://openstack.tacc.chameleoncloud.org:5000/v2.0/tokens
  compute_name: nova
  protocol: ipv4

  compute_region: RegionOne

  # Configure Openstack authentication credentials
  #
  user: $USERNAME
  password: $PASSWORD
  # tenant is the project name
  tenant: $TENANT

  driver: nova
  provider: openstack

  # skip SSL certificate validation (default false)
  insecure: false

  networks:
    - fixed:
      - a89ad160-c3ac-4c42-b9a9-2ba910442fb3
    - floating:
      - ext-net

cat <<EOF | tee -a /etc/salt/cloud.profiles.d/elgg_profile.conf
standard_profile:
  provider: chameleon-config
  size: m1.large
  image: Ubuntu 14.04 Cloud Ready
  ssh_key_file: /var/lib/jenkins/.ssh/id_rsa
  ssh_key_name: jenkins_key
  ssh_interface: private_ips
  ssh_username: ubuntu
EOF

mkdir -p /srv/salt
mkdir -p /srv/pillar

cd /srv/salt
git clone https://github.com/ml6973/jenkins-formula.git
git clone https://github.com/ml6973/elgg-formula.git

SMTP_USERNAME= <SMTP EMAIL ADDR>
SMTP_PASSWORD= <SMTP PASSWORD>
MYSQL_PASSWORD= <PASSWORD>
JENKINS_IP= <IP>
JENKINS_ENDPOINT='http://$JENKINS_IP:8080'
 


cat <<EOF | tee -a /srv/salt/top.sls
base:
  '*jenkins*':
    - jenkins-formula.jenkins
  '*elgg*':
    - elgg-formula.elgg
    - jenkins-formula.jenkins.slave
EOF

cat <<EOF | tee -a /srv/pillar/top.sls
base:
  '*jenkins*':
    - jenkins
  '*elgg*':
    - elgg
    - jenkins
EOF


cat <<EOF | tee -a /srv/pillar/elgg.sls
mysql:
  server:
    root_password: $MYSQL_PASSWORD
smtp:
  server: ssl://smtp.gmail.com
  port: 465
  username: $SMTP_USERNAME
  password: $SMTP_PASSWORD
EOF

cat <<EOF | tee -a /srv/pillar/jenkins.sls
jenkins:
  lookup:
    port: 80
    home: /var/lib/jenkins
    user: jenkins
    group: jenkins
    master_url: JENKINS_ENDPOINT
    slave:
      user: slave
      password: slavepass
EOF

sed -i 's/#master: salt/master: localhost/' /etc/salt/minion

sed -i 's/#user: root/user: jenkins/' /etc/salt/master

cat <<EOF | tee -a /etc/salt/cloud
query.selection:
  - private_ips
EOF

chown -R jenkins: /etc/salt /var/cache/salt /var/log/salt /var/run/salt

service salt-master restart
service salt-minion restart

salt-key -y -A
On the Jenkins Master Node


su jenkins
cd ~/.ssh
ssh-keygen

cat ~/.ssh/id_rsa.pub
Jenkins Configuration
Afterthe Salt/Jenkins Installation, follow Install Wizard from Browser at :

<Floating IP>:8080

Jenkins Plugins needed:

Pipeline
Node/Label
Xvnc
Thin Backup
Swarm
Jenkins Pipeline SetUp
In Jenkins we will primarily be using two different types of Jenkins projects to construct the overall Pipeline. These are called "Freestyle" projects and "Pipeline" projects. While the entire process is "The Pipeline" we will using these two different kinds of Jenkins projects to define it.
Now That Jenkins and Salt are set up on your node, log into the web interface at http://your-jenkins-ip:8080 with the credentials you set so that you can begin defining the jobs.

WebHook

Before getting starting with creating Jenkins jobs, you must be aware that for Jenkins to communicate with your GitHub repo, you must have the Jenkins GitHub Plug-in installed and a webhook created on the GitHub site.

I wanted to metion this now so that as I describe the jobs to create that talk to GitHub, you know that this has been done.

Details on how to do this can be found here: Jenkins/GitHub WebHook Set Up

The actual Jenkins GitHub Plug-in info can be found here: Jenkins GitHub Plugin

Credential Information: ( NEED TO PUT BETTER INFO ON THIS )

The Daily Pipeline

This is the series of jobs ( in the order that they execute ) that will perform the "Daily Development" diagram detailed in the "Overview" section above.

Freestyle Project 1 : TriggerDailyPipeline

This job will be watching for any new changes pushed to the '/test/*' branch on the GitHub repo. Upon seeing changes, it will initialize the actual Jenkins Pipeline project called PIPELINE_DAILY and pass GitHub paramters to it.
From the Jenkins Dashboard select 'New Item' and choose 'Freestyle Project'
In the 'Source Code Management' section choose Git. (WebHook Applies Here)
Fill out your 'Repository URL' and credentials.
Now tell it to watch the '*/test/*' branch in the "Branches to build" section
"Repository Browser" is '(Auto)'
Check the "Build when a change is pushed to GitHub" in the "Build Triggers"
In the "Build" section select the "Trigger/call builds on other projects"
In "Projects To Build" define your next job to run(The Daily Pipeline project will be next). Call it: PIPELINE_DAILY
In the Predefined Parameters section declare:
GITURL=$GITURL
GITBRANCH=$GITBRANCH
Click Save
Pipeline Project 1 : PIPELINE_DAILY

After the previous Freestyle Job is triggered, it will initiate this Pipeline Job while passing it GitHub parameters(GITURL and GITBRANCH). This Pipeline Job will then manage the flow of the next Freestyle Jobs to be executed. To put it simply, Freestyle Jobs define tasks while Pipeline Jobs manage the execution of those tasks.
From the Jenkins Dashboard select 'New Item' and choose 'Pipeline Project'
In the 'General' section choose "This Project Is Parameterized" .
Define Three String Parameters
Name: GIT_URL, Description: ""
Name: GIT_BRANCH, Description: ""
Name: MERGE_BRANCH, Description: "Integration"
Now go down to the "Pipeline" section and give a name to your Pipeline Script. This script uses .groovy syntax and has assistance if you click the "Pipeline Syntax" link
Enter the following code to execute the next jobs sequentially:
node {
    def smoke_tests = build job: "TESTS_SMOKE", parameters: [[$class: 'StringParameterValue', name: 'GIT_URL', value: "$GIT_URL"], [$class: 'StringParameterValue', name: 'GIT_BRANCH', value: "$GIT_BRANCH"]]
    
    def merge_code = build job: 'MERGE', parameters:[[$class: 'StringParameterValue', name: 'GIT_URL', value: "$GIT_URL"], [$class: 'StringParameterValue', name: 'GIT_BRANCH', value: "$GIT_BRANCH"], [$class: 'StringParameterValue', name: 'MERGE_BRANCH', value: "$MERGE_BRANCH"]]
}
The "build job" command initates the jobs, while "parameters" are values that get passed to these jobs. These parameters must be defined in the jobs they are being passed to before they can be used in those jobs. An example of this is defining the paramters in the "This Project Is Parameterized" section.
Click Save
Freestyle Project 2 : TESTS_SMOKE

This is the first Freestyle Job called in PIPELINE_DAILY. This job runs the first round of basic tests against the new code changes.
From the Jenkins Dashboard select 'New Item' and choose 'Freestyle Project'
In the 'General' section choose "This Project Is Parameterized" .
Define Three String Parameters
Name: GIT_URL, Description: ""
Name: GIT_BRANCH, Description: ""
In the 'Source Code Management' section choose Git. (WebHook Applies Here)
Fill out your "Repository URL" as '$GIT_URL' , and then set credentials.
Now set '$GIT_BRANCH' branch in "Branches to build"
"Repository Browser" is '(Auto)'
In the "Build" section, tell it to initalize your Smoke_Tests in whatever form they may be in. Currently ours is set to a shell script: sh bash tests/smoke_tests.sh > Keep in mind Smoke Tests are your first round of basic tests against the code. These tests can be written in any manner that serves your code, the "Build" section in this job is where you would put logic to run those. In these smoke tests, there will also be logic that defines pass or failure.
Click Save
Freestyle Project 3 : MERGE

This job will be called from the Pipeline job after a successful run of the TESTSSMOKE job. The purpose of this job is to merge the successfuly tested code back into the "Integration" branch from the "/test/*" branch. This job is made to take arbitray branch names so it can merge flexibly. The parameters used in this job are set in the PIPELINEDAILY job.
From the Jenkins Dashboard select 'New Item' and choose 'Freestyle Project'
In the 'General' section choose "This Project Is Parameterized" .
Define Three String Parameters
Name: GIT_URL, Description: ""
Name: GIT_BRANCH, Description: ""
Name: MERGE_BRANCH, Description: ""
In the 'Source Code Management' section choose Git. (WebHook Applies Here)
Fill out your "Repository URL" as '$GIT_URL, and set credentials.
Now set the '$GIT_BRANCH' branch in the "Branches to build" section
"Repository Browser" is '(Auto)'
Add an "Additional Behaviour" called "Merge Before Build"
Name of repository : origin
Branch to merge : $Merge_Branch
Merge Strategy : default
Fast-forward mod : --ff
In the "Post-build Actions" section select "Git Publisher"
Check the "Push Only If Build Succeeds" option
In "Branches"
Branch to push : $MERGE_BRANCH
Target remote name : origin
Click Save
Testing

To test the jobs we just made, assuming everything was input correctly and your GitHub project is structured properly: pull down your Git repo, make changes in "Integration", then push that back to the remote branch folder "test/" as a new branch with a name of your choosing. As soon as you make that push, the TriggerDailyPipeline job should kick in and start running the rest of the jobs. If everything was successful you should see your new changes merged into "Integration" on GitHub afer the completion of the PIPELINE_DAILY job.

TroubleShooting

If you are having issues achieving a successful run, you can look at the individual console logs of each job from the Jenkins portal. Each run of every job is logged and will help in identifying any issues.

If you pinpoint an issue as local to a single job, you can also build and run jobs singularly from the Jenkins Dashboard. Just cick on an individual job and click "Build" or "Build with Parameters" based on how the job is defined.
