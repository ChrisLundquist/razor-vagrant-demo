require 'rake'
require 'net/ssh'
require 'net/scp'
require "net/http"
require "json"

USER = PASSWORD = 'vagrant'
HOSTS = {
  'gold' => '172.16.0.2',
}

def ssh(host, user = USER, password = PASSWORD)
  (@ssh_connections ||= {})[host.to_s] ||= begin
                                             Net::SSH.start(HOSTS[host.to_s], user, :password => password)
                                           end
end

def scp(host, local_path, remote_path, user = USER, password = PASSWORD)
  ssh(host, user, password).scp.upload!(local_path, remote_path)
end

def razor(*a)
  ssh(:gold).exec!(['sudo /opt/razor/bin/razor', *a].join(' '))
end

def razor_api(*a) # As of 2012.06.27 this image api isn't implemented
  if(a.first == "image")
    return ghetto_api(a)
  end
  # We create a URI object with our IP, Port, and the path to the Slice (Node)
  uri = URI "http://#{HOSTS['gold']}:8026/razor/api/#{a.join("/")}"
  # We run an HTTP GET against our Node Slice API
  res = Net::HTTP.get(uri)
  # This returns
  response_hash = JSON.parse(res)
end

#XXX Hack until images api works
def ghetto_api(*a)
  result = razor(a)
  slice = a.first.first.capitalize

  # Each object is separated by a blank line
  # Plus an extra blank link at the end we need to trim
  objects = result.split(/^$/)[0...-1]

  objects.map! do |object|
    # Break it into lines and select the hash rocket ones
    # Giving us an Array of stuff that looks like
    # UUID =>  6LlBHrlbmHAEM8XRwjbQjB
    # Type =>  OS Install
    attributes = object.split("\n").select{ |i| i.include?( "=>" ) }

    hash = attributes.each_with_object({}) do |line,hash|
      key,value = line.split("=>").map { |i| i.strip } # remove leading and trailing spaces on keys / values
      hash[key] = value
    end
  end

  {"resource"=>"ProjectRazor::Slice::#{slice}", "command"=>"#{slice.downcase}_query_all", "result"=>"Ok", "http_err_code"=>200, "errcode"=>0, "response"=> objects}
end

def download(options)
  url = options[:url]
  file_name = options[:as] || File.basename(url)
  folder = options[:to] || "."

  file_path = folder + "/" + file_name

  command = "curl #{url} -o #{file_path}"
  if File.exists?(file_path)
    puts "File #{file_path} already exists. skipping download!"
  else
    system(command) or raise "Download command failed: #{command}"
  end
end


namespace :microkernel do

  url = 'https://github.com/downloads/puppetlabs/Razor/rz_mk_dev-image.0.8.9.0.iso'
  file_name = File.basename(url)
  remote_file_name = "/tmp/#{file_name}"

  file file_name do
    download(:url => url)
  end

  task :upload => file_name do
    scp(:gold, file_name, remote_file_name)
  end

  task :setup => :upload do
    puts razor('image', 'add', 'mk', remote_file_name)
  end
end

task :microkernel => 'microkernel:setup'

namespace :ubuntu do

  url = 'http://releases.ubuntu.com/precise/ubuntu-12.04-server-amd64.iso'
  file_name = File.basename(url)
  remote_file_name = "/tmp/#{file_name}"

  file file_name do
    download(:url => url)
  end

  task :upload => file_name do
    scp(:gold, file_name, remote_file_name)
  end

  task :setup => :upload do
    puts razor('image', 'add', 'os', remote_file_name, 'ubuntu_precise', '12.04')
  end
end

task :ubuntu => 'ubuntu:setup'

namespace :scientific do

  url = 'ftp://ftp.scientificlinux.org/linux/scientific/6.0/x86_64/iso/SL-62-x86_64-2012-02-06-boot.iso'
  file_name = File.basename(url)
  remote_file_name = "/tmp/#{file_name}"

  file file_name do
    download(:url => url)
  end

  task :upload => file_name do
    scp(:gold, file_name, remote_file_name)
  end

  task :setup => :upload do
    puts razor('image', 'add', 'os', remote_file_name, 'scientific', '6.2')
  end

  task :create_model => :setup do
    response = razor_api("image")['response']
    target_uuid = response.select { |image| image["OS Name"] == "scientific" }.first["UUID"]
    puts razor('model','add',"template=centos_6", "label=install_centos_6", "image_uuid=#{target_uuid}")
  end

  task :create_policy => :create_model do
  end
end

task :scientific => 'scientific:create_model'

namespace :centos do

  url = "http://mirror.metrocast.net/centos/6.2/isos/x86_64/CentOS-6.2-x86_64-minimal.iso"
  file_name = File.basename(url)
  remote_file_name = "/tmp/#{file_name}"

  file file_name do
    download(:url => url)
  end

  task :upload => file_name do
    scp(:gold, file_name, remote_file_name)
  end

  task :setup => :upload do
    razor('image', 'add', 'os', remote_file_name, 'centos', '6.2')
  end
end

task :centos => 'centos:setup'
