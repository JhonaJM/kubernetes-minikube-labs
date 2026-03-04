--intro a namespace
kubectl get namespace

kubectl get namespace default

kubectl describe namespace default

kubectl get pods -n default

--crear y borar namespaces
crear mediante comando
kubectl create namespace dev1
crear mediante yaml
kubectl apply -f namespace.yaml

kubectl delete namespace dev1

--crear objetos en un namespace

kubectl apply -f deploy_elastic.yaml --namespace=dev1
kubectl describe deploy elastic -n dev1

kubectl rollout history deploy elastic -n dev1

--establecer un namespace por defecto
kubectl config view
kubectl config set-context --current --namespace=dev1


--limite cpu y memorias en namespace
kubectl apply -f .\limitex.yaml

-- tambien se puede ver los namespace desde el panel, no recuerdo el comando para levantar el panel, ayudame con eso

--controlar eventos dentro de un namespace
kubectl get events --namespace dev1
kubectl get events --namespace dev1 --field-selector type="warning"
kubectl get events --namespace dev1 --field-selector reason="schedule"