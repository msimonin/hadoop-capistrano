require 'rubygems'
require 'bundler/setup'
require 'xp5k'
require 'yaml'
require 'colored'


# Specific configuration 
# You might want to change it according to you needs
set :g5k_user, "#{XP5K::Config["user"]}"
set :gateway, XP5K::Config[:gateway] if XP5K::Config[:gateway]
set :ssh_public, XP5K::Config[:public_key]
XP5K::Config.load

$myxp = XP5K::XP.new(:logger => logger)

roles = []
roles << XP5K::Role.new({ :name => 'master', :size => 1 })
nodes = 1
roles << XP5K::Role.new({ :name => 'nodemanager', :size => XP5K::Config[:nodemanager] })
nodes += XP5K::Config[:nodemanager]

if XP5K::Config[:colocated]
  roles << XP5K::Role.new({ :name => 'datanode', :size => XP5K::Config[:datanode], :inner => 'nodemanager' })
else
  roles << XP5K::Role.new({ :name => 'datanode', :size => XP5K::Config[:datanode] })
  nodes += XP5K::Config[:datanode]
end

$myxp.define_job({
  :resources               => ["nodes=#{nodes}, walltime=#{XP5K::Config[:walltime]}"],
  :site                    => "#{XP5K::Config[:site]}",
  :types                   => ["deploy"],
  :name                    => "hadoop",
  :roles                   => roles,
  :command                 => "sleep 86400"
})

#####
set :user, "root"
set :tarball_url, "http://www.eu.apache.org/dist/hadoop/common/hadoop-2.6.0/hadoop-2.6.0.tar.gz"
set :tarball_destination, "/opt"
set :hadoop_home, "/opt/hadoop"
set :hadoop_conf, "#{hadoop_home}/etc/hadoop/"
set :hadoop_bin, "#{hadoop_home}/bin/"
set :hadoop_sbin, "#{hadoop_home}/sbin/"
set :hadoop_examples_dir, "#{hadoop_home}/share/hadoop/mapreduce"
set :wget, "http_proxy=http://proxy:3128 https_proxy=http://proxy:3128 wget"
set :tmp_dir, "./tmp"


$myxp.define_deployment({
  :environment => "wheezy-x64-nfs",
  :site        => "#{XP5K::Config[:site]}",
  :jobs        => %w(hadoop),
  :key         => File.read("#{ssh_public}")
})

# Define roles
role :master do
  $myxp.role_with_name('master').servers
end

role :nodemanager do
  $myxp.role_with_name('nodemanager').servers
end

role :datanode do
  $myxp.role_with_name('datanode').servers
end

role :hadoop do
  $myxp.job_with_name('hadoop')['assigned_nodes']
end

desc 'Automatic deployment'
task :automatic do
  submit
  deploy
  prepare::default
  configure::default
  cluster::format_hdfs
  cluster::start
end


desc 'Submit jobs'
task :submit  do
  $myxp.submit
  $myxp.wait_for_jobs
end

desc 'Deploy with Kadeploy'
task :deploy  do
  $myxp.deploy
end

namespace :prepare do

  desc 'Prepare nodes'
  task 'default' do
    install
    packages
  end


  desc 'Download tarball'
  task :install, :roles => [:hadoop] do
    run "mkdir -p #{tarball_destination}"
    logger.debug "Download the hadoop distribution"
    run "#{wget} #{tarball_url} -O #{tarball_destination}/hadoop.tar.gz 2>1"
    logger.debug "Untar the hadoop distribution"
    run "cd #{tarball_destination} && tar -xvzf hadoop.tar.gz"
    run "cd #{tarball_destination} && mv hadoop-* hadoop"
  end


  desc 'Install extra pacakges'
  task :packages, :roles => [:hadoop] do
    run "apt-get update"
    run "apt-get install -y openjdk-7-jre openjdk-7-jdk"
  end

  end

namespace :configure do
  desc 'configure nodes'
  task 'default' do
    topology::default
    core_site::default
    yarn_site::default
    mapred_site::default
    hadoop_env
    permissions
  end

  namespace :topology do

    desc 'create the topology of the cluster'
    task :default do
      generate
      transfer
    end

    task :generate do
      master = $myxp.role_with_name('master').servers.first
      File.open("tmp/master", "w") {|f| f.write master }

      slaves = $myxp.role_with_name('nodemanager').servers
      File.open("tmp/slaves", "w") {|f| f.write slaves.join("\n")}
    end

    task :transfer, :roles => [:master] do
      upload "tmp/master", "#{hadoop_conf}/master", :via => :scp
      upload "tmp/slaves", "#{hadoop_conf}/slaves", :via => :scp
    end

  end

  namespace :core_site do

    desc 'configure core-site.xml'
    task :default do
      generate
      transfer
    end

    task :generate do
      template = File.read("templates/core-site.xml.erb")
      renderer = ERB.new(template)
      @namenode = $myxp.role_with_name('master').servers.first
      generate = renderer.result(binding)
      core_site = File.open("tmp/core-site.xml", "w")
      core_site.write(generate)
      core_site.close
    end
    
    task :transfer, :roles => [:hadoop] do
      upload "tmp/core-site.xml", "#{hadoop_conf}/core-site.xml", :via => :scp
    end
  end



  namespace :yarn_site do

    desc 'configure yarn-site.xml'
    task :default do
      generate
      transfer
    end

    task :generate do
      template = File.read("templates/yarn-site.xml.erb")
      renderer = ERB.new(template)
      @jobtracker= $myxp.role_with_name('master').servers.first
      generate = renderer.result(binding)
      core_site = File.open("tmp/yarn-site.xml", "w")
      core_site.write(generate)
      core_site.close
    end
    
    task :transfer, :roles => [:hadoop] do
      upload "tmp/yarn-site.xml", "#{hadoop_conf}/yarn-site.xml", :via => :scp
    end
  end

  namespace :mapred_site do

    desc 'configure mapred-site.xml'
    task :default do
      generate
      transfer
    end

    task :generate do
      template = File.read("templates/mapred-site.xml.erb")
      renderer = ERB.new(template)
      @jobtracker = $myxp.role_with_name('master').servers.first
      generate = renderer.result(binding)
      core_site = File.open("tmp/mapred-site.xml", "w")
      core_site.write(generate)
      core_site.close
    end
    
    task :transfer, :roles => [:hadoop] do
      upload "tmp/mapred-site.xml", "#{hadoop_conf}/mapred-site.xml", :via => :scp
    end

  end

  task :hadoop_env, :roles => [:hadoop] do
    run "perl -pi -e 's,.*JAVA_HOME.*,export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64/jre,g' #{hadoop_conf}/hadoop-env.sh"
  end

  desc "Give #{g5k_user} permission to deploy hadoop"
    task :permissions, :roles => [:hadoop] do
    logger.debug "Give permissions to #{g5k_user}"
    run "chown -R #{g5k_user}:users /opt/hadoop*"
  end

end

namespace :cluster do
  
  desc 'Format the cluster'
  task :format_hdfs, :roles => [:master] do
    run "su #{g5k_user} -c '#{hadoop_bin}/hadoop namenode -format'"
  end

  namespace :start do
    desc 'Start the cluster'
    task :default do
      start_from_master
      start_nodemanager
      start_datanode
    end

    task :start_from_master, :roles => [:master] do
      # start namenode
      run "su #{g5k_user} -c '#{hadoop_sbin}/hadoop-daemon.sh --script hdfs start namenode'"
      # run resource manager
      run "su #{g5k_user} -c '#{hadoop_sbin}/yarn-daemon.sh start resourcemanager'"
      # start history server
      run "su #{g5k_user} -c '#{hadoop_sbin}/mr-jobhistory-daemon.sh start historyserver'"
    end

    task :start_nodemanager, :roles => [:nodemanager] do
      # start all the nodemanager 
      run "su #{g5k_user} -c '#{hadoop_sbin}/yarn-daemon.sh start nodemanager'"
    end

    task :start_datanode, :roles => [:datanode] do
      run "su #{g5k_user} -c '#{hadoop_sbin}/hadoop-daemon.sh --script hdfs start datanode'"
    end
  end

  namespace :stop do
    desc 'Stop the cluster'
    task :default do
      stop_from_master
      stop_nodemanager
      stop_datanode
    end

    task :stop_from_master, :roles => [:master] do
      # stop namenode
      run "su #{g5k_user} -c '#{hadoop_sbin}/hadoop-daemon.sh --script hdfs stop namenode'"
      # run resource manager
      run "su #{g5k_user} -c '#{hadoop_sbin}/yarn-daemon.sh stop resourcemanager'"
      # stop history server
      run "su #{g5k_user} -c '#{hadoop_sbin}/mr-jobhistory-daemon.sh stop historyserver'"
    end

    task :stop_nodemanager, :roles => [:nodemanager] do
      # stop all the nodemanager 
      run "su #{g5k_user} -c '#{hadoop_sbin}/yarn-daemon.sh stop nodemanager'"
    end

    task :stop_datanode, :roles => [:datanode] do
      run "su #{g5k_user} -c '#{hadoop_sbin}/hadoop-daemon.sh --script hdfs stop datanode'"
    end
  end


  desc 'Status of the cluster'
  task :status, :roles => [:hadoop] do
    run "jps"
  end

end

desc 'Launch a benchmark, BENC variable has to be set to the bench name and parameter'
task :benchmark, :roles => [:master] do
  set :hadoop_bench, ENV["BENCH"] 
  run "su #{g5k_user} -c '#{hadoop_bin}/yarn jar #{hadoop_examples_dir}/hadoop-mapreduce-examples*.jar #{hadoop_bench}'"
end

desc 'Remove all'
task :uninstall, :roles => [:hadoop] do
  run "rm -rf /opt/hadoop*"
end



desc 'Remove all running jobs'
task :clean do
  $myxp.clean
end

desc 'Describe the cluster'
task :describe do
  servers = find_servers
  servers_by_roles = {}
  servers.each do |server|
    role_names = role_names_for_host(server)
    role_names.each do |role|
      servers_by_roles[role] ||= [] 
      servers_by_roles[role] << server
    end
  end 
  puts "+----------------------------------------------------------------------+"
  servers_by_roles.each do |role, servers|
    print "|"+"%-30s".blue % role
    server = servers.pop
    puts "%-40s|" % server
    servers.each do |server|
      print "|"+"%-30s" % " "
      puts "%-40s|" % server
    end 
    puts "+----------------------------------------------------------------------+"
  end
end

