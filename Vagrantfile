# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vagrant.plugins = ["vagrant-reload", "vagrant-scp"]
end

openfortivpn_ubuntu1804_vagrantfile = './vagrant/Vagrantfile.openfortivpn-ubuntu1804'
load openfortivpn_ubuntu1804_vagrantfile if File.exists?(openfortivpn_ubuntu1804_vagrantfile)
