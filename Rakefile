ROOT = File.expand_path(File.dirname(__FILE__))
$LOAD_PATH.unshift File.join(ROOT, 'lib')

require 'colorize'
require 'ssh_driver'
require 'aws_driver'
require 'highline/import'
require 'rake/clean'
require 'json'

OUTPUT = 'build'
CLEAN << OUTPUT
directory OUTPUT

TARBALL_NAME = 'chef-solo.tgz'

EC2 = AWSDriver.new(ROOT)

task :default => :check_credentials

desc 'Create a named node'
task :start, [:node_name] => [:check_credentials] do |t, args|
  node = EC2.start_node args.node_name
  write_connect_script node
end

desc 'Terminate named node'
task :stop, [:node_name] => [:check_credentials] do |t, args|
  filename = connect_script_name args.node_name
  File.delete(filename) if File.exists?(filename)
  EC2.terminate_node args.node_name
end

desc 'Terminate all running nodes'
task :stop_all => :check_credentials do
  Dir[connect_script_name('*')].each { |script| File.delete script }
  EC2.terminate_all
end

task :check_credentials do
  unless EC2.credentials?
    access_key_id = ask('EC2 Access Key ID? ')
    secret_access_key = ask('EC2 Secret Access Key? ')
    EC2.save_credentials access_key_id.to_s, secret_access_key.to_s
  end
  unless EC2.ssh_key_file?
    aws_key = ask('EC2 SSH Key File? ')
    EC2.save_ssh_key_file aws_key.to_s
  end
end

desc 'Provision a broker node with chef-solo'
task :provision_broker => [:check_credentials, OUTPUT] do |t, args|
  provision 'broker', 'config/broker.json'
end

desc 'Provision a named node with chef-solo'
task :provision, [:node_name] => [:check_credentials, OUTPUT] do |t, args|
  broker = EC2.start_node 'broker'

  config = open('config/node.json') { |fp| JSON.parse fp.read }

  config['mcollective'] = Hash.new
  config['mcollective']['stomp_host'] = broker.hostname

  open("#{OUTPUT}/node.json", 'w') { |fp| fp.puts JSON.pretty_generate(config) }

  provision args.node_name, "#{OUTPUT}/node.json"
end

desc 'Run mcollective ping on the broker'
task :mco_ping => :check_credentials do
  node = EC2.start_node 'broker'
  SSHDriver.start(node.hostname, node.user, node.keyfile) do |ssh|
    ssh.exec! 'mco ping'
  end
end

def provision(node_name, config_file)
  node = EC2.start_node node_name
  SSHDriver.start(node.hostname, node.user, node.keyfile) do |ssh|
    install_chef_solo ssh
    ssh.upload config_file, "chef/#{File.basename(config_file)}"
    ssh.exec! "sudo chef-solo -c ~/chef/solo.rb -j ~/chef/#{File.basename(config_file)}"
  end
  write_connect_script node
end

def install_chef_solo(ssh)
  result = ssh.exec 'chef-solo --version'
  unless result.success
    ssh.exec! 'sudo yum -y update'
    ssh.exec! 'sudo yum -y install ruby ruby-devel ruby-ri ruby-rdoc gcc gcc-c++ automake autoconf make curl dmidecode rubygems'
    ssh.exec! 'sudo gem install chef --no-ri --no-rdoc'
  end
  sh "tar cvzf #{OUTPUT}/#{TARBALL_NAME} chef"
  ssh.exec 'rm -rf chef*'
  ssh.exec "mkdir /tmp/chef-solo"
  ssh.upload "#{OUTPUT}/#{TARBALL_NAME}", '.'
  ssh.exec! "tar xvzf #{TARBALL_NAME}"
end

def write_connect_script(node)
  filename = connect_script_name node.name
  File.open(filename, 'w') do |out|
    out.puts "#!/bin/sh"
    # use /dev/null as known_hosts to keep EC2 signatures from filling up the normal known_hosts file
    out.puts "ssh -i #{node.keyfile} -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no #{node.user}@#{node.hostname}"
  end
  File.chmod(0755, filename)
  puts "Connect to #{node.name} using ./#{filename}".green
end

def connect_script_name(node_name)
  "connect_" + node_name
end
