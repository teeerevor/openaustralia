# Capistrano 2.x recipe file
#
# Requirement cap 2.2
#
# Execute with:
# cap -S stage=<production|test> task

require 'tempfile'
# SHA and OpenSSL required for encryption/decryption
require 'digest/sha2'
require 'openssl'
# Required for processing JSON input to Chef
require 'json'

set :application, "openaustralia"

# default_run_options[:pty] = true

set :use_sudo, false

set :scm, :git
set :repository, "git://git.openaustralia.org/openaustralia.git"
set :git_enable_submodules, true
set :deploy_via, :remote_cache

set :stage, "test" unless exists? :stage

set :user, "matthewl"
role :web, "openaustralia.org"

# A great little trick I learned recently. If you have a machine running on a non-standard ssh port
# put the following in your ~/.ssh/config file:
# Host myserver.com
#   Port 1234
# This will change the port for all ssh commands on that server which saves a whole lot of typing

if stage == "production"
  set :deploy_to, "/www/www/#{application}"
elsif stage == "test"
  set :deploy_to, "/www/test/#{application}"
  set :branch, "test"
end

load 'deploy' if respond_to?(:namespace) # cap2 differentiator

# Symmetric encryption using AES and arbitrary length alpha key
module SymmetricCrypto
  def SymmetricCrypto.decrypt(data, key)
    aes = OpenSSL::Cipher::Cipher.new("AES-256-ECB")
    aes.decrypt
    # This ensures we get the correct length key out of an arbitrary password
    aes.key = Digest::SHA256.digest(key)
    aes.update(data) + aes.final  
  end
  
  def SymmetricCrypto.encrypt(data, key)
    aes = OpenSSL::Cipher::Cipher.new("AES-256-ECB")
    aes.encrypt
    # This ensures we get the correct length key out of an arbitrary password
    aes.key = Digest::SHA256.digest(key)
    aes.update(data) + aes.final      
  end
  
  # Little convenience methods
  def SymmetricCrypto.encrypt_file(file_in, file_out, key)
    File.open(file_out, "w") do |f|
      f << SymmetricCrypto.encrypt(File.read(file_in), key)
    end
  end
  
  def SymmetricCrypto.decrypt_file(file_in, file_out, key)
    File.open(file_out, "w") do |f|
      f << SymmetricCrypto.decrypt(File.read(file_in), key)
    end
  end
end

set :chef_private_password_file, File.expand_path("~/.chef_private_data_password")
set :chef_private_decrypted, "openaustralia-chef/config/dna.json"
set :chef_private_encrypted, "openaustralia-chef/config/dna.json.encrypted"
set :chef_public_data, "openaustralia-chef/config/dna.json.public"
set :chef_private_data, "openaustralia-chef/config/dna.json.private"

if File.exists?(fetch(:chef_private_password_file))
  password = File.read(fetch(:chef_private_password_file))
else
  password = Capistrano::CLI.password_prompt("Type in your password for decrypting secret chef data: ")
  File.open(fetch(:chef_private_password_file), "w") {|f| f << password }
end
set(:chef_encryption_password) { password }

namespace :chef do
  # Using Chef (http://wiki.opscode.com/display/chef/Home) configure the server to
  # have all the software we need. WORK IN PROGRESS
  desc "Update the server configuration using Chef"
  task :default do
    run "rm -rf /tmp/chef"
    upload("openaustralia-chef", "/tmp/chef")
    # Using "sudo -E" to ensure that environment variables are propogated to new environment
    # so that pkg_add knows to use passive ftp. What a PITA.
    #run "chef-solo -l debug -c /tmp/chef/config/solo.rb -j /tmp/chef/config/dna.json"
    sudo "-E chef-solo -c /tmp/chef/config/solo.rb -j /tmp/chef/config/dna.json"
  end
  
  task :combine_public_and_private_data do
    pub = JSON.parse(File.read(fetch(:chef_public_data)))
    pri = JSON.parse(File.read(fetch(:chef_private_data)))
    File.open(fetch(:chef_private_decrypted), "w") do |f|
      f << JSON.generate(pub.merge(pri))
    end
  end

  desc "Decrypt/Encrypt the private chef data"
  task :private_data do
    unless File.exists?(fetch(:chef_private_decrypted))
      SymmetricCrypto.decrypt_file(fetch(:chef_private_encrypted), fetch(:chef_private_decrypted), fetch(:chef_encryption_password))      
    end
    # Always encrypt the file
    SymmetricCrypto.encrypt_file(fetch(:chef_private_decrypted), fetch(:chef_private_encrypted), fetch(:chef_encryption_password))
  end
end

namespace :deploy do
	# Restart Apache because for some reason a deploy can cause trouble very occasionally (which is fixed by a restart). So, playing safe
	task :restart do
	  sudo "apachectl restart"
	end

	task :finalize_update do
		run "chmod -R g+w #{latest_release}" if fetch(:group_writable, true)
	end

	desc "After a code update, we link the images directories to the shared ones"
	task :after_update_code do
		links = {
			"#{release_path}/searchdb"                                => "../../shared/searchdb",

			"#{release_path}/openaustralia-parser/configuration.yml"  => "../../../shared/parser_configuration.yml",

			"#{release_path}/twfy/conf/general"                       => "../../../../shared/general",
			"#{release_path}/twfy/scripts/alerts-lastsent"            => "../../../../shared/alerts-lastsent",

			"#{release_path}/twfy/www/docs/sitemap.xml"               => "../../../../../shared/sitemap.xml",
			"#{release_path}/twfy/www/docs/sitemaps"                  => "../../../../../shared/sitemaps",

		  "#{release_path}/twfy/www/docs/images/mps"                => "../../../../../../shared/images/mps",
			"#{release_path}/twfy/www/docs/images/mpsL"               => "../../../../../../shared/images/mpsL",
			"#{release_path}/twfy/www/docs/regmem/scan"               => "../../../../../../shared/regmem_scan",
			"#{release_path}/twfy/www/docs/rss/mp"                    => "../../../../../../shared/rss/mp",
			"#{release_path}/twfy/www/docs/debates/debates.rss"       => "../../../../../../shared/rss/senate.rss",
			"#{release_path}/twfy/www/docs/senate/senate.rss"       => "../../../../../../shared/rss/senate.rss"
		}
		# First copy any images that have been checked into the repository to the shared area
		run "cp #{release_path}/twfy/www/docs/images/mps/* #{shared_path}/images/mps"
		run "cp #{release_path}/twfy/www/docs/images/mpsL/* #{shared_path}/images/mpsL"
		# HACK: Remove directories next because they have files in them
		run "rm -rf #{release_path}/twfy/www/docs/images/mps #{release_path}/twfy/www/docs/images/mpsL #{release_path}/twfy/www/docs/rss/mp"
		# "ln -sf <a> <b>" creates a symbolic link but deletes <b> if it already exists
		run links.map {|a| "ln -sf #{a.last} #{a.first}"}.join(";")
		# Now compile twfy/scripts/run-with-lockfile.c
		run "gcc -o #{release_path}/twfy/scripts/run-with-lockfile #{release_path}/twfy/scripts/run-with-lockfile.c"
	end
end
