
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

## День 3

### Поднятие инфраструктуры

```bash
☁ cd infra/envs/demo
```
Следуем инструкциям в [README.md](infra/envs/demo/README.md), кроме шага про удаление инфраструктуры.

После сборки переходим обратно в корневой каталог репозитория.


### Github Actions

#### Секреты для использования github actions

Для проверки деплоя, рекомендую использовать `.github/workflows/deploy-app_v2.yaml`
Ниже, инструкция для переменных которые использует этот workflow.
Все эти переменные необходимо прописать в настройках репозитория `settings/secrets/actions` на github.


#### YC_CR_PUSHER_SA_JSON_CREDENTIALS

Создаем сервисный аккаунт для отправки docker образа в наш Container Registry.

```bash
☁ yc config profile activate default
☁ FOLDER_ID=$(yc config get folder-id)
☁ SA_NAME=cr-pusher-sa-$FOLDER_ID
☁ yc iam service-account create --name $SA_NAME
☁ SA_ID=$(yc iam service-account get --name $SA_NAME --format json | jq .id -r)

☁ yc resource-manager folder add-access-binding --id $FOLDER_ID \
--role container-registry.images.pusher \
--subject serviceAccount:$SA_ID

```
Создаем авторизованный ключ для сервисного аккаунта и запишем его в файл.

```bash
☁ yc iam key create \
--service-account-id $SA_ID \
--folder-id $FOLDER_ID \
--output "${SA_NAME}-key.json"
```
Содержимое этого файла запишем в ключ `YC_CR_PUSHER_SA_JSON_CREDENTIALS`.
Далее этот ключ будет использоваться для push dodker образов и push/pull helm пакетов в наш Container Registry.


#### CR_REGISTRY

```bash
CR_NAME=demo
REGISTRY_ID=$(yc container registry get --name "${CR_NAME}" --format json | jq -r .id)
```
Значение $REGISTRY_ID записываем в секреты github в ключ CR_REGISTRY.


#### CR_REPOSITORY
Запишем в секреты в github ключ CR_REPOSITORY c названием нашего образа, например demo-rails-app


#### YC_TOKEN
Запишем в секреты в github ключ YC_TOKEN c токеном который можно получить следующим образом:

```bash
yc config profile activate default
YC_TOKEN=$(yc config get token)
```

#### RAILS_APP_K8S_SECRET
Ключ RAILS_APP_K8S_SECRET
со значением
```yaml
secret:
  data:
    config.json: "ewogICJmb28iOiAiYmFyIgp9"
  stringData:
    secret_key_base: "123"
```

#### YC_CLOUD_ID
Ключ YC_CLOUD_ID со значением из:
```bash
yc config get cloud-id
```

#### YC_FOLDER_ID
Ключ YC_FOLDER_ID со значением из:
```bash
yc config get folder-id
```


### Работа с тегами
Создаем и пушим тег
```bash
VERSION=0.0.3
git tag v$VERSION
git push origin v$VERSION
```
либо
```bash
bin/make_and_push_tag 0.0.3
```

После пуша тега можно вручную запустить свой `workflow`.

Примечание. Желательно создавать сервис аккаунты отдельные для пуша докер образа и для пуша/пулла helm чарта.
Также желательно создавать сервис аккаунты для kubectl отдельно, внутри кластера.
В нашем workflow мы использовали только 1 сервис аккаунт для отправки образа в container registry. Для остальных задач использовали дефолтный профиль yc.