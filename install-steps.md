# Порядок работы

## Кластер

1. создаем кластер на 5 нод (2 под воркеры, 3 под цеф)
2. создаем три диска и монтируем их на ноды с цефом
3. подключаем кластер в линзу
4. прописываем на нодах с ceph тенанты:

```yaml
  taints:
    - key: ceph-nodes
      value: osd
      effect: NoExecute
```

## базовые настройки

0. через настройки кластера активируем метрики (все три галки), идем в ямлик датасета node-exporter и прописываем афинити и тенанты, что бы собирать метрики отовсюду

```yaml
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values:
                      - linux
              - matchExpressions:
                  - key: beta.kubernetes.io/os
                    operator: In
                    values:
                      - linux
              - matchExpressions:
                  - key: ceph-nodes
                    operator: In
                    values:
                      - osd
      schedulerName: default-scheduler
      tolerations:
      - key: ceph-nodes
        operator: Exists
        effect: NoExecute
```

1. создаем NS cert-manager и запускаем установку через Apps
2. не забываем: crd=true, ns=cert-mamager
3. создаем подписанта:

```yml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: orc-letsencrypt-issuer
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: vasyakrg@gmail.com
    # Name of a secret used to store the ACME account private key from step 3
    privateKeySecretRef:
      name: orc-letsencrypt-private-key
    # Enable the HTTP-01 challenge provider
    solvers:
    # An empty selector will 'match' all Certificate resources that
    # reference this Issuer.
    - selector: {}
      http01:
        ingress:
          class: nginx
```

4. создаем NS ingress-nginx и запускаем установку 4.0.1 версии
5. должен создасться loadbalancer с белым адресом

## Rook-ceph

1. устанавливаем оператор
2. устанавливаем кластер
3. просаживаем ингресс для панели (не забываем привязать ip-адрес балансера к доменному имени)

## Деплоим тестовое приложение

1. сначала выставляем в днс привязку с loadbalancer
2. запускаем деплой

## Операции с кластером ceph

### добавление OSD

1. добавляем в кластер k8s +1 и цепляем к ней диск на 30 гигов
2. прописываем на ноде тенант и лейбл `ceph-nodes: "osd"`
3. (не обязательно) видим, что оператор потыкал в нее и не успел найди второй прицепленный диск
4. перезапускаем оператор и ждем, что нода добавилась и стала osd-3

### удаление OSD

0. убеждаемся, что данные (реплики) залезут на оставшиеся ноды
1. выставляем реплику деплоймента оператора ceph в 0
2. помечаем в toolbox контейнере, что osd упала `ceph osd down osd.<ID>`
3. удаляем ноду из кластера k8s
4. помечаем в toolbox контейнере, что osd нужно убрать `ceph osd purge osd.<ID>`
5. возвращяем реплику деплоймента оператора ceph в 1
6. прибиваем ненужные деплойменты руками

### если прибили случайно ноду с монитором

1. удаляем через toolbox сломанный монитор `ceph mon remove <mon-id>`
2. оператор тут же запустит +1 реплику с id=+1 от старого на другой ноде (главное, что бы их хватило по кол-ву мониторов - минимум три)

3. возможные проблемы с warning и указанием на присутствие флагов in\out в хостах и crush

* ceph osd crush remove <hode_name>
* ceph crash archive-all

### очистка и замена OSD

1. запускаем purge-osd, указывая в нем какую (какие) osd надо обнулить
2. меняем диск на новый, оператор это видит и через какое-то время добавляет OSD в строй
