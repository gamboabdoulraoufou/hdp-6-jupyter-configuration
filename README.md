### hdp-6-jupyter-configuration

> Configuration
- 4 VMs on google compute Engine (3 VM for Hadoop Cluster and 1 VM for data backup and micro services)
- OS: CentOS 7
- RAM: 15 Go
- CPU: 4
- Boot disk: 200Go
- Attached disk: 2000G

> Cluster model

![MetaStore remote database](https://github.com/gamboabdoulraoufou/hdp-1-host-config/blob/master/img/archi_v2.png)


> 1- Install python and all dependencies
```sh
# install prerequisites
yum install bzip2

# Download Anaconda
wget https://3230d63b5fc54e62148e-c95ac804525aac4b6dba79b00b39d1d3.ssl.cf1.rackcdn.com/Anaconda2-4.0.0-Linux-x86_64.sh

# Install Anacondo (set installation folder to: /usr/local/anaconda2)
bash Anaconda2-4.0.0-Linux-x86_64.sh

# Add Anaconda path to paht
export PATH="/usr/local/anaconda2/bin:$PATH"

# close and reopen the terminal

  ```
  
> 2- Install Jupyter 
> 2-1- Install pip3 and other dependencies
```sh
# Install python3 dependencies
yum install -y gcc

# install python3
cd /usr/src
wget https://www.python.org/ftp/python/3.5.2/Python-3.5.2.tgz
tar xzf Python-3.5.2.tgz
cd Python-3.5.2
./configure
make altinstall
rm Python-3.5.2.tgz

# check python3 installation
python3.5 -V

# install pip3
wget https://bootstrap.pypa.io/get-pip.py
chmod 777 get-pip.py
/usr/local/bin/python3.5 get-pip.py

# install nodejs 
rpm -ivh http://dl.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm
yum install -y npm
yum install -y git
npm install -g configurable-http-proxy
```

> 2-2- Install JupyterHub and Jupyter for python 3 kernel
```sh
/usr/local/bin/python3.5  -m pip install jupyterhub
/usr/local/bin/python3.5  -m pip install "ipython[notebook]"
yum install -y python-setuptools
yum install -y build-essential
yum install -y python-devel.x86_64
yum install -y gcc gcc-c++ make openssl-devel
yum install -y python-pip
pip install --upgrade pip
pip install py4j
pip install "ipython[notebook]"
pip install kafka-python
/usr/local/anaconda2/bin/python -m pip install google-compute-engine
pip install boto
```

> 3- Configure Jupyter 
> 3-1- Create Jupyter configuration file
```sh
mkdir /usr/local/jupyter_conf
jupyterhub --generate-config -f /usr/local/jupyter_conf/jupyterhub_config.py

```

> 3-2- Create and add SSL certificat, so that your password is not sent unencrypted by your browser
```sh
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -nodes -days 365

```

> 3-3- Create cookie secret
```sh
#openssl rand -base64 2048 > /usr/local/jupyter_conf/cookie_secret
openssl rand -hex 32 > /usr/local/jupyter_conf/cookie_secret
sudo chmod a-srwx /usr/local/jupyter_conf/cookie_secret

```

> 3-4- Create auth token
```sh
openssl rand -hex 32 > /usr/local/jupyter_conf/proxi_auth_token

```

> 3-5- Create auth token
```sh
sudo touch /var/log/jupyterhub.log

```

> 3-6- Change Jupyter configuration
```sh
# Edit configuration file
vi /usr/local/jupyter_conf/jupyterhub_config.py

# Add the content below in configugation file
c = get_config()
# IP and Port
c.JupyterHub.ip = '192.168.0.92' # IP local
c.JupyterHub.port = 9084
# Security - SSL
c.JupyterHub.ssl_key = '/usr/local/jupyter_conf/key.pem'
c.JupyterHub.ssl_cert = '/usr/local/jupyter_conf/cert.pem'
# Security - cookie secret
c.JupyterHub.cookie_secret_file ='/usr/local/jupyter_conf/cookie_secret'
c.JupyterHub.db_url = '/usr/local/jupyter_conf/jupyterhub.sqlite'
# Security - http token
c.JupyterHub.proxy_auth_token = '/usr/local/jupyter_conf/proxi_auth_token'
# put the log file in /var/log
c.JupyterHub.extra_log_file = '/var/log/jupyterhub.log'
# specify users and admin
c.Authenticator.whitelist = {'test'}
c.Authenticator.admin_users = {'test'}

c.Spawner.cmd = '/usr/local/bin/jupyterhub-singleuser'

```

> 4- Setup Spark kernel
```sh
# Create folder for Spark Kernel
mkdir -p /usr/local/share/jupyter/kernels/pyspark/

# Create and add configuration information json file
cat <<EOF | sudo tee /usr/local/share/jupyter/kernels/pyspark/kernel.json
{
 "display_name": "PySpark",
 "language": "python",
 "argv": [
  "/usr/local/anaconda2/bin/python",
  "-m",
  "ipykernel"
  "-f",
  "{connection_file}"
 ],
 "env": {
  "SPARK_HOME": "/hadoop/usr/hdp/2.5.6.0-40/spark",
  "PYTHONPATH": "/hadoop/usr/hdp/2.5.6.0-40/spark/python/:/hadoop/usr/hdp/2.5.6.0-40/spark/python/lib/py4j-0.9-src.zip",
  "PYTHONSTARTUP": "/hadoop/usr/hdp/2.5.6.0-40/spark/python/pyspark/shell.py",
  "PYSPARK_SUBMIT_ARGS": "--master yarn-client pyspark-shell"
 }
}
EOF

```

> 5- Setup Python 2.7 kernel
```sh
# Create folder for Python 2 Kernel
mkdir -p /usr/local/share/jupyter/kernels/python2.7/

# Create and add configuration information json file
cat <<EOF | sudo tee /usr/local/share/jupyter/kernels/python2.7/kernel.json
{
"display_name": "Python 2", 
"language": "python", 
"argv": 
  ["/usr/local/anaconda2/bin/python", 
  "-c", 
  "from ipykernel.kernelapp import main; main()", 
  "-f", 
  "{connection_file}" ]
}
EOF

```


> 7- Run Jupyter
```sh
# Launch Jupyter server
/usr/local/bin/python3.5 -m jupyterhub -f /usr/local/jupyter_conf/jupyterhub_config.py

```

> Configure crontab to run jupyter after reboot
```sh
crontab -e

# Add the following line
#### START ####
# start jupyter
@reboot nohup sudo /usr/local/bin/python3.5 -m jupyterhub -f /usr/local/jupyter_conf/jupyterhub_config.py &
#### END ####
```

> Configure Maven
```sh
# go to installation folder
cd /usr/local

# download maven
wget http://www-eu.apache.org/dist/maven/maven-3/3.5.2/binaries/apache-maven-3.5.2-bin.tar.gz

# unzip file
tar xzf apache-maven-3.5.2-bin.tar.gz

# create sym link for maven
ln -s apache-maven-3.5.2  maven

# configure environment variables
vi /etc/profile.d/maven.sh

export MAVEN_HOME=/usr/local/maven
export PATH=${MAVEN_HOME}/bin:${PATH}

# load env variables
source /etc/profile.d/maven.sh

# check maven
mvn -version 
```

> Install and configure SBT

```sh 
# get SBT on yum repo 
curl https://bintray.com/sbt/rpm/rpm | sudo tee /etc/yum.repos.d/bintray-sbt-rpm.repo

# install SBT
sudo yum -y install sbt

```

> Install and configure SBT

```sh
cd ~
wget http://downloads.lightbend.com/scala/2.11.8/scala-2.11.8.rpm
sudo yum install scala-2.11.8.rpm
```

__Go to https://IP or your.host.com and enjoy!__

