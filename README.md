# microk8s-to-go
microk8s on Vagrant

# Features
- Based on the official Ubuntu image with microk8s and helm installed from snaps
- Lightweight
- Enabled plugins:
  - dns
  - storage
  - metrics-server

# How to use
## Starting up
```
git clone https://github.com/giner/microk8s-to-go
cd microk8s-to-go
# Make sure you have a recent version of Vagrant which supports automatic installation for plugins.
vagrant up

# ssh-in
vagrant ssh
kubectl ...
```

## Cleaning up
```
# When you are done or messed up
vagrant destroy
```
