
.PHONY: help

help: ## Helpful commands
	@awk 'BEGIN {FS = ":.*?## "} /^[a-zA-Z_-]+:.*?## / {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}' $(MAKEFILE_LIST)

.DEFAULT_GOAL := help

local/storage: ## Setup OpenEBS as Storage Class
	@echo 'Install openebs'
	helm repo add openebs https://openebs.github.io/charts
	helm repo update
	helm status openebs -o json -n openebs  | jq '.info.status' | grep -qw 'deployed' || helm install openebs --namespace openebs openebs/openebs --create-namespace
	kubectl rollout status deployment openebs-localpv-provisioner -n openebs
	kubectl rollout status deployment openebs-ndm-operator -n openebs
	# Verify StorageClasses
	kubectl get sc

local/mysql: ## Create MySql DB in local k8s cluster
	@echo 'Create MySql Database'
	kubectl apply -f secret.yaml
	kubectl apply -f storage.yaml
	kubectl apply -f mysql-deployment.yaml

local/k8s: ## Applying k8s manifests into local k8s cluster
	@echo 'Create Application deployment'
	kubectl config set-context `kubectl config current-context` --namespace production
	kubectl apply -f ../eks-prod/deployment.yaml
	kubectl apply -f ../eks-prod/service.yaml

clean: ## Clean up development cluster from local machine
	@echo 'clear project namespace, volumes and database'
	kubectl config set-context `kubectl config current-context` --namespace default
	kubectl get ns production --no-headers && kubectl delete ns production

dev: local/storage  local/mysql local/k8s ## Build development cluster in local machine