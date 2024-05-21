# Original source: https://github.com/giner/microk8s-to-go

# Notes:
# - Make sure `VBoxHeadless` process is not swapping out. If it does try to lower the requested memory for the VM.

microk8s_ip = "192.168.51.101"
k8s_version = "1.29/stable"
vagrant_vm_box = "ubuntu/noble64"
dns_forwarders = ["8.8.8.8", "8.8.4.4"]  # Specify ["/etc/resolv.conf"] to use local resolver

Vagrant.require_version ">= 2.2.4"

# VM configuration
cpus   = 2    # Cores
memory = 4    # GiB
swap   = 4    # GiB
disk   = 40   # GiB

variables = <<~SHELL
  MICROK8S_IP="#{microk8s_ip}"
  K8S_VERSION="#{k8s_version}"
  DNS_FORWARDERS="#{dns_forwarders.join(",")}"
  DONE_DIR="/home/vagrant/done"
  SWAP="#{swap}"
SHELL

scripts_common = <<~'SHELL'
  set -euo pipefail
  trap exit_message EXIT

  echo_yellow () {
    local message=$@
    echo -e "\e[1;93m${message}\e[0m"
  }

  exit_message () {
    if [[ $? -ne 0 ]]; then
      echo -e "The provisioning has been interrupted. Please try again:\n\n  vagrant up --provision\n " >&2
    fi
  }

  run_once () {
    local cmd=${1^^}

    date +%FT%T
    if [[ -f "$DONE_DIR/$cmd" ]]; then
      echo "INFO: run_once $cmd - executed before, skipping"
    else
      echo_yellow "INFO: run_once $cmd - executing ..."
      "$@"
      touch "$DONE_DIR/$cmd"
    fi
  }

  retry () {
    local retries=$1; shift
    local command=("$@")
    local output

    for ((counter=1; counter<=retries; counter++)); do
      if output=$("${command[@]}" 2>&1); then
        break
      else
        sleep 1
        echo "Retrying: \"${command[@]}\" ($counter / $retries) ..." >&2
        continue
      fi
      return 1
    done

    echo "$output"
  }

  kubectl () {
    local kubectl_cmd
    kubectl_cmd=$(which kubectl)

    # Make sure the API server is up and running
    retry 5 "$kubectl_cmd" version > /dev/null

    "$kubectl_cmd" "$@"
  }
SHELL

Vagrant.configure("2") do |config|
  config.vm.box = vagrant_vm_box
  config.vm.network "private_network", ip: microk8s_ip

  config.vagrant.plugins = ["vagrant-disksize"]
  config.disksize.size = disk.to_s + 'GB'

  config.vm.provider "virtualbox" do |v|
    v.cpus = cpus
    v.memory = memory * 1024
    v.customize ["storagectl", :id, "--name", "SCSI", "--hostiocache", "on"]
  end

  config.vm.provision "shell", name: "system", upload_path: "/tmp/vagrant-shell-system", reset: true do |s|
    s.inline = variables + scripts_common + <<~'SHELL'
      prepare () {
        # Prepare directory
        mkdir -p "$DONE_DIR"
        chown vagrant:vagrant "$DONE_DIR"

        # Install some utils
        export DEBIAN_FRONTEND=noninteractive
        apt-get update
        apt-get install -y jq

        # Add swapfile
        local swapfile="/microk8s_swapfile"
        fallocate -l "${SWAP}G" "$swapfile"
        chmod 600 "$swapfile"
        mkswap "$swapfile"
        echo "$swapfile none swap sw 0 0" | tee -a /etc/fstab
        swapon -a
      }

      install_microk8s () {
        # Install microk8s
        snap install microk8s --classic --channel="$K8S_VERSION"

        # Alias microk8s.kubectl -> kubectl
        snap alias microk8s.kubectl kubectl

        # Allow kubectl to be run by user vagrant
        usermod -a -G microk8s vagrant
      }

      enable_addons () {
        # Enable dns, storage, metrics-server, helm
        microk8s disable dns && microk8s enable dns:"$DNS_FORWARDERS"
        microk8s enable hostpath-storage
        microk8s enable metrics-server
        microk8s enable helm
        snap alias microk8s.helm helm
      }

      main () {
        run_once prepare
        run_once install_microk8s
        run_once enable_addons
      }

      main "$@"
    SHELL
  end

  config.vm.provision "shell", name: "user", upload_path: "/tmp/vagrant-shell-user", privileged: false do |s|
    s.inline = variables + scripts_common + <<~'SHELL'
      configure_user_shell () {
      sed 's/^  //' > ~/.inputrc << 'EOF'
        $include  /etc/inputrc
        # alternate mappings for "page up" and "page down" to search the history
        "\e[5~": history-search-backward
        "\e[6~": history-search-forward
      EOF
      }

      main () {
        run_once configure_user_shell
      }

      main "$@"
    SHELL
  end
end
