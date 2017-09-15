#
# Rakefile
#
# !!!!!!!!!!!!!!!!!!!!!!!!!!!!! STOP !!!!!!!!!!!!!!!!!!!!!!!!!!!!!
# Do not modify this file unless it is in the chef-repo repository
# under the cookbook-skeleton directory. Review the README.md file
# in that directory for additional information.
#
# Based on https://github.com/mlafeldt/skeleton-cookbook

require 'chef/cookbook/metadata'
require 'chef'
require 'json'
require 'open-uri'
require 'fileutils'
require 'rubygems/package'
require 'zlib'

def windows?
  windows = false
  if ::File.exist?('Vagrantfile')
    windows = !open('Vagrantfile').select { |s| /^\W*#/ !~ s }.grep(/config\.vm\.guest\W*=/).grep(/windows/i).empty?
  elsif ::File.exist?('packer_config.json')
    json = JSON.parse(File.read('packer_config.json'))
    if json.key?('source_vmx_name')
      windows = json['source_vmx_name'].include?('windows')
    elsif json.key?('vmware_guest_os_type')
      windows = json['vmware_guest_os_type'].include?('windows')
    else
      fail "Could not determine if project builds a windows host using packer_config.json"
    end
  else
    fail "Could not determine if project builds a windows host"
  end
  windows
end

def get_vagrant_version
  output = `vagrant --version`
  output.split(" ").last
end

desc "Run RuboCop style and lint checks for CI"
task :rubocop_ci do
  begin
    sh 'rubocop --require rubocop/formatter/checkstyle_formatter --format RuboCop::Formatter::CheckstyleFormatter --no-color --rails --out rubocop-checkstyle.xml'
  rescue
    puts "rubocop failed!"
  end
end

desc "Run RuboCop style and lint checks"
task :rubocop do
  begin
    sh 'rubocop'
  rescue
    puts "rubocop failed!"
  end
end

task :kitchen_setup do
  if windows?
    # kitchen-vagrant does not fully support windows. The gem below is a workaround.
    begin
      gem "kitchen-driver-vagrant_provision"
    rescue Gem::LoadError
      raise "kitchen-driver-vagrant_provision gem must be installed to continue. \
        Install it with chef gem install kitchen-driver-vagrant_provision"
    end
  end
end

desc "Run Kitchen tests"
task :kitchen => [:berks_vendor, :kitchen_setup] do
  sh "kitchen test"
end

desc "Run Kitchen tests for CI"
task :kitchen_ci => [:berks_vendor, :kitchen_setup] do
  sh "kitchen test -d always"
end

desc "Run Foodcritic lint checks"
task :foodcritic do
  sh "foodcritic --epic-fail none ."
end

desc "Run Berkshelf and install cookbooks for Vagrant"
task :berks_vendor do
  puts "Installing cookbooks locally using Berkshelf"
  FileUtils.rm_rf Dir.glob(".cookbooks/*")
  sh 'berks vendor .cookbooks'
end

desc "Package the cookbook and dependencies into a ZIP"
task :package => [:berks_vendor] do
  metadata = Chef::Cookbook::Metadata.new
  metadata.from_file("metadata.rb")
  archive = "#{metadata.name}-#{metadata.version}.zip"
  FileUtils.rm archive, :force => true
  Dir.chdir('.cookbooks') do
    sh "zip -r ../#{archive} ."
  end
end

desc "Continuous integration master build"
task :ci_master => [
    :bump,
    :package,
    :test_ci,
    :upload
  ]

desc "Continuous integration gerrit build"
task :ci_gerrit => [
    :test_ci
  ]

desc "Run chef-solo on a running Vagrant VM"
task :chefsolo, [:host] => [:berks_vendor] do |_t, args|
  # newer versions of vagrant place the files in a more predictable location
  ver = get_vagrant_version
  puts "Vagrant version is #{ver}"
  if windows?
    if Gem::Version.new(ver) >= Gem::Version.new('1.7.3')
      sh 'vagrant winrm -e -c "chef-solo --force-formatter -c c:\vagrant-chef\solo.rb -j c:\vagrant-chef\dna.json"'
    else
      sh 'vagrant winrm -e -c "chef-solo --force-formatter -c c:\tmp\vagrant-chef-*\solo.rb -j c:\tmp\vagrant-chef-*\dna.json"'
    end
  else
    ssh_host = args.has_key?(:host) ? args[:host] : ""
    if Gem::Version.new(ver) >= Gem::Version.new('1.7.3')
      sh "vagrant ssh #{ssh_host} -c \"sudo chef-solo --force-formatter -c /tmp/vagrant-chef/solo.rb -j /tmp/vagrant-chef/dna.json\""
    else
      sh "vagrant ssh #{ssh_host} -c \"sudo chef-solo --force-formatter -c `find /tmp/vagrant-chef* -maxdepth 2 -name solo.rb` -j `find /tmp/vagrant-chef* -maxdepth 2 -name dna.json`\""
    end
  end
end

desc "Bumps cookbook version"
task :bump do
  # most of this is lifted from the knife-spork gem
  metadata_file = "metadata.rb"
  metadata = Chef::Cookbook::Metadata.new
  metadata.from_file(metadata_file)
  
  cookbook_name = metadata.name
  old_version = metadata.version
  
  version_array = old_version.split('.').collect(&:to_i)
  version_array[2] += 1
  
  new_version = version_array.join('.')
  
  new_contents = File.read(metadata_file).gsub(/(version\s+['"])[0-9\.]+(['"])/, "\\1#{new_version}\\2")
  File.open(metadata_file, 'w') { |f| f.write(new_contents) }

  puts "Successfully bumped #{cookbook_name} from #{old_version} to v#{new_version}"

  # update Berksfile.lock
  sh "berks install"

  sh "git add #{metadata_file} Berksfile.lock"
  sh "git commit -m \"bumped #{cookbook_name} version to #{new_version}\""
  sh "git tag -a v#{new_version} -m \"#{cookbook_name} v#{new_version}\""
  sh "git push origin HEAD:master"
end

desc "Upload cookbooks to Chef server"
task :upload do
  sh "berks upload"
end

desc "Run all tests"
task :test => [:berks_vendor, :rubocop, :foodcritic, :kitchen]

desc "Run all tests for CI"
task :test_ci => [:berks_vendor, :rubocop_ci, :foodcritic, :kitchen_ci]

desc "Clear any proxy environment variables that may interfere with Packer"
task :packer_proxy_check do
  if windows? && (ENV['http_proxy'] || ENV['https_proxy'])
    # WinRM (which operates over http) and vmware's communication with packer's build in
    # http server won't work with a proxy.
    puts "WARNING: Packer does not work properly when proxy environment variables are set."
    puts "The http_proxy and https_proxy environment variables are being cleared."
    ENV['http_proxy'] = nil
    ENV['https_proxy'] = nil
  end
end

desc "Create the variable file for Packer"
task :packer_var_file do
  # barrow the vsphere credentials from the knife config
  ::Chef::Config.from_file "#{ENV['HOME']}/.chef/knife.rb"
  vsphere_user = ::Chef::Config[:knife][:vsphere_user]
  vsphere_pass = ::Chef::Config[:knife][:vsphere_pass]

  raise "vsphere user not found in knife.rb" if vsphere_user.nil?
  raise "vsphere pass not found in knife.rb" if vsphere_pass.nil?

  metadata = Chef::Cookbook::Metadata.new
  metadata.from_file('metadata.rb')

  vars = {
    :cookbook_name => metadata.name,
    :version => metadata.version,
    :vsphere_user => vsphere_user,
    :vsphere_pass => vsphere_pass
  }
  
  vars.merge!(JSON.parse(File.read('packer_config.json'))) if File.exist?('packer_config.json')

  %w(
    vsphere_cluster
    vsphere_datacenter
    vsphere_host
    vsphere_datastore
    vsphere_vm_network
  ).each do |var_name|
    vars[var_name] = ENV[var_name.upcase] if ENV.has_key?(var_name.upcase) && !ENV[var_name.upcase].empty?
  end
  
  # packer cannot handle arrays
  vars.delete('additional_disk_sizes')

  File.write('packer_variables.json', vars.to_json)
end

def packer_cleanup
  File.unlink('packer_variables.json') if File.exist?('packer_variables.json')
  FileUtils.rm_rf('packer_source_box') if File.exist?('packer_source_box')
end

desc "Configure shared packer cache directory"
task :packer_cache_dir do
  ENV['PACKER_CACHE_DIR'] = File.join(ENV['HOME'], 'packer_cache')
  Dir.mkdir(ENV['PACKER_CACHE_DIR']) if !File.exist?(ENV['PACKER_CACHE_DIR'])
end

desc "Download and extract a source machine if configured"
task :packer_source_box => [:packer_cache_dir] do
  config = {}
  config.merge!(JSON.parse(File.read('packer_config.json'))) if File.exist?('packer_config.json')

  next if !config.has_key?('source_box_url')
    
  box_url = config['source_box_url']
  box_file = File.basename(URI.parse(config['source_box_url']).path)
  box_path = File.join(ENV['PACKER_CACHE_DIR'], box_file)
  
  if !File.exist?(box_path)
    puts "Downloading #{box_url} to #{box_path}"
    uri = URI(box_url)
    Net::HTTP.start(uri.host, uri.port) do |http|
      request = Net::HTTP::Get.new uri    
      http.request request do |response|
        open box_path, 'wb' do |io|
          response.read_body do |chunk|
            io.write chunk
          end
        end
      end
    end    
  end

  if config.has_key?('source_box_checksum')
    puts "Verifying box #{box_path}"
    expected_sha1 = config['source_box_checksum'] 
    actual_sha1 = Digest::SHA1.file box_path
    if expected_sha1 != actual_sha1.to_s
      fail "checksum mismatch: expected = '#{expected_sha1}', actual = '#{actual_sha1}'" 
    end
  end
  
  FileUtils.rm_rf('packer_source_box') if File.exist?('packer_source_box')
  Dir.mkdir('packer_source_box')  
  
  puts "Extracting box #{box_path}"
  `tar -xz -f #{box_path} -C packer_source_box`
  fail "Problem extracting box" if 0 != $?.exitstatus 
end

desc "Create additional disks if configured"
task :packer_create_disks do
  config = {}
  config.merge!(JSON.parse(File.read('packer_config.json'))) if File.exist?('packer_config.json')
  
  next if !config.has_key?('additional_disk_sizes')
  puts "Creating additional disks"
  disks = config['additional_disk_sizes']

  metadata = Chef::Cookbook::Metadata.new
  metadata.from_file('metadata.rb')
  cookbook_name = metadata.name
  
  vmx_file = File.open File.join('packer_source_box', config['source_vmx_name']), 'a'

  disk = 0
  disks.each do |size|
    disk += 1
    disk_file = "packer-#{cookbook_name}-#{disk}.vmdk"
    disk_path = File.join('packer_source_box', disk_file)
    File.delete(disk_path) if File.exists?(disk_path)
    `vmware-vdiskmanager -c -s #{size}GB -a lsilogic -t 0 #{disk_path}`
    fail if 0 != $?.exitstatus
    
    vmx_file.puts "scsi0:#{disk}.filename = \"#{disk_file}\""
    vmx_file.puts "scsi0:#{disk}.present = \"TRUE\""
    vmx_file.puts "scsi0:#{disk}.redo = \"\""
  end
  
  vmx_file.close
end

desc "Build a vSphere template using Packer"
task :packer_vsphere => [
    :packer_var_file,
    :packer_cache_dir,
    :packer_source_box,
    :packer_create_disks,
    :berks_vendor,
    :packer_proxy_check
  ] do
  on_error = ENV.has_key?('PACKER_ON_ERROR') ? ENV['PACKER_ON_ERROR'] : 'cleanup'
  # ENV['PACKER_LOG'] = "1"
  sh "packer build -on-error=#{on_error} -only=vsphere -var-file=packer_variables.json packer.json"
  packer_cleanup
end

desc "Build a Vagrant VMware box using Packer"
task :packer_vagrant_vmware => [
    :packer_var_file,
    :packer_cache_dir,
    :packer_source_box,
    :packer_create_disks,
    :berks_vendor,
    :packer_proxy_check
  ] do
  on_error = ENV.has_key?('PACKER_ON_ERROR') ? ENV['PACKER_ON_ERROR'] : 'cleanup'
  # ENV['PACKER_LOG'] = "1"
  sh "packer build  -on-error=#{on_error} -only=vagrant_vmware -var-file=packer_variables.json packer.json"
  packer_cleanup
end
