
# Репозиторий для воркшопа "Kubernetes для Rails-разработчиков"


## Предварительные требования

- Учетная запись в Yandex Cloud.
- Установленный Terraform (версия 0.12 или выше).
- Инициализированный `yc` CLI (Yandex Cloud CLI).

Убедитесь что все утилиты работают корректно:
```bash
☁ yc --version
☁ terraform --version
☁ kubectl version --client
☁ docker --version
☁ docker run hello-world
```

## День 2

### Поднятие инфраструктуры

```bash
☁ cd infra/envs/day-2
```
Следуем инструкциям в [README.md](infra/envs/day-2/README.md), кроме шага про удаление инфраструктуры.

После сборки переходим обратно в корневой каталог репозитория.

### Сборка образа
```bash
☁ bin/build_image 0.0.1
export IMAGE=cr.yandex/crpihjptomdo51spscdp/hello-rails:0.0.1
bin/push_image $IMAGE
```

# Разворачиваение деплоймента

Ознакомимся c файлами манифестов

```bash
./k8s
├── config-rails-app.yaml - Файл с ConfigMap
├── deployment-rails-app.yaml - Файл c Deployment
├── job-rails-app-migrate.yaml - Файл с Job для миграций
├── secret-rails-app.yaml - Файл с Secret
└── service-rails-app.yaml - Файл с Service
```

Убедимся что IMAGE доступен как env переменная
```bash
☁ echo $IMAGE
cr.yandex/crpihjptomdo51spscdp/hello-rails:0.0.1
```

Выполним по шагам применение манифестов
```bash
☁ kubectl apply -f k8s/config-rails-app.yaml
☁ kubectl apply -f k8s/secret-rails-app.yaml
☁ envsubst < k8s/job-rails-app-migrate.yaml | kubectl apply -f -
☁ kubectl wait --for=condition=complete --timeout=600s job/job-rails-app-migrate
☁ envsubst < k8s/job-rails-app-migrate.yaml | kubectl delete -f -
☁ envsubst < k8s/deployment-rails-app.yaml | kubectl apply -f -
☁ kubectl apply -f k8s/service-rails-app.yaml
```

Проверим что приложение работает используя команды
```bash
☁ kubectl get deployments
☁ kubectl logs service/rails-app-service
☁ kubectl get pods
☁ kubectl logs <pod-name>
```

Зайдем в контейнер с нашим приложением и убедимся что наши ENV переменные пристутствуют
```bash
☁ kubectl get pods
☁ kubectl exec -it <pod-name> -- bash
☁ echo $RAILS_ENV
☁ echo $SECRET_KEY_BASE
☁ echo $FOO_KEY
```

Проверим работу приложения пробросив порт с сервиса на localhost

```bash
☁ kubectl port-forward service/rails-app-service 3000:3000
```

Откроем браузер по ссылке http://localhost:3000

# Удаление инфраструктуры
Переходим обратно в папку infra/envs/day-2
Выполняем шаги "Удаление инфраструктуры" из [README.md](infra/envs/day-2/README.md)

### Ссылки на ресурсы использованные в проекте

- https://kubernetes.io/docs/concepts/configuration/configmap/
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://kubernetes.io/docs/concepts/workloads/controllers/job/
- https://kubernetes.io/docs/concepts/configuration/secret/
- https://kubernetes.io/docs/concepts/services-networking/service/


# Ответ

```bash
# Вносим изменения в проект
# Билдим и пушим образ
# Прогоняем манифесты

kubectl get deployments
# NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
# deployment-rails-app   1/1     1            1           9s

kubectl logs service/rails-app-service
# => Booting Puma
# => Rails 7.1.3.4 application starting in production 
# => Run `bin/rails server --help` for more startup options
# W, [2024-06-22T16:46:25.852293 #1]  WARN -- : You are running SQLite in production, this is generally not recommended. You can disable this warning by setting "config.active_record.sqlite3_production_warning=false".
# Puma starting in single mode...
# * Puma version: 6.4.2 (ruby 3.3.1-p55) ("The Eagle of Durango")
# *  Min threads: 5
# *  Max threads: 5
# *  Environment: production
# *          PID: 1
# * Listening on http://0.0.0.0:3000
# Use Ctrl-C to stop

kubectl get pods
# NAME                                   READY   STATUS    RESTARTS   AGE
# deployment-rails-app-5fb9557df-7lz86   1/1     Running   0          111s

kubectl logs pod/deployment-rails-app-5fb9557df-7lz86=> Booting Puma
# => Rails 7.1.3.4 application starting in production 
# => Run `bin/rails server --help` for more startup options
# W, [2024-06-22T16:46:25.852293 #1]  WARN -- : You are running SQLite in production, this is generally not recommended. You can disable this warning by setting "config.active_record.sqlite3_production_warning=false".
# Puma starting in single mode...
# * Puma version: 6.4.2 (ruby 3.3.1-p55) ("The Eagle of Durango")
# *  Min threads: 5
# *  Max threads: 5
# *  Environment: production
# *          PID: 1
# * Listening on http://0.0.0.0:3000
# Use Ctrl-C to stop

kubectl port-forward service/rails-app-service 3000:3000
curl http://localhost:3000
# ...
# <h1>Привет со второго дня вебинара!</h1>
# ...

kubectl exec -it deployment-rails-app-5fb9557df-7lz86 -- bash
root@deployment-rails-app-5fb9557df-7lz86:/rails# echo $RAILS_MASTER_KEY
# c1d030b67adc991ac8149827454197ec

root@deployment-rails-app-5fb9557df-7lz86:/rails# bin/rails c
irb(main):001> puts Rails.application.credentials.config
# {:secret_key_base=>"190af7eece80ad4b4091d5e3c21db5927badddb886cc0a85797370a08d938839e896f9b2be1bd0b5ea2ab06263395216c449e121c739d801886dc47546e5c695"}

# Удаляем секреты и конфиги
# Применяем манифесты секретов и конфига
# Создаем джобу миграции и ждем ее:
envsubst < k8s/job-rails-app-migrate.yaml | kubectl apply -f -
kubectl wait --for=condition=complete --timeout=600s job/job-rails-app-migrate

# Ничего не происходит...
# Смотрим джобы и логи:
kubectl get jobs.batch 
# NAME                    COMPLETIONS   DURATION   AGE
# job-rails-app-migrate   0/1           8s         8s

kubectl logs jobs/job-rails-app-migrate
# Error from server (BadRequest): container "job-rails-app-migrate" in pod "job-rails-app-migrate-d88n8" is waiting to start: CreateContainerConfigError

# Т.к. джоба пересоздается, то она сразу подтягивает изменения в секретах и конфигах 
# и не может запуститься без необходимых данных

# Убиваем эту джобу
envsubst < k8s/job-rails-app-migrate.yaml | kubectl delete -f -

# Применяем деплоймент
envsubst < k8s/deployment-rails-app.yaml | kubectl apply -f -
# deployment.apps/deployment-rails-app configured

# Но в облаке вообще ничего не поменялось, крутится тот же самый деплоймент и под
kubectl get deployments
# NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
# deployment-rails-app   1/1     1            1           3h48m

kubectl get pods
# NAME                                   READY   STATUS    RESTARTS   AGE
# deployment-rails-app-5fb9557df-7lz86   1/1     Running   0          3h48m

# Слегка меняем манифест деплоймента, добавляя новые переменные и пытаемся применить манифест
envsubst < k8s/deployment-rails-app.yaml | kubectl apply -f -
# deployment.apps/deployment-rails-app configured

# И вот теперь все отвалилось:
kubectl get deployments
# NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
# deployment-rails-app   0/1     1            0           3h51m

kubectl get pods
# NAME                                    READY   STATUS   RESTARTS      AGE
# deployment-rails-app-6c7d77ff4c-n2drq   0/1     Error    4 (58s ago)   105s

# Возвращаем обратно необходимые переменные в манифесты конфига и секретов, применяем их
kubectl apply -f k8s/config-rails-app.yaml
kubectl apply -f k8s/secret-rails-app.yaml

# Применяем манифест деплоймента
envsubst < k8s/deployment-rails-app.yaml | kubectl apply -f -
# deployment.apps/deployment-rails-app configured

# И теперь все норм
kubectl get deployments
# NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
# deployment-rails-app   1/1     1            1           3h54m
kubectl get pods
# NAME                                    READY   STATUS    RESTARTS   AGE
# deployment-rails-app-6d77cd6b85-ssj8c   1/1     Running   0          8s

# Проверяем, что новые секреты и конфиги применились:
kubectl exec -it deployment-rails-app-6d77cd6b85-ssj8c -- bash
root@deployment-rails-app-6d77cd6b85-ssj8c:/rails# bin/rails c
irb(main):001> ENV["NEW_CONFIG"]
# => "this is not a secret"
irb(main):002> ENV["NEW_SECRET"]
# => "very secret!!"
```
