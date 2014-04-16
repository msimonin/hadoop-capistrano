require 'rubygems'
require 'bundler/setup'
require 'xp5k'
require 'yaml'

load 'config/deploy.rb'

set :g5k_user, "msimonin"
set :ssh_public,  File.join(ENV["HOME"], ".ssh", "id_rsa.pub")

set :tarball_url, "http://www.eu.apache.org/dist/hadoop/common/hadoop-1.2.1/hadoop-1.2.1.tar.gz"
set :tarball_destination, "/opt/hadoop"
set :wget, "http_proxy=http://proxy:3128 https_proxy=http://proxy:3128 wget"
set :tmp_dir, "./tmp"

XP5K::Config.load

$myxp = XP5K::XP.new(:logger => logger)

$myxp.define_job({
  :resources       => ["nodes=4, walltime=3:00:00"],
  :site           => "lille",
  :types          => ["deploy"],
  :name           => "hadoop",
  :roles      => [
    XP5K::Role.new({ :name => 'master', :size => 1 }),
    XP5K::Role.new({ :name => 'slaves', :size => 3 })
  ],
  :command        => "sleep 86400"
})

$myxp.define_deployment({
  :environment => "wheezy-x64-nfs",
  :site => "lille",
  :roles => %w(master slaves),
  :key => File.read("#{ssh_public}")
})

# Define roles
role :master do
  $myxp.role_with_name('master').servers
end

role :slave do
  $myxp.role_with_name('slaves').servers
end

role :hadoop do
  $myxp.job_with_name('hadoop')['assigned_nodes']
end

desc 'Submit jobs'
task :submit  do
  $myxp.submit
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
    permissions
  end


  desc 'Download tarball'
  task :install, :roles => [:hadoop] do
    set :user, "root"
    run "mkdir -p #{tarball_destination}"
    logger.debug "Download the hadoop distribution"
    run "#{wget} #{tarball_url} -O #{tarball_destination}/hadoop.tar.gz 2>1"
    logger.debug "Untar the hadoop distribution"
    run "cd #{tarball_destination} && tar -xvzf hadoop.tar.gz"
  end


  desc 'Install extra pacakges'
  task :packages, :roles => [:hadoop] do
    set :user, "root"
    run "apt-get update"
    run "apt-get install -y openjdk-7-jre openjdk-7-jdk"
  end


  desc "Give #{g5k_user} permission to deploy hadoop"
  task :permissions, :roles => [:hadoop] do
    set :user, "root"
    logger.debug "Give permissions to #{g5k_user}"
    run "chown -R #{g5k_user}:users /opt/hadoop*"
  end

end

namespace :configure do
  desc 'configure nodes'
  task 'default' do
    topology::default
    core_site::default
    mapred_site::default
    hadoop_env
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

      slaves = $myxp.role_with_name('slaves').servers
      File.open("tmp/slaves", "w") {|f| f.write slaves.join("\n")}
    end

    task :transfer, :roles => [:master] do
      set :user, "#{g5k_user}"
      upload "tmp/master", "#{tarball_destination}/hadoop-1.2.1/conf/master", :via => :scp
      upload "tmp/slaves", "#{tarball_destination}/hadoop-1.2.1/conf/slaves", :via => :scp
    end

  end

  namespace :core_site do

    desc 'configure core-site.xml'
    task :default do
      generate
      transfer
    end

    task :generate, :roles => [:all] do
      template = File.read("templates/core-site.xml.erb")
      renderer = ERB.new(template)
      @namenode = $myxp.role_with_name('master').servers.first
      generate = renderer.result(binding)
      core_site = File.open("tmp/core-site.xml", "w")
      core_site.write(generate)
      core_site.close
    end
    
    task :transfer, :roles => [:hadoop] do
      set :user, "#{g5k_user}"
      upload "tmp/core-site.xml", "#{tarball_destination}/hadoop-1.2.1/conf/core-site.xml", :via => :scp
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
      set :user, "#{g5k_user}"
      upload "tmp/mapred-site.xml", "#{tarball_destination}/hadoop-1.2.1/conf/mapred-site.xml", :via => :scp
    end

  end

  task :hadoop_env, :roles => [:hadoop] do
    set :user, "#{g5k_user}"
    run "perl -pi -e 's,.*JAVA_HOME.*,export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64/jre,g' #{tarball_destination}/hadoop-1.2.1/conf/hadoop-env.sh"
  end
end

namespace :cluster do
  
  desc 'Format the cluster'
  task :format_hdfs, :roles => [:master] do
    set :user, "#{g5k_user}"
    run "#{tarball_destination}/hadoop-1.2.1/bin/hadoop namenode -format"
  end

  desc 'Start the cluster'
  task :start, :roles => [:master] do
    set :user, "#{g5k_user}"
    run "#{tarball_destination}/hadoop-1.2.1/bin/start-all.sh"
  end

  desc 'Stop the cluster'
  task :stop, :roles => [:master] do
    set :user, "#{g5k_user}"
    run "#{tarball_destination}/hadoop-1.2.1/bin/stop-all.sh"
  end

  desc 'Status of the cluster'
  task :status, :roles => [:hadoop] do
    set :user, "#{g5k_user}"
    run "jps"
  end

end

desc 'Remove all'
task :uninstall, :roles => [:hadoop] do
  set :user, "root"
  run "rm -rf /opt/hadoop*"
end



desc 'Remove all running jobs'
task :clean do
  $myxp.clean
end


