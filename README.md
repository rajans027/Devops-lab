# Devops-lab

Project also a wiki for better readability https://github.com/rajans027/Devops-lab

Process followed
Started with starting two EC2 Ubuntu instances on AWS. ( Master and test server). I am communicating with these nodes via SSH

On master instance, I have installed java as a prereq to facilitate Jenkins installations https://www.theserverside.com/blog/Coffee-Talk-Java-News-Stories-and-Opinions/jenkins-ubuntu-20-install-git-jdk-java-ci-cd
Allow Jenkins to communicate on port 8080, and make the necessary changes to the firewall (ufw in this case)

sudo ufw allow 8080

NOTE: when you able ufw firewall, remember to allow port 22 as well to prevent loss of ssh access Make sure to enable ufw with default settings first and then move on to enable ufw. Order mentioned below sudo ufw default allow $ sudo ufw enable sudo ufw allow 22/tcp Restart Jenkins service if needed

Move on to allow port 8080 traffic in your AWS security group.

Startup Jenkins in your browser with publicip:8080. Jenkins start-up page should show up at this stage.

JENKINS CONFIG
After accessing Jenkins on your browser, it will provide us instructions to log in with our first pre-set admin password. Upon logging in I changed the password and filled in other metadata for your account. Once the account is set up, proceeded to manage nodes and clouds under Manage Jenkins. Added my desired test server instance (ie: test server on AWS).

Provided a root directory for Jenkins to store its logs and other info.

Connected the new agent to the Jenkins cluster in one of these ways:

Connect agent to the cluster by running below command using the provided jnlp file

sudo java -jar agent.jar -jnlpUrl http://3.84.108.211:8080/computer/test%2Dserver/jenkins-agent.jnlp -secret a0856042613d0014643c034264de6e6fb8f3a3326bc1dcff9ebdf42dd9e98977 -workDir "/home/ubuntu/jenkins"

Run from the agent command line using the provided command and jar file.

In order to connect this new slave agent to the Jenkins cluster, ie the master node we worked on AWS. I downloaded the above-mentioned .jar files provided by Jenkins and shared them with our slave node using Filezilla.

SSH into the slave- update the baseline Linux and continued with the installation of Jenkins as I did for Master. Ran the command provided below in the slave terminal

sudo java -jar agent.jar -jnlpUrl http://3.84.108.211:8080/computer/test%2Dserver/jenkins-agent.jnlp -secret a0856042613d0014643c034264de6e6fb8f3a3326bc1dcff9ebdf42dd9e98977 -workDir "/home/ubuntu/jenkins"

Once the above cmd executes successfully, our agent/slave is successfully connected to the Jenkins cluster. Status should read INFO: Connnected. This will also reflect in Jenkins GUI.

Started a new terminal and ssh into the agent/slave again to perform Git setup.

GIT SETUP ON SLAVE
Initiated git on the slave node using sudo git init Cloned my git repo using sudo git clone "https://github.com/rajans027/Devops-lab" Also performed sudo git add . to create a staging area, any changes made to the files on the repo will not be altered unless a git commit is also executed by the devs. example git commit cmd sudo git commit -m"first commit"

Integrating Jenkins to collect info from Github
#setup webhook on Github Go to repo settings on GH and choose webhooks -> add webhooks Used below info payload= (Jenkins URL) http://3.84.108.211:8080/github-webhook/

The functionality of this webhook is pretty simple, any PUSH committed to the rep will be notified to our Jenkins. Created a new Jenkins job, a freestyle project to receive info from the changes being made to Github on slave node only.

Project URL - used my GitHub repo link Label expression- restricted to test-server (this has to match the agent on which git changes are being made, in our case Test server is where we are pushing Git changes via CLI Under source code management, choose Git and provide the repo URL Mention the branch link (/main or /master) depends on your repo where the changes are being committed and pushed Under triggers use the Github trigger provided and execute a custom shellcode to reflect successful run. Save the job settings. Now when I make changes to index.html with nano. Followed by the below commands

sudo git add . (to add it to the staging area)

sudo git commit -m"random name to this change"

sudo git push origin main (this will prompt you with username and password (personal access token)

I can now see a building history develop in Jenkins, keeping a track of changes made.

Gitflow branching
Create a separate branch on slave

sudo git branch develop (to create develop branch)

sudo git branch( to view the branches)

sudo git checkout develop (to go inside develop branch)

Any changes made to the "develop" will only affect the main branch when a merge is done. In DevOps rarely changes are made directly to the main repo, branching it further becomes an integral part of the lifecycle. We can create a separate job or tweak the existing Jenkins job by adding the */develop branch in the configuration. This way we can have separate triggers set up for changes made to develop and the main branch.

For example, let's make a Dockerfile on the develop branch in slave and create a new job on Jenkins (build-website) with an executable to create a Docker container using the below command.

sudo docker rm -f $(sudo docker ps -a -q)

sudo docker build /home/ubuntu/Devops-lab/. -t test

sudo docker run -it -p 82:80 -d test

Our docker file is simple:

FROM hshar/webapp

ADD . /var/www/html

Now we can merge the main and develop branch and then git add, commit and push the changes. Once it's pushed we will see a build-website job being triggered on Jenkins and now if we go on slave-agent on AWS public IP with port:82 we can see the website and the content.

We will create and re-arrange the Jenkins jobs in order to build a pipeline in the following downstream order (post-build actions)

Webhook gets triggered when a change is made to the repository.

If develop branch (repo) is changed, we will build the docker container and push our website on test-server only
If the main branch (repo) is changed, we will build the docker container and push our website on the test server, if the test server build is stable the pipeline further continues and builds the docker container and the website is pushed on the prod server's public IP.
Setting up Puppet to establish configuration management
Puppet architecture for this lab One master (same host we have our Jenkins on), two-agent nodes (test and prod servers).

Installed puppet on my master instance using the official guide https://puppet.com/docs/puppet/7/install_puppet.html ON MASTER:

sudo apt-get update

sudo apt-get install wget (if needed)

wget https://apt.puppetlabs.com/puppet-release-bionic.deb

sudo dpkg -i puppet-release-bionic.deb

sudo apt-get update

apt policy puppet-master

sudo apt-get install puppet-master

Allocated 512Mb to the puppet by modifying the /etc/default/puppetserver
allowed port 8140/tcp on the firewall
define the hosts in /etc/hosts/ - add your public ip along with the name puppet
create a directory for manifests, /etc/puppet/code/environments/productions/manifests
restart puppetserver
ON AGENT:

sudo apt-get update

sudo apt-get install wget

wget https://apt.puppetlabs.com/puppet-release-bionic.deb

sudo dpkg -i puppet-release-bionic.deb

sudo apt-get update

sudo apt-get install puppet

/etc/hosts add my master's public ip with the name puppet

Followed the same procedure on the prod server we have.

Back on the master node

sudo puppet cert list -all

sudo puppet cert sign --all

Basic conditions added to the manifest file on Master: I am naming this file as site.pp

node default{ exec{'Conditions': command=> ' /bin/echo "Apache is isntalled" > /home/ubuntu/config-management/status.txt', onlyif=> ' /binwhihch apache2', } }

{ exec{"Conditions': command=> 'bin/echo "Apache is not installed" > /home/ubuntu/config-management/status.txt', unless => '/bin/whihch apache2', } }

On your agent node (test or prod) run the command

sudo puppet agent --test

This test fails in my lab because the apache server is running in the container we spin up every time with the help of our Jenkins pipeline. As a test, installed apache2 and the file status.txt showed Apache is installed This manifest file can be written and configured in any desired way, the idea is to create a file and let the nodes know whether the expected conditions/configurations are met.

FINAL STEP- MONITORING USING NAGIOS
sudo apt-get update

sudo apt-get install wget build-essential unzip openssl libssl-dev

sudo systemctl disable puppet-master

sudo apt-get install apache2 php libapache2-mod-php php-gd libgd-dev

sudo adduser nagios

sudo groupadd nagcmd

sudo usermod -a -G nagcmd nagios

sudo usermod -a -G nagcmd www-data

cd /tmp

wget -O nagioscore.tar.gz https://github.com/NagiosEnterprises/nagioscore/archive/nagios-4.4.5.tar.gz

ls tar xzf nagioscore.tar.gz

cd /tmp/nagioscore-nagios-4.4.5/

sudo ./configure --with-httpd-conf=/etc/apache2/sites-enabled

sudo make all

sudo make install

sudo make install-init

sudo make install-commandmode

sudo make install-config

sudo make install-webconf

sudo a2enmod rewrite

sudo a2enmod cgi

sudo ufw allow Apache

sudo ufw reload

sudo htpasswd -c /usr/local/nagios/etc/htpasswd.users nagiosadmin

sudo service apache2 restart

sudo systemctl start nagios.service

sudo apt-get install -y autoconf gcc libc6 libmcrypt-dev make libssl-dev wget bc gawk dc build-essential snmp libnet-snmp-perl gettext

cd /tmp

wget --no-check-certificate -O nagios-plugins-2.2.1.tar.gz https://github.com/nagios-plugins/nagios-plugins/releases/download/release-2.2.1/nagios- plugins-2.2.1.tar.gz

tar zxf nagios-plugins-2.2.1.tar.gz

cd nagios-plugins-2.2.1/

sudo ./configure

sudo make

sudo make all

sudo make install

sudo systemctl start nagios.service

sudo systemctl restart nagios.service

sudo systemctl status nagios.service

On node (in this case our prod instance where the live website is up)

sudo apt-get install nagios-nrpe-server nagios-plugins

sudo nano /etc/nagios/nrpe.cfg

add this to the cfg file allowed_hosts=masternodeip and restart nagios

On master define the host info cd /usr/local/nagios/etc/objects

sudo nano localhost1.cfg

add prod server info

define host {

use hostname alias address }

sudo nano nagios.cfg

under defining monitoring add the above files path cf_file=/usr/local/nagios/etc/objects/localhost1.cfg

added a simple check for http service monitoring

define service {

name Http-service

use generic-service

hostname nagios

service_description HTTP

check_command check_http

check_interval 1

retry_interval 1

}
