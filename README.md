Installing Kong Gateway on Azure AKS with Terraform
===========================================================

This example stands up a simple Azure AKS cluster, then provides a procedure to install postgres and Kong Gateway Enterprise.  It also makes use of Kong Ingress Controller to proxy requests in through a single loadbalancer, which is best practice.  Lastly, it enables BasicAuth RBAC for Kong Manager and Developer Portal.

## Prerequisites
1. AKS Credentials (App ID and Password)
2. Terraform CLI
3. Azure CLI
4. AKS Domain name

## Procedure

1. Open `/tf-provision-aks/aks-cluster.tf` to search & replace `simongreen` with your own name.  That way, all AKS objects will be tagged with your name making them easily searchable. Also, update the Azure region in this file to the region of your choice.
2. If you haven't done so already, create an Active Directory service principal account via the CLI:

 ```bash
 az login
 az ad sp create-for-rbac --skip-assignment`.  # This will give you the `appId` and `password` that Terraform requires to provision AKS.
 ```

3.  In `/tf-provision/aks` directory, create a file called `terraform.tfvars`.  Enter the following text, using your credentials from the previous command:

```bash
appId    = "******"
password = "******"
```

4. Search and replace 'simongreen' for a unique tag in the `/tf-provision-aks/aks-cluster.tf` file.
5. Via the CLI, `cd tf-provision-aks/` then run the following Terraform commands to provisions AKS:

```bash
terraform init
terraform apply
```

6. Once terraform has stoodup AKS, setup `kubectl` to point to your new AKS instance:

```bash
az aks get-credentials --resource-group $(terraform output -raw resource_group_name) --name $(terraform output -raw kubernetes_cluster_name)
kubectl get all
```

## Install Kong

1. Update license under ./license/license of this directory.
2. Update the following in values-dns.yaml file. I used an existing DNS record found via the Azure web interface, and created an Alias record set on the public IP that gets created through this process.

![](images/create-record-set.png "create-record-set")


Change with your hostname:

- Admin: https://kong-admin.CHANGE-ME.com
- Manager: https://kong-manager.CHANGE-ME.com
- Portal API: https://kong-portal-api.CHANGE-ME.com
- Portal: kong-portal.CHANGE-ME.com

It is best to do a search on CHANGE-ME.com and replace all occurances with your DNS name

### Install

1. Create namespace

`kubectl create namespace kong`

2. Create license secret

`kubectl create secret generic kong-enterprise-license -n kong --from-file=./license/license`

3. Create RBAC user:

`kubectl create secret generic kong-enterprise-superuser-password -n kong --from-literal=password=kong`

4. Create session configurations, Make sure to change "cookie_domain" in each:

```bash
echo '{"cookie_name":"admin_session","cookie_domain": ".CHANGE-ME.com","cookie_samesite":"off","secret":"password","cookie_secure":false,"storage":"kong"}' > admin_gui_session_conf

echo '{"cookie_name":"portal_session","cookie_domain": ".CHANGE-ME.com","cookie_samesite":"off","secret":"password","cookie_secure":false,"storage":"kong"}' > portal_session_conf
```

Using the following command to create the secrets

`kubectl create secret generic kong-session-config -n kong --from-file=admin_gui_session_conf --from-file=portal_session_conf`

5. Remove files:

`rm portal_session_conf admin_gui_session_conf`

6. Add kong charts repo:

`helm repo add kong https://charts.konghq.com`

and

`helm repo update`

7. Install kong:

`helm install kong kong/kong -n kong --values ./values/values-dns.yml`

if you wanted to upgrade:

`helm upgrade kong kong/kong -n kong --values ./values/values-dns.yml`

8. Monitor:

`kubectl get po -n kong -w `

### Post install:

1. Run `kubectl get ing -n kong`, it will return:

```NAME                  CLASS    HOSTS                                                      ADDRESS       PORTS     AGE
kong-kong-admin       <none>   kong-admin.kong-changeme.kong-sales-engineering.com        35.226.56.8   80, 443   91s
kong-kong-manager     <none>   kong-manager.kong-changeme.kong-sales-engineering.com      35.226.56.8   80, 443   91s
kong-kong-portal      <none>   kong-portal.kong-changeme.kong-sales-engineering.com       35.226.56.8   80, 443   91s
kong-kong-portalapi   <none>   kong-portal-api.kong-changeme.kong-sales-engineering.com   35.226.56.8   80, 443   91s
```

2. Update DNS in provider pointing to load balancer IP in your cloud account
