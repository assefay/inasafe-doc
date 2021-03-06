.. _jenkins_windows_slave:

Set up of Windows 'slave' builds for Jenkins
============================================

Outline of procedure:
---------------------

Set up a virtual machine. In our testing regime we will be using:

* Windows 8 64bit
* Windows 8 32bit
* Windows 7 64bit
* Windows 7 32bit

We have purchased licenses of these OS's specifically for deployment in
our VirtualBox virtual machines.
The setup and installation of windows and VirtualBox is not described here
except as pertains to testing.
We will be installing the following components on each windows slave:

* **Microsoft .NET 3.5 (versions 4+ are not supported).** Under windows 8 you
  need to do this as an explicit install as the OS ships with .NET 4.5.
  .NET is needed to run the Jenkins service.
* **Python 2.7.** Although QGIS ships with a python interpreter, we need
  to install our own copy to support running unit tests and so on.
* **QGIS (1.8 at time of writing).** The unit tests depend on QGIS being present
  to run.
* **GitHub for Windows.** This is the convenient windows GIT client provided
  by GitHub - we will use it to clone and update our repositories.

With the base software installed, we will clone three GitHub repositories to
the VM:

* **InaSAFE** The application that we will be testing.
* **inasafe_data** Test data required for unit testing the  library.
* **inasafe-doc** Documentation that builds the webpage and the Docs.

We will configure the system to be able to run the unit tests, first
independently of Jenkins, and then as a Jenkins job.

Lastly, we will configure the VM Jenkins instance to be a slave for our
Jenkins master and fire off the |project_name| test suite in the VM whenever
a commit is made to the |project_name| master branch.

We will try to keep things as simple as possible.
The VM instances should not be used for any other purpose in order to keep
them small, fast and to avoid configuration issues caused by conflicting
libraries.

VM Configuration:
-----------------

We used the following configuration options when creating our virtual machines:

* 4 GB Ram (thus your host OS needs 8GB or more)
* 25 GB Virtual Hard Disk
* Installation of guest extensions.

All virtual machines were created using the free VirtualBox software.

Note that you may need to enable virtualisation in your PCs BIOS too.

All Virtual Machines were created with a *local* user account with

Username: inasafe
Password: XXXXXX (redacted)

We did not enable the new 'cloud login' options of Windows 8.

After the windows installation, we turned off all extraneous services to
maximise performance and increase test times.

Software Installation
----------------------

The following were the standard software installed on each Windows VM:

QGIS Install
............

* Install QGIS standalone from http://download.qgis.org
* Create the .qgis\python\plugins dir

GitHub for Windows Install
..........................

* Install from http://windows.github.com
* Do not log in with your account details (security!!!)
* In options, set your default storage directory to
  :file:`C:\\Users\\inasafe\\.qgis2\\python\\plugins`

Copy over the repositories from the host system (quickest) or check them out
from github anonymously by using the Git Shell (It seems that the GUI does not
support cloning projects anonymously).

Do that by opening the Git Shell and assigning the command::

  git clone https://github.com/AIFDR/inasafe.git

We also need the test data available in this freshly cloned directory.
So clone the test data inside this inasafe clone.

To do this enter the commands::

  git clone https://github.com/AIFDR/inasafe_data.git inasafe_data

If you take the former route of just copying them over, after copying them in
to the plugins directory use the GitHub Windows apps options dialog to find
them by clicking the 'scan for repositories' button.

Python
......

* Follow the process described in :ref:`windows_shell_launcher-label` so that
  you can use the QGIS libraries from a python shell.
  Note that you probably need to change the second last line of that script to
  :samp:`cd "%HOMEPATH%\\.qgis\\python\\plugins\\inasafe"` (removing the
  '-dev') at the end.
* Follow the processed described in :ref:`windows-nose-setup` so that you have
  a working pip, nose etc.

Now run the tests and ensure that they can be run from the command line
*before* attempting to run them via Jenkins.
Again, this is just to make sure that everything is setup nicely to avoid any
problems on Jenkins.
::

    C:\Users\inasafe\.qgis\python\plugins\inasafe>runtests.bat


.NET 3.5 Install
................

To install Jenkins, you first need to ensure you have .NET 3.5 on your system.
With Windows 7 you are safe. You already have .NET 3.5 installed.

Windows 8 ships with .NET 4+ so you need to manually install the older version
too.
First visit http://www.microsoft.com/en-us/download/details.aspx?id=21 and
either choose the .NET Framework Full Package or get the online installer.
Note that the full package link is near the bottom of the page.

Run the installer and accept all the defaults to install the .NET 3.5
framework.

Jenkins Install
...............

Simply go to http://jenkins-ci.org/ and download the windows native package
and then install it.
If it is a dedicated machine only for Jenkins it is a very good idea to not
install it to C://Program Files// or C://Program Files (x86) though the
hassle in the file paths but rather install it to C://Jenkins.

Once Jenkins is set up, open your browser at http://localhost:8080 to access
the Jenkins page.

Jenkins Configuration
---------------------

Plugins
.......

The first thing you need to do is install some jenkins plugins. To do this
do :menuselection:`Manage Jenkins --> Manage Plugins --> Available tab`.

Now install at least these plugins:

* Jenkins GIT plugin
* Git Plugin
* GitHub API plugin
* GitHub plugin

In addition these plugins should be available by default:

* Jenkins mailer plugin
* External Monitor Job Type Plugin
* pam-auth

For simplicity, I also disabled the following plugins:

* LDAP Plugin
* ant
* javadoc
* Jenkins CVS Plug-in
* Maven Integration plugin
* Jenkins SSH Slaves plugins
* Jenkins Subversion plugin
* Jenkins Translation Assistance plugin

System configuration
....................

We need to provide the path to git so that Jenkins can automatically make
checkouts of each version.

:menuselection:`Jenkins --> Manage Jenkins --> Configuration --> Git
Installations --> Path to Git executable` needs to be set.
On my system I used the following path::

    C:\Users\inasafe\AppData\Local\GitHub\PortableGit_XXXXXXXXXXXX\bin\git.exe

The GitHub application's git installer is a portable app and the path for you
is going to look a little different - just look in in your AppData dir and you
should find it.

.. note:: The Jenkins system user will need to have read permissions on the
    above directory. And don't forget to give the user that Jenkins is run as
    permissions to the jobs Directory within the Jenkins installation.

Job Configuration
.................

Next we create our build job with the following options:

* :menuselection:`New Job`
* :menuselection:`Job name` : :kbd:`inasafe-win7-32` (adjust the name as
  appropriate)
* :menuselection:`Build a free-style software project` : select

On the job configuration page use the following options:

* :menuselection:`Description` : :kbd:`Windows 8 64 bit build of InaSAFE`
* :menuselection:`GitHub project` : :kbd:`https://github.com/AIFDR/inasafe/`
* :menuselection:`Source Code Management` section
* :menuselection:`Git` : Check
* :menuselection:`Repository URL` : :kbd:`git://github.com/AIFDR/inasafe.git`
* :menuselection:`Branches to build` : :kbd:`version-1_1`
* :menuselection:`Repository browser` : :kbd:`githubweb`
* :menuselection:`Url` : :kbd:`http://github.com/AIFDR/inasafe/`

* :menuselection:`Build triggers` section
* :menuselection:`Poll SCM` : check and set to :kbd:`H * * * *` for
  minutely checks.

It might be possible that the initial checkout has to be done manually by
using the command ::

  git clone git://github.com/AIFDR/inasafe.git workspace
  within the specific job directory.

Save your changes at this point and make a commit, you should see the job
produce output something like this the next time a commit takes place::

    Started by timer
    Building in workspace C:\Jenkins\jobs\inasafe-win7-32\workspace
    Checkout:workspace / C:\Jenkins\jobs\inasafe-win7-32\workspace - hudson.remoting.LocalChannel@1fd5730
    Using strategy: Default
    Last Built Revision: Revision 5403e3ba45129b42edaa2bc0ebd12e8c9ead868e (origin/master)
    Fetching changes from 1 remote Git repository
    Fetching upstream changes from git://github.com/AIFDR/inasafe.git
    Commencing build of Revision 5403e3ba45129b42edaa2bc0ebd12e8c9ead868e (origin/master)
    Checking out Revision 5403e3ba45129b42edaa2bc0ebd12e8c9ead868e (origin/master)
    Finished: SUCCESS

That validates that at least your git checkout is working as expected.
