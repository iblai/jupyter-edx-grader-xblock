# Graded Jupyter Notebook Integration

## Overview

_Auto-grade a student assignment created as a Jupyter notebook, using the [nbgrader](http://nbgrader.readthedocs.io/en/stable/) Jupyter extension, and write the score in the Open edX gradebook_ 

> See also the [Jupyter Notebook Viewer XBlock](https://github.com/ibleducation/jupyter-viewer-xblock) to populate course content from publicly available Jupyter notebooks.

This XBlock uses Docker and nbgrader to create a Python environment and auto-grade a Jupyter Notebook, and tracks the resulting score as a problem within an EdX graded sub-section.
It allows an instructor to upload an assignment created with nbgrader, upload a `requirements.txt` file to configure the environment, set the maximum number of tries for the student, and set a deadline for the submission.
The student downloads the assignment file, answers the questions (executing all cells), and uploads the solution, which get immediately auto-graded. 
The student gets a visual score report, and the score gets added to his/her progress in the Open edX gradebook.

## Features and Support
- Integrated into EdX Grading System
- Maximum point values are pulled from the instructor version of the notebook
- A separate Python3 virtual environment is kept for each course
- Each student's notebook is run within its own Docker container
- [Several Other Configuration Options](#xblock-settings)
- Only supports auto-graded cells - **Does not support manually graded cells**.

## Important Notes
Efforts have been made to isolate the student notebook from the EdX Server by running them in an isolated docker container with no network access (by default). The only links back to the EdX server when the notebook is executed are the following mapped files/folders:
 - /var/www/nbgrader/courses/\<course-id\>/source/\<unit-id\>/\<nb-name\> (set to read-only)
 - /var/www/nbgrader/courses/\<course-id\>/submitted/\<username\>/\<unit-id\>/<nb-name\> (set to read-only)
 - /var/www/nbgrader/courses/\<course-id\>/autograded/\<username\>/\<unit-id\>/<nb-name\>
 - /var/www/nbgrader/courses/\<course-id\>/feedback/\<username\>/\<unit-id\>/\<nb-name\> 
 - /tmp/\<random-filename\> (temporary config file, set to read-only)

**PLEASE REMEMBER: Jupyter Notebooks do allow arbitrary code execution**

**Until more granular network controls are implemented it would be best to leave network access off unless you know and trust your student population**

## Installation and Setup
### XBlock
* login as the root user: `sudo -i`
* New Installation:
    * `/edx/bin/pip.edxapp install git+https://github.com/ibleducation/jupyter-edx-grader-xblock.git`
* Re-Installation:
    * `/edx/bin/pip.edxapp install --upgrade --no-deps --force-reinstall git+https://github.com/ibleducation/jupyter-edx-grader-xblock.git`

### Docker Installation
- Install Docker
    - [Docker CE Installation](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
- Create the `docker` group (this may already be done via installing docker)
    - `sudo groupadd docker`
- Create jupyter user
    - `adduser --no-create-home jupyter`
        - A strong password is recommended as any user within the `docker` group is considered a user with root privileges
    - `usermod -aG docker jupyter`
- Allow `www-data` user to switch to `jupyter` user and execute `docker` commands without a password
    - As a root user, enter: `visudo`
    - Enter the following line at the bottom of the file:
        - `www-data ALL=(jupyter) NOPASSWD:/usr/bin/docker`
    - This allows `www-data` user to execute `docker` commands as the `jupyter` user without using a password

### LMS/CMS Setup
In the following two files:
* `/edx/app/edxapp/edx-platform/lms/urls.py` 
* `/edx/app/edxapp/edx-platform/cms/urls.py` 

Add the following to the bottom of each file:
```python
# Jupyter Graded XBlock Endpoints
urlpatterns += (
    url(r'^api/jupyter_graded/', include('xblock_jupyter_graded.rest.urls')),
)
```

In the following two files:
* `/edx/app/edxapp/edx-platform/lms/envs/common.py` 
* `/edx/app/edxapp/edx-platform/cms/envs/common.py` 

Add the following at the bottom of the `INSTALLED_APPS` section:
```python
    # Jupyter Notebook Graded XBlock
    'xblock_jupyter_graded',
```

Restart `edxapp` via `/edx/bin/supervisorctl restart edxapp:`
Restart `edxapp_worker`'s via `/edx/bin/supervisorctl restart edxapp_worker:`

### Database Migration
Run the following as a root user to create the necessary database models:
```shell
sudo su edxapp -s /bin/bash
cd ~/edx-platform
source ../venvs/edxapp/bin/activate
./manage.py cms migrate xblock_jupyter_graded --settings=aws
```

### Nginx Setup
Currently, this XBlock makes some long running Ajax calls which can exceed the default nginx timeout of 60s on some machines. We will temporarily increase this timeout until the architecture can be updated to make this a background task that can be polled for state.

In the followingfile:
- `/etc/nginx/nginx.conf`

Add the following line in the `http` section under `# Basic Settings`:
- `proxy_read_timeout 300;`

Restart nginx via: `service nginx restart`

### Course Setup
Login to the EdX Studio and navigate to the course you would like to implement the XBlock in.

On the top Menu bar, select:
- Settings => Advanced Settings

In the `Advanced Module List`, add:
- "xblock\_jupyter\_graded"

Click `Save` at the bottom of the screen.

## XBlock Organization
### Studio Author View
On the main author screen in the Studio, there are three major instructor related sections:
- Python Virtual Environment
- Instructor Notebook Upload
- Uploaded Notebook Details

#### Python Virtual Environment
  - Allows you select a `requirements.txt` file to upload for this course.
    - You **only** need to add the packages that are imported from within the notebook.
    - You do **not** need to include nbgrader, jupyter, ipykernel, or any other jupyter related packages. 
    - `ipykernel` is included by default as it is required to install the student virtualenv as an execution environment
  - **This python environment will be shared by all Notebooks in this course**
  - A list of the currently installed packages is listed under **Installed Packages**
  - `requirements.txt` should be formatted like a normal `requirements.txt` file, but only supports the following formats:
    - `package_name==package_version` - Pinned version of a package
    - `package_name` - Unpinned version - installs newest version of the package
  - The docker container will be built each time a new environment is uploaded that doesn't already exist

#### Instructor Notebook Upload
- Allows you to select and upload a `source` notebook that will be used to generate the student version.

#### Uploaded Notebook Details
- Shows details about the currently uploaded notebook and allows the instructor to download the version currently attached.

### XBlock Settings
- _Display Name_: The title of this XBlock that the student will see
- _Student Instructions_: A set of instructions that will be shown to the student.
- _Allowed Submissions (Default: 0)_: The max number of submissions that a student may make. If set to `0`, infinite attempts are allowed.
- _Network Allowed (Default: False)_: If True, allows network access from within the container.
  - **CAUTION**: **Because Jupyter Notebooks allow arbitrary execution of code, it is best to leave this to `False` otherwise a student could make unauthorized web requests from the perspective of the server.**
  - More work should be done here to create more granular network isolation
- _Cell Timeout (Default: 15s)_: Max amount of time to allow a single cell to execute before raising a `KeyboardInterrupt` Exception to attempt to stop the cell
- _Allow Graded NB Download (Default: False)_: If `True`, allows student to download the autograded `.html` file generated by running `nbgrader feedback`. 
  - **Note**: This will show all `### HIDDEN TESTS` from the instructor notebook as they are included in the autograded feedback
- _Max File Size (B)_: Maximum file size student is allowed to upload (in bytes).
  - Default is 10kB > the size of the instructor version of the notebook if this field is unset when the instructor version is uploaded. Otherwise it can be modified to your liking, but it must be set.

## Instructor Workflow
- Add a `Graded Jupyter Notebook` from the `Advanced` button in the course Studio.
- Optionally upload a `requirements.txt` file containing all packages required for this course.
- Create the source jupyter notebook from a local installation of jupyter and nbgrader and save/export it to your system.
  - It is suggested to set the cell `ID` value for each `Autograder tests` cell to something meaningful to the student
  - For instance, if your notebook has two sections and two graded exercises per section:
    - Section 1
      - Exercise 1
      - Exercise 2
    - Section 2
      - Exercise 1
      - Exercise 2
  - naming the test cells something like:
    - `section1_excercise1` will help the student associate the point value with a test or section in the notebook when they see their results.
- Upload the source notebook in the `Instructor Notebook Upload` section.
  - The max score and notebook details will be reflected in the `Uploaded Notebook Details` section.
  - An error will be thrown if the text `BEGIN SOLUTION` is not found somewhere in the notebook as this likely indicates the appropriate version was not uploaded
- The student version will now be generated and can be downloaded in the Student Section
- Update the settings for this XBlock as appropriate
- Publish the Unit

## Student Workflow
- Select the `Download Student Notebook` link to download the student version
- Complete the notebook via a local installation of jupyter
- Upload the completed version of the student notebook
  - The notebook must be named the same as what is specified in the `Notebook Name:` label
- The notebook will be graded and results reflected in the `Results` section
  - If the student has no more attempts left, they will no longer be able to upload a new notebook
  - The most recent submission is the one recorded for grading

## Usage notes

### Watch the demo!

[![demo](https://github.com/ibleducation/jupyter-viewer-xblock/blob/master/demo-thumbnail.png)](http://www.youtube.com/watch?v=SwRAs8_FIdo)


## Copyright and License
(c) 2017 IBL Studios and Lorena A. Barba, [code is under BSD-3 clause](https://github.com/engineersCode/EngComp/blob/master/LICENSE). 

[![License](https://img.shields.io/badge/License-BSD%203--Clause-blue.svg)](https://opensource.org/licenses/BSD-3-Clause)
