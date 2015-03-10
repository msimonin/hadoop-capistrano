Hadoop-capistrano
=================

Hadoop deployment for Grid'5000.

* hadoop 1.2.1 -> branch ```hadoop 1```
* hadoop 2.6.0 -> branch ```hadoop 2``` and ```master```


## Deployment Model


* One ```master``` node with
  * ResourceManager
  * NameNode
  * SecondaryNameNode
  * JobHistoryServer

* Some ```slave``` nodes with  
  * Namenode * (see below)
  * Nodemanager * (see below)

> Note on namenode and nodemanager colocation

The ```xp.conf``` files allows you to specify to not colocate ```namenode``` and ```datanode```. If colocation is set to ```false``` you'll get two separated clusters of nodes running either ```namenode``` or ```datanode``` processes.

### Deploy !

* Create an ```xp.conf``` file from the ```xp.conf.sample```.
* Install the required dependency ```bundle install``` (the use [```rvm```](htttp://rvm.io) is encouraged)
* Configure [```restfully```](http://github.com/crohr/restfully)
* Deploy

```
cap automatic
```

### Verify the installation

```
BENCH="pi 4 100" cap benchmark
```

Note : this will call
```yarn jar hadoop-mapreduce-examples*.jar #{BENCH}```

### Gui

Gui are available on the default port (see the hadoop documentation)

## Available commands

```
─$ cap -T
cap automatic             # Automatic deployment
cap benchmark             # Launch a benchmark, BENC va...
cap clean                 # Remove all running jobs
cap cluster:format_hdfs   # Format the cluster
cap cluster:start         # Start the cluster
cap cluster:status        # Status of the cluster
cap cluster:stop          # Stop the cluster
cap configure             # configure nodes
cap configure:core_site   # configure core-site.xml
cap configure:mapred_site # configure mapred-site.xml
cap configure:permissions # Give msimonin permission to...
cap configure:topology    # create the topology of the ...
cap configure:yarn_site   # configure yarn-site.xml
cap deploy                # Deploy with Kadeploy
cap invoke                # Invoke a single command on ...
cap prepare               # Prepare nodes
cap prepare:install       # Download tarball
cap prepare:packages      # Install extra pacakges
cap shell                 # Begin an interactive Capist...
cap submit                # Submit jobs
cap uninstall             # Remove all
```

## Examples of Customizations

> Custom ```xp.conf```

The ```xp.conf``` allows some basic customizations (reservation, site, node collocation)

```yaml
# OAR jobs
site            "toulouse"
walltime        '1:00:00'
nodemanager     2
datanode        3
colocated       true # whether datanode, nodemanager are colocated

# Your Grid'5000 login
user        'msimonin'

# SSH configuration
public_key      File.expand_path '~/.ssh/id_rsa.pub'
gateway         "#{self[:user]}@access.grid5000.fr"
```

> Custom ```*-site.xml``` configuration files

  * Override the file in ```templates/*-site.xml```
  * Restart the cluster with : ```cap cluster:stop configure cluster:start```
