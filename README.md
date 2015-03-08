Hadoop-capistrano
=================

Hadoop deployment for Grid'5000.

* hadoop 1.2.1 -> branch ```hadoop 1```
* hadoop 2.6.0 -> branch ```hadoop 2``` and ```master```


## Default deployment


* One ```master``` node with
  * ResourceManager
  * NameNode
  * SecondaryNameNode
  * JobHistoryServer

* Some ```slave``` nodes with  
  * Namenode
  * Nodemanager


### Deploy !

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

The ```Capfile``` is given as a starting point. Each experiment setup will probably require to write your own. Here are some examples of possible customizations :


> Custom reservation (number of nodes / walltime / site ...)

  * Check the first section of the ```Capfile```

> Custom ```*-site.xml``` configuration files

  * Override the file in ```templates/*-site.xml```
  * Restart the cluster with : ```cap cluster:stop configure cluster:start```

> Change the version of hadoop

  * You can change the ```:tarball_url``` variable in the ```Capfile```

> Separation between mapreduce node and datanode

  * The most common deployment model for hadoop colocates compute nodes and data nodes. If you want to separate them you'll need to deploy your own hdfs on a separated set of nodes (a new role in the deployment recipe could be added).
The deployment script to the default model but can easily be extended to support both.

> ?
