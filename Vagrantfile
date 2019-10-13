# Notes:
# - Make sure `VBoxHeadless` process is not swapping out. If it does try to lower the requested memory for the VM.

microk8s_ip = "192.168.51.101"
k8s_version = "1.15/stable"
dns_forwarders = ["8.8.8.8", "8.8.4.4"]

# VM configuration
cpus   = 2    # Cores
memory = 4    # GiB
swap   = 4    # GiB
disk   = 32   # GiB

variables = <<~SHELL
  MICROK8S_IP="#{microk8s_ip}"
  K8S_VERSION="#{k8s_version}"
  DNS_FORWARDERS="#{dns_forwarders.join(" ")}"
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
  config.vm.box = "ubuntu/bionic64"
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

        # Enable privileged containers
        microk8s.stop
        echo '--allow-privileged=true' | tee -a /var/snap/microk8s/current/args/kube-apiserver
      }

      install_helm () {
        snap install helm --classic
      }

      start_microk8s () {
        # Start microk8s
        microk8s.start
      }

      enable_addons () {
        # Enable dns, storage and metrics-server
        microk8s.enable dns
        sleep 5
        microk8s.enable storage
        microk8s.enable metrics-server
      }

      main () {
        run_once prepare
        run_once install_microk8s
        run_once install_helm
        run_once start_microk8s
        run_once enable_addons
      }

      main "$@"
    SHELL
  end

  config.vm.provision "shell", name: "user", upload_path: "/tmp/vagrant-shell-user", privileged: false do |s|
    s.inline = variables + scripts_common + <<~'SHELL'
      configure_dns_forwarders () {
        # Update coredns settings
        # FIXME: This is a not reliable way of configuring dns and may not work sometimes, see https://github.com/ubuntu/microk8s/issues/543
        local dns_patch
        dns_patch=$(kubectl -n kube-system get configmap/coredns -o jsonpath='{.data.Corefile}' | sed "s/\(forward .\) 8.8.8.8 8.8.4.4/\1 $DNS_FORWARDERS/" | jq -sR '{"data":{"Corefile":.}}')
        kubectl -n kube-system patch configmap/coredns --type merge -p "$dns_patch"

        # Delete coredns pod to make sure the new settings are applied (sometimes coredns gets stuck in a failed state and not restarted)
        kubectl -n kube-system delete pod -l k8s-app=kube-dns
      }

      helm_init () {
        # Initialize helm and wait for tiller to start (this will deploy tiller to k8s)
        helm init --wait --history-max 200
        helm repo remove local >/dev/null || true
      }

      main () {
        run_once configure_dns_forwarders
        run_once helm_init
      }

      main "$@"
    SHELL
  end
end
