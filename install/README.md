# Installing Klara

## Requirements for running Klara:

- GNU/Linux (we recommend `Ubuntu 16.04` or latest LTS)
- SQL DB Server: MySQL / MariaDB
- Python 2.7
- Python virtualenv package
- Yara (installed on workers)

Installing Klara consists of 4 parts:

- Database installation
- Worker installation
- Dispatcher installation
- Web interface installation

Components are connected between themselves as follows:

```
                              +----------+          +----------------+
                              |          |          |                |
                  +---------->+ Database +<--+      |     nginx      |
                  |           |          |   |      |   (optional)   |
                  |           +----------+   |      |                |   
           +------+------+                   |      +-------+--------+   
           |             |                   |              |             
    +----->|  Dispatcher | <---+             |              |            
    |      |             |     |             |              |            
    |      +------+------+     |             |              v            
    |             |            |             |      +-------+--------+
    |             |            |             |      |                |
    |             |            |             |      |                |
+---+----+   +----+---+   +----+---+         ^------+   Web server   |
|        |   |        |   |        |                |                |
| Worker |   | Worker |   | Worker |                |                |
|        |   |        |   |        |                +----------------+
+--------+   +--------+   +--------+


```
Workers connect to Dispatcher using a simple `HTTP REST API`. Dispatcher and the Web server 
connect the MySQL / MariaDB Database using TCP connections. Because of this, components can be installed on 
separated machines / VMs. The only requirements is that TCP connections are allowed between them.

# Installing on Windows

Since entire project is written in Python, Dispatcher and Workers can be set up to run in an Windows environment. Unfortunately, as we only support Ubuntu, instructions will be provided for this platforms, but other GNU/Linux flavors should be able to easily install Klara as well.

## Database installation

Please install a SQL database (we recommend MariaDB) and make it accessible for Dispatcher and Web Interface.

To create a new DB user, allowing access to `klara` database, for any hosts, identified by password `pass12345` use the following command:

```
##### For `klara` DB #####
# Please use random/secure password for user 'klara' on DB 'klara'
CREATE DATABASE klara;

```
~~`CREATE USER 'klara'@'12.0.1' IDENTIFIED BY 'pass12345';`~~  
~~`GRANT USAGE ON *.* TO 'klara'@'12.0.1' IDENTIFIED BY 'pass1345' WITH MAX_QUERIES_PER_HOUR 0 MAX_CONNECTIONS_PER_HOUR 0 MAX_UPDATES_PER_HOUR 0 MAX_USER_CONNECTIONS 0;`~~  
```
CREATE USER 'klara'@'%' IDENTIFIED BY 'pass12345' REQUIRE NONE WITH MAX_QUERIES_PER_HOUR 0 MAX_CONNECTIONS_PER_HOUR 0 MAX_UPDATES_PER_HOUR 0 MAX_USER_CONNECTIONS 0;
CREATE USER 'klara'@'12.0.1' IDENTIFIED BY 'pass12345' REQUIRE NONE WITH MAX_QUERIES_PER_HOUR 0 MAX_CONNECTIONS_PER_HOUR 0 MAX_UPDATES_PER_HOUR 0 MAX_USER_CONNECTIONS 0;
CREATE USER 'klara'@'localhost' IDENTIFIED BY 'pass12345' REQUIRE NONE WITH MAX_QUERIES_PER_HOUR 0 MAX_CONNECTIONS_PER_HOUR 0 MAX_UPDATES_PER_HOUR 0 MAX_USER_CONNECTIONS 0;
GRANT ALL PRIVILEGES ON `klara`.* TO 'klara'@'12.0.1';
GRANT ALL PRIVILEGES ON `klara`.* TO 'klara'@'localhost';
GRANT ALL PRIVILEGES ON `klara`.* TO 'klara'@'%';
```

Once Dispatcher and Web Interfaces are set-up and configured to point to DB, the SQL DB needs to be created. Please run the SQL statements from [db_patches/db_schema.sql](db_patches/db_schema.sql) location:

```
mysql -u klara -p klara < db_schema.sql
```

## Dispatcher installation

Install the packages needed to run Dispatcher:
```
sudo apt -y install libmysqlclient-dev python-dev-is-python2 python2.7-dev default-libmysqlclient-dev build-essential
python2 -m pip install virtualenv
```

We recommend running dispatcher on a non-privileged user. Create an user which will be responsible to run Worker as well as Dispatcher:

```
sudo groupadd -g 500 projects
sudo useradd -m -u 500 -g projects projects
```

Create a folder needed to store all Klara project files:

```
sudo mkdir /var/projects/
sudo chown projects:projects /var/projects/
```

Substitute to projects user and create the virtual env + folders needed to run the Dispatcher:

**Note**: From now on, all commands will be executed under `projects` user

```
su projects
mkdir /var/projects/klara/ -p
mkdir /var/projects/klara/logs/
# Create the virtual-env
virtualenv ~/klara/env
```

Clone the repository:

```
git clone https://github.com/KasperskyLab/klara.git ~/klara-github-repo
```

Copy Dispatcher's files and install python dependencies:
```
cp -R ~/klara-github-repo/dispatcher /var/projects/klara/dispatcher/
cd /var/projects/klara/dispatcher/
cp config-sample.py config.py
. ~/env/bin/activate
sudo wget https://raw.githubusercontent.com/paulfitz/mysql-connector-c/master/include/my_config.h -O /usr/include/mysql/my_config.h
pip install -r ~/klara-github-repo/install/requirements.txt
```
* `sudo wget...` line command above was a fix to a problem where config.h is missing. THe solution is found [here](https://askubuntu.com/questions/1324362/pip-install-mysql-python-not-working-on-ubuntu-20-04)

Now fill in the settings in config.py:

```
# Main settings for the master
# Set debug lvl
logging_level  = "debug"

# What port should the server listen to
listen_port = 8888

# Notification settings
notification_email_enabled  = True
notification_email_from     = "klara-notify@example.com"
notification_email_smtp_srv = "12.0.1"

# MySQL / MariaDB settings for the Dispatcher to connect to the DB
mysql_host      = "12.0.1"
mysql_database  = "klara"
mysql_user      = "klara"
mysql_password  = "{pass12345}"
```
Once settings are set, you can check Dispatcher is working by running the following commands:
```
sudo su projects
# We want to enable the virtualenv
. ~/env/bin/activate
cd /var/projects/klara/dispatcher/
chmod u+x ./klara-dispatcher
./klara-dispatcher
```
If everything went well, you should see:
```
[01/01/1977 13:37:00 AM][INFO]  ###### Starting KLara Job Dispatcher ######
```

In order to start Dispatcher automatically at boot, please check [Supervisor installation](supervisor.md)

Next step would be starting Dispatcher using `supervisorctl`. Before we do this step, make sure we have supervisor installed first; if not, `sudo apt install -y supervisor`

Do the steps as detailed in the above link before continuing this!

```
sudo supervisorctl update
sudo supervisorctl start klara_dispatcher
```


# Worker installation

## Setting up an API key to be used by Workers

Each worker should have its own unique assigned API key. This helps maintaining strict access controls.

In order to insert a new API key to be used by a KLara worker, a new row needs to be inserted into DB table `agents` with the following entries:

* `description` - Short description for the worker (up to 63 chars)
* `auth` - API auth code (up to 63 chars)

```
mysql > use klara;
mysql > INSERT INTO agents value ({id},"{worker description}", "{randomize the code; up to 63 chars}");
```

## Installing the Worker agent

Install the packages needed to run Worker:
```
sudo apt -y install python-virtualenv libmysqlclient-dev python-dev git
```

We recommend running Worker using a non-privileged user. Create an user which will be responsible to run Worker as well as Dispatcher:

```
sudo groupadd -g 500 projects
sudo useradd -m -u 500 -g projects projects
```

Create a folder needed to store all Klara project files:

```
sudo mkdir /var/projects/
sudo chown projects:projects /var/projects/
```

Substitute to projects user and create the virtual env + folders needed to run the Worker:

**Note**: From now on, all commands will be executed under `projects` user

```
su projects
mkdir /var/projects/klara/ -p
mkdir /var/projects/klara/logs/
# Create the virtual-env
python2 -m virtualenv ~/klara/env
```

Clone the repository:

```
git clone https://github.com/KasperskyLab/klara.git ~/klara-github-repo
```

Copy Worker's files to the newly created folder and install python dependencies:
```
cp -R ~/klara-github-repo/worker /var/projects/klara/worker/
cd /var/projects/klara/worker/
cp config-sample.py config.py
. ~/env/bin/activate
pip install -r ~/klara-github-repo/install/requirements.txt
```

Now fill in the settings in config.py:

**Note**: use the API key you just inserted in table `agents` above;  
**Note**: Check [Worker Settings](#setting-up-workers-scan-repositories-and-virus-collection) to understand how to change settings
`virus_collection` and `virus_collection_control_file`

```
# Setup the loglevel
logging_level  = "debug"

# Api location for Dispatcher. No trailing slash !!
# Dispatcher is exposing the API at "/api/" location
api_location = "http://12.0.1:8888/api"
# The API key set up in the `agents` SQL table
api_key      = "test"

# Specify worker refresh time in seconds
refresh_new_jobs    = 60

# Yara settings
# Set-up path for Yara binary
yara_path           = "/opt/yara-latest/bin/yara"
# Use 8 threads to scan and scan dirs recursively
yara_extra_args     = "-p 8 -r"
# Where to store Yara temp results file
yara_temp_dir       = "/tmp/"

# md5sum settings
# binary location
md5sum_path         = "/usr/bin/md5sum"

# tail settings
# We only want the first 1k results
head_path_and_args  = ["/usr/bin/head", "-1000"]

# Virus collection should NOT have a trailing slash !!
virus_collection                = "/var/projects/klara/repository"
virus_collection_control_file   = "repository_control.txt"
```
Once the settings are set, you can check Worker is working by running the following commands:
```
sudo su projects
# We want to enable the virtualenv
. ~/env/bin/activate
cd /var/projects/klara/worker/
chmod u+x ./klara-worker
./klara-worker
```

If everything went well, you should see:
```
[01/01/1977 13:37:00 AM][INFO]  ###### Starting KLara Worker ######
```

In order to start Worker automatically at boot, please check [Supervisor installation](supervisor.md)

Next step would be starting Worker using `supervisorctl`:
```
sudo supervisorctl update
sudo supervisorctl start klara_worker 
```
## Installing Yara on worker machines

Install the required dependencies:
```
sudo apt -y install libtool automake libjansson-dev libmagic-dev libssl-dev build-essential

#
# Get the latest stable version of yara from https://github.com/virustotal/yara/releases
# Usually it's good practice to check the hash of the archive you download, but here we can't, since it's from GH
#

wget https://github.com/VirusTotal/yara/archive/vx.x.x.tar.gz
cd yara-3.x.x
./bootstrap.sh

./configure --prefix=/opt/yara-x.x.x --enable-address-sanitizer --enable-dotnet --enable-macho --enable-dex
make -j4
sudo make install
```

Now you should have Yara version installed on `/opt/yara-x.x.x/`

Create a symlink to the latest version, so when we update it, workers don't have to be reconfigured / restarted:
```
# Symlink to the actual folder
cd /opt/
ln -s yara-3.x.x/ yara-latest
```

## Setting up worker's scan repositories and virus collection

Each time workers contact Dispatcher in order to check for new jobs, will verify first if they can execute them. Klara was designed such as:
* each worker agent has a (root) virus collection where all the scan repositories should exist (setting `virus_collection` from `config.py`)
* multiple `scan repositories` will be checked by KLara workers when trying to accept a job. (for example, if one user wants to scan `/clean` repository, a Worker agent will try to check if it's capable of scanning it, by checking its `virus_collection` folder )
* in order to check if it's capable of scanning a particular `scan repository`, Worker checks if the collection control file exists (setting `virus_collection_control_file` from `config.py`) at location: `virus_collection` + `scan repository` + / + `virus_collection_control_file`.

Basically, if a new job to scan `/mach-o_collection` is to be picked up by a free Worker with the following `config.py` settings:

```
virus_collection                = "/mnt/nas/klara/repository"
virus_collection_control_file   = "repo_ctrl.txt"
``` 
then that Worker will check if it has the following file and folders structures:
```
/mnt/nas/klara/repository/mach-o_collection/repo_ctrl.txt
```

If this file exists at this particular path, then the Worker will accept the job and start the Yara scan with the specified job rules, searching files in `/var/projects/klara/repository/mach-o_collection/`

It is entirely up to you how to organize your scan repositories. An example of organizing directory `/mnt/nas/klara/repository` is as follows:

* `/clean`
* `/mz`
* `/elf`
* `/mach-o`
* `/vt`
* `/unknown`

## Filesystem optimisation

Running Klara (or Yara) on a fast enough machine is very important for stability and getting back results fast enough. Pleas check some tips and tricks for [filesystem optimisations](features_fs_optimisations.md)

## Repository control

KLara Workers check only if the repository control file exists in order to prepare the Yara scan. Contents of the file should only be an empty JSON string:

```
{}
```

Optionally, just for usability, you should write some info about the repository:

```
{"owner": "John Doe", "files_type": "elf", "repository_type": "APT"}
```

Scan Repository control file also has some interesting modifiers that can be used to manipulate Yara scans or results. For further info, please check [Advanced usage](features_advanced.md)

# Web interface installation

Requirements for installing web interface are:

- web server running at least PHP 5.6
- the following php7 extensions:

> Before following the below steps, make sure to install the following libraries first!:

```
sudo apt install -y build-essential
sudo apt install php php-pear php-dev libmcrypt-dev
```

> Confirm that we have the following binaries installed by performing the following commands:

```
gcc --version
make --version
which pecl 
```


```
apt install php-fpm php php-mysqli php-curl php-gd php-intl php-pear php-imagick php-imap php-memcache php-pspell php-sqlite3 php-tidy php-xmlrpc php-xsl php-mbstring php-apcu
```

> the following libraries need to be installed using `pecl`: `php-mcrypt`, `php-recode`, `php-gettext`  
> Update pecl channels:

```
sudo pecl channel-update pecl.php.net
sudo pecl update-channels
```

> Search for the missing libraries mentioned previous

```
sudo pecl search mcrypt
```

> Install these libraries

```
sudo pecl install mcrypt
```

Once you have this installed, copy `/web/` folder to the HTTP server document root. Update and rename the following sample files:

- `application/config/config.sample.php` -> `application/config/config.php`
- `application/config/database.sample.php` -> `application/config/database.php`
- `application/config/project_settings.sample.php` -> `application/config/project_settings.php`

You must configure the `base_url`, `encryption_key` from `config.php` as well as respective settings in `database.php` & `project_settings.php` .
More info about this here:

- https://www.codeigniter.com/userguide3/installation/upgrade_303.html
- https://codeigniter.com/userguide3/libraries/encryption.html
- https://www.codeigniter.com/userguide3/database/configuration.html
- https://www.nginx.com/resources/wiki/start/topics/recipes/codeigniter/ (The correct nginx configuration for codeigniter)

For your convenience, 2 `users`, 2 `groups` and 2 `scan repositories` have been created:

* Users (`users` DB table):

| Username      | Password                | Auth level     | Group ID     | Quota |
| ------------- |:-------------:          | :----------    | ---------    | :---- |
| admin         | `super_s3cure_password` | `16` (Admin)   | `2` (admins) | N/A (Admins don't have quota) |
| john          | `super_s3cure_password` | `4` (Observer) | `1` (main)   | 1000 scans / month |

* Groups (`users_groups` DB table):

| Group name    | `scan_filesets_list` (scan repositories) | Jail status |
| ------------- | :-------------                           | ----------- |
| main          | `[1,2]`                                  | 0 (OFF - Users are not jailed) |
| admins        | `[1,2]`                                  | 0 (OFF - Users are not jailed) |

* Scan Repositories (`scan_filesets` DB table):

| Scan Repository   |
| -------------     |
| /virus_repository |
| /_clean           |


For more info about Web features (creating / deleting users, user quotas, groups, auth levels, etc..), please check dedicated page [Web Features](features_web.md)

--------

That's it! If you have any issues with installing this software, please submit a bug report, or join our [Telegram channel #KLara](https://t.me/kl_klara)

Happy hunting!


