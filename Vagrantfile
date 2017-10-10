# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'getoptlong'

opts = GetoptLong.new(
  [ '--test-name', GetoptLong::OPTIONAL_ARGUMENT ],
  [ '--provider', GetoptLong::OPTIONAL_ARGUMENT ],
  [ '--help', '-h', GetoptLong::OPTIONAL_ARGUMENT ]
)

test_name=''

opts.each do |opt, arg|
  case opt
    when '--test-name'
      test_name=arg
  end
end

Vagrant.configure("2") do |config|

  config.vm.box = "fedora/26-cloud-base"

  config.vm.provider "libvirt" do |libvirt|
    libvirt.memory = 2048
    libvirt.cpu_mode = "host-model"
  end

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
  end

  # sync tests folder
  config.vm.synced_folder "tests", "/mnt/tests", type: "sshfs"
  config.vm.synced_folder "scripts", "/scripts", type: "sshfs"

  # provision script
  config.vm.provision "shell", inline: <<-SHELL
        set -x
        # make sure we have updated system to mitigate various issues with dnf later on
        dnf update -y --refresh

        # install git
        dnf install -y git

        # install ansible test runner prerequisites
        dnf install -y ansible python2-dnf libselinux-python beakerlib standard-test-roles vim

        # Uncomment this if you want to use the latest standard test roles from merlin's copr
        # dnf copr enable -y merlinm/standard-test-roles
        # dnf --enablerepo=updates-testing install -y  standard-test-roles

        #If test name was given run the tests for it
        if [ -n "#{test_name}" ]; then
            git clone https://upstreamfirst.fedorainfracloud.org/#{test_name}.git
            if [ $? -ne 0 ]; then
                echo "FAIL: Could not clone repo for #{test_name}"
                exit 1
            fi
            cd #{test_name}

            #Check if tests have any reference to bugzillas
            #using \\< and \\> to match whole words only
            bz_regex="\\(\\<bug\\>\\|bz\\)\\([[:space:]]\\|#\\|:\\)*[[:digit:]]\\{4\\}"
            grep --exclude-dir=.git -r -i -e "$bz_regex" .
            if [ $? -ne 1 ]; then
                echo "FAIL: It seems there are references to bugzilla numbers!"
                exit 1
            fi


            ANSIBLE_INVENTORY=$(test -e inventory && echo inventory || echo /usr/share/ansible/inventory)
            TEST_SUBJECTS="" 
            TEST_ARTIFACTS=$PWD/artifacts
            ansible-playbook --tags classic tests.yml
            if [ $? -ne 0 ]; then
                echo "FAIL: Could not execute ansible-playbook"
                exit 1
            fi
            cat $PWD/artifacts/test.log
            #Check if any test did not pass
            grep -ve ^PASS $PWD/artifacts/test.log
            if [ $? -eq 1 ]; then
                echo "PASS: all tests passed."
                exit 0
            fi
            echo "FAIL: At least one test did not PASS."
            exit 1
        fi
    SHELL
end
