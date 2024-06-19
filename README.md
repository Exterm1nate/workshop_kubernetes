Билдим и тестим локально:

```bash
cd app/dummy
export IMAGE_NAME=dummy-app
../bin/build_image 0.0.1

IMAGE=cr.yandex/crp5e24gs798lprtftgg/dummy-app:0.0.1
../bin/push_image $IMAGE

docker run -d -e RAILS_MASTER_KEY=bdec790de4ef182e6f1b619f8b06872f --name dummy -p 3000:3000 $IMAGE ./bin/rails server

# Update force_ssl = false

../bin/build_image 0.0.2

IMAGE=cr.yandex/crp5e24gs798lprtftgg/dummy-app:0.0.2
../bin/push_image $IMAGE

docker run -d -e RAILS_MASTER_KEY=bdec790de4ef182e6f1b619f8b06872f --name dummy -p 3000:3000 $IMAGE ./bin/rails server

curl http://localhost:3000
# ...
#   <h1>Привет с первого вебинара</h1>
# ...

docker stop dummy
docker rm dummy
```

Запускаем вручную в Кубе:

```bash
kubectl run dummy --image=$IMAGE --expose --port=3000 --env="RAILS_MASTER_KEY=bdec790de4ef182e6f1b619f8b06872f" --command -- ./bin/rails server

kubectl get pods
kubectl logs dummy
# All OK

kubectl port-forward pod/dummy 3000:3000

curl http://localhost:3000
# ...
#   <h1>Привет с первого вебинара</h1>
# ...

kubectl delete pod dummy
```

Запускаем через конфиги в Кубе:

```bash
kubectl apply -f k8s/rails-deployment.yaml
kubectl apply -f k8s/rails-service.yaml

kubectl get deployments
# NAME         READY   UP-TO-DATE   AVAILABLE   AGE
# demo-dummy   4/4     4            4           5m12s

kubectl get services
# NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
# dummy        ClusterIP   10.102.146.255   <none>        3000/TCP   16m
# dummy-svc    ClusterIP   10.102.200.236   <none>        3000/TCP   10m
# kubernetes   ClusterIP   10.102.128.1     <none>        443/TCP    83m

kubectl get pods
# NAME                         READY   STATUS    RESTARTS   AGE
# demo-dummy-8bc5f87f4-c5pm6   1/1     Running   0          11s
# demo-dummy-8bc5f87f4-l5tjj   1/1     Running   0          10s
# demo-dummy-8bc5f87f4-qwwpv   1/1     Running   0          12s
# demo-dummy-8bc5f87f4-v7bs4   1/1     Running   0          12s

kubectl logs demo-dummy-8bc5f87f4-c5pm6
# => Booting Puma
# => Rails 7.1.3.4 application starting in production 
# => Run `bin/rails server --help` for more startup options
# W, [2024-06-19T22:33:56.901030 #1]  WARN -- : You are running SQLite in production, this is generally not recommended. You can disable this warning by setting "config.active_record.sqlite3_production_warning=false".
# Puma starting in single mode...
# * Puma version: 6.4.2 (ruby 3.3.3-p89) ("The Eagle of Durango")
# *  Min threads: 5
# *  Max threads: 5
# *  Environment: production
# *          PID: 1
# * Listening on http://0.0.0.0:3000
# Use Ctrl-C to stop

kubectl port-forward service/dummy-svc 3000:3000
# Forwarding from 127.0.0.1:3000 -> 3000
# Forwarding from [::1]:3000 -> 3000
# Handling connection for 3000
# ...

curl http://localhost:3000
# ...
#   <h1>Привет с первого вебинара</h1>
# ...

kubectl logs demo-dummy-8bc5f87f4-c5pm6
# => Booting Puma
# => Rails 7.1.3.4 application starting in production 
# => Run `bin/rails server --help` for more startup options
# W, [2024-06-19T22:33:56.901030 #1]  WARN -- : You are running SQLite in production, this is generally not recommended. You can disable this warning by setting "config.active_record.sqlite3_production_warning=false".
# Puma starting in single mode...
# * Puma version: 6.4.2 (ruby 3.3.3-p89) ("The Eagle of Durango")
# *  Min threads: 5
# *  Max threads: 5
# *  Environment: production
# *          PID: 1
# * Listening on http://0.0.0.0:3000
# Use Ctrl-C to stop
# I, [2024-06-19T22:43:51.313099 #1]  INFO -- : [4d462dab-dd76-4e0e-9b67-9f680da152ff] Started GET "/" for 127.0.0.1 at 2024-06-19 22:43:51 +0000
# I, [2024-06-19T22:43:51.314035 #1]  INFO -- : [4d462dab-dd76-4e0e-9b67-9f680da152ff] Processing by HomeController#show as */*
# I, [2024-06-19T22:43:51.319922 #1]  INFO -- : [4d462dab-dd76-4e0e-9b67-9f680da152ff]   Rendered layout layouts/application.html.erb (Duration: 4.2ms | Allocations: 2366)
# I, [2024-06-19T22:43:51.320223 #1]  INFO -- : [4d462dab-dd76-4e0e-9b67-9f680da152ff] Completed 200 OK in 6ms (Views: 5.0ms | ActiveRecord: 0.0ms | Allocations: 3652)
```
