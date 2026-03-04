PS C:\Users\jho_j\Documents\studies\kubernetes\09-Ejemplo+Servicio+Deploy+Declarativo> docker build -t jhonajm/web-practice .

docker container run --name web1 -d -p 9090:80 jhonajm/web-practice:latest


PS C:\Users\jho_j\Documents\studies\kubernetes\09-Ejemplo+Servicio+Deploy+Declarativo> kubectl apply -f completo.yaml  
deployment.apps/web-d created
service/web-svc created

kubectl get endpoints web-svc -o yaml



