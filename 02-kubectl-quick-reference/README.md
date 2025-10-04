# kubectl Quick Notes

This document summarizes useful commands for aliasing, autocomplete, and switching clusters/users when working with `kubectl`.

---

## 🔹 1. Alias for kubectl

Instead of typing `kubectl` each time, set an alias.

### Bash
```bash

## Add the below in .bashrc

alias k=kubectl
complete -o default -F __start_kubectl k
source <(kubectl completion bash)

source ~/.bashrc

## Know your cluster and switch between clusters 

kubectl config get-contexts

kubectl config current-context

kubectl config use-context <context-name>

## switch users

kubectl config set-context --current --user=<user-name>
