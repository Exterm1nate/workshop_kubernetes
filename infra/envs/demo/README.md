# Настройка Terraform для работы с Yandex Cloud

#### Настраиваем зеркала для terraform
```bash
☁ mv ~/.terraformrc ~/.terraformrc.old # Если у вас такой файл уже был
☁ touch ~/.terraformrc
☁ nano ~/.terraformrc
```

Добавляем блок
```tf
provider_installation {
  network_mirror {
    url = "https://terraform-mirror.yandexcloud.net/"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}

```
### Разворачиваем инфраструктуру

В Yandex Cloud создадим каталог demo, внутри своего облака.

Реинициализируем профиль yc
```bash
☁ yc init
# Прописываем токен, выбираем папку в yandex cloud, указываем Compute zone ru-central1-a
```

```bash
# Переходим в папку infra/envs/demo
☁ cd infra/envs/demo
```

Создаем сервисный аккаунт в yandex cloud с ролью admin

```bash
☁ sudo apt install -y jq
☁ FOLDER_NAME=demo2
☁ FOLDER_ID=$(yc config get folder-id)
☁ echo $FOLDER_ID
☁ CLOUD_ID=$(yc config get cloud-id)
☁ echo $CLOUD_ID

☁ SA_NAME=terraform-admin-sa-$FOLDER_ID
☁ echo $SA_NAME

☁ yc iam service-account create --name $SA_NAME

☁ SA_ID=$(yc iam service-account get --name $SA_NAME --format json | jq .id -r)
☁ echo $SA_ID


☁ yc resource-manager folder add-access-binding --id $FOLDER_ID \
--role admin \
--subject serviceAccount:$SA_ID
```

Создаем авторизованный ключ для сервисного аккаунта и запишем его в файл.

```bash
☁ yc iam key create \
--service-account-id $SA_ID \
--folder-name $FOLDER_NAME \
--output "${SA_NAME}-key.json"
```

Где:

`service-account-id` — идентификатор сервисного аккаунта.  
`folder-name` — имя каталога, в котором создан сервисный аккаунт.  
`output` — имя файла с авторизованным ключом.


Создаем профиль CLI для выполнения операций от имени сервисного аккаунта. Укажите имя профиля:

```bash
☁ yc config profile create sa-demo-terraform
```
Результат:

```bash
Profile 'sa-demo-terraform' created and activated
```

Проверяем:
```bash
☁ yc config profile list                                                       
default
sa-demo-terraform ACTIVE
```

Задайте конфигурацию профиля:

```bash
☁ yc config set service-account-key ${SA_NAME}-key.json
☁ yc config set cloud-id $CLOUD_ID
☁ yc config set folder-id $FOLDER_ID
```
Где:

`service-account-key` — файл с авторизованным ключом сервисного аккаунта.  
`cloud-id` — идентификатор облака.  
`folder-id` — идентификатор каталога.  
Проверить текущий конфиг профиля можно командой:
```bash
☁ yc config list
```


Пропишем переменные окружения для terraform.
```bash
☁ export YC_TOKEN=$(yc iam create-token)
☁ export YC_CLOUD_ID=$(yc config get cloud-id)
☁ export YC_FOLDER_ID=$(yc config get folder-id)
```
Примечание. Убедитесь что активный профиль sa-demo-terraform
```bash
☁ yc config profile list
default
sa-demo-terraform ACTIVE
```
Однако, токены можно указать вручную в providers.tf в секции `provider "yandex"`
```
provider "yandex" {
  token     = <yc_token>     # Токен для доступа к API Yandex Cloud (yc iam create-token)
  cloud_id  = <yc_cloud_id>  # Идентификатор вашего облака в Yandex Cloud
  folder_id = <yc_folder_id> # Идентификатор папки, в которой будут создаваться ресурсы
  ....
}
```
Пропишем каталог для terraform
```bash
☁ cp terraform.tfvars.example terraform.tfvars
```
И изменим переменную yc_folder_name на наше значение $FOLDER_NAME в файле `terraform.tfvars`

Запустим terraform
```bash
☁ cd ..
☁ cd infra/envs/demo
☁ terraform init
☁ terraform validate
☁ terraform plan
☁ terraform apply
```

### Настройка kubectl

Переключимся на основной профиль yc

```bash
☁ yc config profile activate default
```

Получим конфиги для kubectl
```bash
CLUSTER_NAME=k8s-demo-cluster 
☁ yc managed-kubernetes cluster get-credentials $CLUSTER_NAME --external --force

Context 'yc-k8s-demo-cluster' was added as default to kubeconfig '~/.kube//config'.
Check connection to cluster using 'kubectl cluster-info --kubeconfig ~/.kube/config'.

Note, that authentication depends on 'yc' and its config profile 'default'.
To access clusters using the Kubernetes API, please use Kubernetes Service Account.

```

Проверить конфиг можно с помощью команды:
```bash
☁ kubectl config view
```


### Удаление инфраструктуры

```bash
yc config profile activate default
REGISTRY_NAME=demo
for image in $(yc container image list --registry-name $REGISTRY_NAME --format json | jq -r '.[].id'); do
  yc container image delete $image
done
```

```bash
☁ yc config profile activate sa-demo-terraform
☁ export YC_TOKEN=$(yc iam create-token)
☁ export YC_CLOUD_ID=$(yc config get cloud-id)
☁ export YC_FOLDER_ID=$(yc config get folder-id)
☁ terraform destroy
```