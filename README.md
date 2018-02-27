# ProjectX-Ansible
Ansible scripts (AWX) for ProjectX

Debugging Tips
--------------

these scripts are designed to be used through AWX (Ansible Tower). This means that there can be a lengthy cycle of:

1. code
1. commit to repository
1. AWX to pull from repository
1. AWX run job
1. Understand errors and go back to 1.

This can all take quite a bit of time as well as cluttering up your git repository with lots of meaningless commits as you try to resolve errors in your Ansible scripts.


If you are in an extended debug cycle it can be easier to copy your scripts to **a representation** of the target machine and debug them there.

If you are using roles from a requirements file you will need to run the following in the roles directory:
```
ansible-galaxy install -r requirements.yml
```
This will download the roles you are using into the roles directory.

To run the playbook you are debugging:
```
ansible-playbook <playbook> --user <user> --ask-pass -i localhost,
```

This runs the playbook against local host (the trailing "," is quite important) as a particular user.

There are further options to enable the playbook to be run via sudo etc.

***Also*** remember the ***-v*** option. add to ***ansible-playbook*** command to get extra levels of debug indormation. From debug,***-v*** to fine detail, ***-vvvvv***.

jenkins-cicd.yml
----------------

This script installs Jenkins on a node that has already been stood up by Vagrant. This script is designed to be run from within AWX (Ansible Tower). Again, this script leans massivley on Guy Geerling's roles.

The script follows the same pattern as the AWX Ansible script to persist data. Effectively, the Vagrant script stans up the CICD node and then replaces the newly created Jenkins datastore (/var/lib/jenkins) with one from the shared direve (/vagrant/cicd/jenkins) if it exists. To prevent the DB from being restored constantly if the Ansible job is set up on a schedule within AWX there is a control file (/var/lib/jenkins/restored.txt). This file is created on first install of Jenkins, successive setups (vagrant up, when node has prevoously been destroyed) will copy across the archived database and make a note of the restore date time in this file. If the ansible job runs it only replaces the Jenkins data store if restored.txt does not exist.

When the CICD node is destroyed (vagrant destroy cicd) the curent jenkins DB is archived and the current DB is copied from /var/lib to the shared /vagrant/cicd (host backes) directory so that it will persist between node re-creations.

The script is currently configured to set up Jenkins for Python development. It adds Python development tools to the node and relevant Python quality checking plugins to the Jenkins installation.

Jenkins for Python
------------------

Python is an interpreted language, as such it does not need to be compiled in the same manner that Java or C# do. We do still need to run unit tests, code coverage, style checking and other static qualatitive an quantative metrics. Here we are setting up a very simple example that will run some unit tests and report on pass/fail and code coverage and then run a lint syntax checker and publish a report.  

For this very simple example we require the following Python tools to be installed onto the Jenkins build node:
 
* python
* python-nose
* python-coverage
* pylint
* python-mock

Jenkins will require the following plug-ins:
  
* Cobertura
* GitHub
* Warnings

***NOTE:*** Many blogs and examples on the web use the Violations plugin. This plugin is no longer maintained. you can follow other examples but just use Warnings instead of Violations.

Jenkins will need to be configured as follows:
1. From the Jenkins dashboard select ***New Item***. Enter a project name and select ***Freestyle Project***
2. General - Select GitHub project and add the project url
```
https://github.com/AgentCormac/ProjectX-Python/
```
3. source code Management - Select Git and add the project repository
```
https://github.com/AgentCormac/ProjectX-Python/
```
4. build Triggers - your choice. Add a GitHub trigger and/or a schedule to pull from Github or leave empty for on demand only.
5. Build - Execute shell - enter the following
```
PYTHONPATH=''
nosetests --with-xunit --all-modules --traverse-namespace --with-coverage --cover-package=projectx_python --cover-inclusive
python -m coverage xml --include=projectx_python*
pylint -f parseable -d I0011,R0801 projectx_python | tee pylint.out
```

   This runs the unit tests, saves the coverage report to ***coverage.xml*** and then runs lint and saves the output to ***pylint.out***.

7. Post-build actions - Add ***Scan for compiler warnings*** and select the Parser as pylint and enter the file pattern of pyliny.out. This will pick up the lint report and publish back to the Jenkinsw build job.
8. Post-build actions - Add ***Publish Cobertura coverage Report*** and enter coverage.xml into the cml report pattern field. This will publish the coverage report back to Jenkins.
9. Post-build actions - Add ***Publish JUnit test result report*** and enter nosetests.xml into the Test report XMLs field. This will publish the unit tests results report back to Jenkins.
  
Save and then select ***Build Now*** from the Build Project. Hopefully, the project will now build succesfully and display a blue ball agains the job in ***Build History***. this is beacuse there are a few lint issues that need cleaning up. Select the job number to see a bit more detail. it should show that there are 4 lint warnings. Drill down into the warnings to see what they are. you can clean these up by cloning the code base, modifying the code and pointing this job at your own repository. Re-run the job and the little blue ball should now be green.

AWX configuration
----------
The above script is designed to be run from AWX (Ansible Tower). The script persists Jenkis data between CID node's destruction and recreation by Vagrant by copying and restoring the Jenkins home directory to the host backed shared drive, following the same pattern as for AWX persistence.

Simply log into AWX (admin/admin) and creat a new inventory pointing at the CICD node (192.168.100.102). Create a project pointing at [ProjectX-Ansible](https://github.com/AgentCormac/ProjectX-Ansible), selecting ***Update on launch***. Add a new credential vagrant/vagrant. Add a new Job Template using the vagrant credential to run the jenkins-cicd.yml script from the CICD project against the CICD inventory.

Run the job to build Jenkins on the CICD node. You can add a schedule to it if you like, the persistence scripts are indempotent, just like Ansible.

Job done.