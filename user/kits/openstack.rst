.. _openstack-blueprint:

A la OpenStack
==============

Summary
-------
If you want to develop like the Openstack project, the "a la Openstack" :term:`blueprint` is what you are after.

Learn more on the processes that are used by the Openstack project here:

* `How to contribute to Openstack <https://wiki.openstack.org/wiki/How_To_Contribute>`_
* `Openstack infrastructure team <https://wiki.openstack.org/wiki/InfraTeam>`_

Tools and features
------------------
In its current version, the Openstack score contains the following features:

* Puppet master automation, create new modules and promote them in the :term:`forj-config` project.
* Gerrit/git services for revision control and change management.
	* Login with google openid, use www.google.com to setup your email account for authentication.
	* Create open commits that can be reviewed with the team and gated for change before they hit master.
	* Tag and release changes when they are ready for testing or production.
	* Mark commits as work in progress or abandon revert old changes.
	* Create new gerrit projects and acl’s managed from source in forj-config project.
* Jenkins/Zuul integration
	* Create zuul macros to automate common build task, or re-use pre-existing openstack macros included in the pre-configured forj-config project.
	* Create automatic Jenkins jobs so that your team no longer manages Jenkins setup on Jenkins master, all configuration is managed as source in forj-config git project.
	* Gate changes and execute automated test as soon as changes make it to gerrit/git.
	* Promote changes based on gerrit events for tagging or comments made during reviews.
	* Login with google openid
	* Zuul status: see what changes are being automatically tested, the progress and the status from one interface
* Pastebin integration
	* Collaborate with team mates with code snippets and log pasting for troubleshooting defects.
* Defect tracking
	* Mark gerrit changes with defect id’s to link to your defect system with Launchpad or HP Agile Manager

Project Management
------------------

You can manage project using Maestro user interface, or by editing the "forj-config" GIT repository (through Gerrit). 

Adding a new project
********************

In the "A la OpenStack" blueprint, projects are managed by code, exactly like the OpenStack infrastructure project. This code, which sits in the forj-config Gerrit repository describes a project and is used to provision the entire chain associated to a project (Gerrit, Jenkins, Zuul).

With Maestro User Interface
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
	* Enter the maestro UI and sign-in if needed, so you become the administrator and can add projects. The first user who logs becomes administrator and can add other administrators. Only administrator can create projects.
	* Connect to Gerrit. The first user who logs in to Gerrit becomes administrator
	* As a Gerrit administrator, add yourself to the "forj-config" group. This allows you to review and approve changes in the forj-config project
	* Go to the projects tab then add projects icon, finally set the name for you project and wait ~15 minutes until the change is propagated to gerrit (this uses Puppet mechanisms)
	* Go to Gerrit and approve (+2) or reject (-2) the project creation change
	* Once Puppet has done its work (up to 30 minutes), the new project is fully created and configured in Gerrit, Jenkins and Zuul

.. note::
	You can force "Puppet Apply" and speed up the process through the Jenkins "puppet-apply-all-nodes" job, or directly in a terminal session on the Maestro / Puppet master system


Using "A la Openstack" mechanisms
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You have full control of project creation by checking out the forj-config project and modifying the files that you need.

.. note::
	In the "A la OpenStack" blueprint, everything is threated as code. You must go through Gerrit mechanisms to edit project configuration. Otherwise, your changes will be dismissed the next time the puppet master runs.

* Check out the forj-config project
	Each forge has its own forj-config project. Make sure your ssh keys are added to the gerrit server. To approve a change, make sure your user account is a member of the forj-core group.

* Assigning someone to be a verifier for forj-config project
	When there is no jenkins/zuul functionality on the forj-config project, it's sometimes necessary to verify changes manually. This can be performed by updating the group membership for Continuous Integration Tools group in gerrit.

* add an acl file (to the acls/production directory below) by using cookiecutter.config as an example found in the below path (i.e. make a copy)::

	[forj-config.git] / modules / runtime_project / files / gerrit / acls / templates /  cookiecutter.config

* place cookiecutter.config in directory (projectname should match the name of the project): (note: change the 'cookiecutter-core' to whatever group name you would like to use in your project: example: my-new-proj1-core)::

	[forj-config.git] / modules / runtime_project / files / gerrit / acls / production / projectname.config

* to create project edit file::

	[forj-config.git] / modules / runtime_project / templates / gerrit / config / production / review.projects.yaml.erb

* push changes to your gerrit repo:

	.. sourcecode:: console

		$ git add <new-project-acl-file>
		$ git add review.projects.yaml.erb
		$ git commit -m "my new project"
		$ git push 

* To learn more on how to configure yaml, see `jeepb <http://ci.openstack.org/jeepyb.html>`_ docs.
* you can migrate public projects with the upstream option.
* Projects that are created in gerrit currently have no approach for deletion, but these can be removed from normal users view through acl changes. For more info, please refer to : `rename project <http://ci.openstack.org/gerrit.html#renaming-a-project>`_ or remove project


Adding a new jenkins job and configure Zuul for a given project in gerrit
*************************************************************************
Zuul configuration consist of 4 basic parts.

1. update **hieradata** to include any new templates that will be used for the job in **runtime_project/files/hiera/hieradata/Debian/nodetype/ci.yaml**

	* this is only needed if you need a new compiler option, or new tool that will not exist on the build server.

	* configure in the following section ci-node -> class cdk_project::jenkins -> job_builder_configs. 

	Example:

		.. sourcecode:: yaml

			cdk_project::jenkins::job_builder_configs:
				- 'tutorials.yaml'
				- '<new_job_template_name>.yaml'

2. configure the new template into **runtime_project/templates/jenkins_job_builder/config/**

	* a pre-existing template file can be used to describe the builders for the job, or a new one can be created

	* pre-existing macros can be found in runtime_project/files/jenkins_job_builder/config/macros.yaml

3. update layout.yaml in **runtime_project/files/zuul/config/production/layout.yaml**

	* the projects section should be updated with the new project and gates, along with jobs that will be executed from projects.yaml, example:

	.. sourcecode:: yaml

		projects:
		 - name: tutorials
		   check:
		     - tutorials-flake8
		   gate:
		     - tutorials-flake8
		   post:
		     - puppet-apply-all-nodes
		   release:
		     - tutorials-flake8


4. add the project section to **runtime_project/files/jenkins_job_builder/config/projects.yaml**

	* this will define the jobs to be created in jenkins, job names will be mapped to buiders by zuul. The "name" must match the job-template layout file (line 2 in the jenkins_job_builder file), and the "git_project" must match with the name of your project in gerrit.

	.. sourcecode:: yaml

		projects:
		   name: tutorials
		   git_project: tutorials
		   branch: master
		   jobs:
		    - '{name}-flake8'
		    - '{name}-<new_job_name>'

Once this is done, you will need to push the changes to gerrit, verify and submit. Next the eroplus box will need to run puppet cycle, or puppet agent -t to get the new runtime_project udpates. Finally the ci server will need to run a puppet cycle or puppet agent -t so that the job builder can setup the job.

.. Note:: More info on zuul: `http://ci.openstack.org/zuul <http://ci.openstack.org/zuul>`_


Remove a project in gerrit
**************************

* Stop gerrit:

	.. sourcecode:: console

		$ sudo service gerrit stop

* start the gsql client on local admin bash shell:

	.. sourcecode:: console

		$ java -jar /home/gerrit2/review_site/bin/gerrit.war gsql -d /home/gerrit2/review_site

* remove entries from table account_project_watches

	.. sourcecode:: sql

		select * from account_project_watches;
		delete from account_project_watches where project_name = 'tutorials-2'
		delete changes
		select * from changes where dest_project_name = 'tutorials-2';
		delete from changes where dest_project_name = 'tutorials-2';

* Remove the repo from disk.

	.. sourcecode:: console

		$ rm -rf /var/lib/git/tutorials-2.git
		$ rm -rf /home/gerrit2/review_site/git/tutorials-2.git/

.. Note:: this should be done on all replicas

* Start gerrit back up

	.. sourcecode:: console

		$ service gerrit start


User management
---------------

In the "A la OpenStack" blueprint, the first user who authenticates to Gerrit and Jenkins become administrator. Then, it is the role of the administrator to add users in the respective tools and projects.

.. _openstack-blueprint-faq:

FAQ
---
... How do I create a new project?

   Creating a new project on a "à la OpenStack" forge means creating a new Gerrit repository.
   We use the CI workflow of the forge itself to manage the project creation process.
   Configuration files are modified and updated to provide the administrator of 
   the forge an oportunity to review the commit. Currently we do not provide 
   automatic review option, but one could be setup using zuul gate triggers.

... re-trigger the verification for project create change request?

   If your forge did not trigger a verification check for the project creation 
   request, it is possible to re-trigger the request on the change request.
   Go to the change request and add a new comment.  Make the comment text say:
   'recheck no bug'. This should trigger a zuul gate check for the change request.

... approve a new project creation request on gerrit?

    First you must be the administrator of your forge or contact and the administrator
    of the forge you will try to access.  The approving user must be added to the 
    group, forj-core.  This can be done in Gerrit from the Admin->Groups menu by the 
    Gerrit administrator.  Once done, the user added can then administer
    approvals by adding +2 for Code Review and +1 for Approved on the change request.

... change the group that approves changes for forj-config on gerrit?

    Approval permissions for groups is managed by the forj-config acl's file.
    This can be updated with a change request update to the forj-config source on the
    file : 
    [forj-config.git]/modules/runtime_project/files/gerrit/acls/production/forj-config.config
    
    Change the group forj-core to a new group name.  If the group does not exist
    a new one will be created.
