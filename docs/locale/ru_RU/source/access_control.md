# Access Control Lists (ACL)

## Что такое ACL?

*Note: Эта тема рассматривает access control и политики канала со стороны администратора.
Для информации о access control в чейнкоде, обратитесь к нашему туториалу: [чейнкод для разработчиков](./chaincode4ade.html#Chaincode_API).*

Fabric использует ACL (список контроля доступа), чтобы управлять доступ к ресурсам через
[Политики](policies/policies.html). Fabric содержит некоторый набор стандартных ACLs. В этой статье мы
разберем, в каком формате они задаются и как переопределить стандартные ACL.

Но прежде чем мы начнем, необходимо некоторое понимание ресурсов и политик.

### Ресурсы

Пользователи взаимодействуют с Fabric, используя [пользовательский чейнкод](./chaincode4ade.html),
или [источник потоков событий](./peer_event_services.html) или системный чейнкод.
Эти точки управления рассматриваются как "ресурсы", которые должны подчиняться некому порядку доступа, access control.

Разработчики приложений должны знать про эти ресурсы и связанные с ними стандартные политки.
Полный список этих ресурсов находится в `configtx.yaml`. Вы можете найти пример `configtx.yaml`
[здесь](http://github.com/hyperledger/fabric/blob/release-2.0/sampleconfig/configtx.yaml).

Ресурсы, перечисленные в `configtx.yaml` - полный перечень всех внутренних ресурсов, определенных в Fabric на текущий момент.
Принято перечислять их в формате `<component>/<resource>`.
Так, `cscc/GetConfigBlock` - это ресурс, отвечающий за вызов `GetConfigBlock` компонента `CSCC`.

### Политики

Политики - один из основных элементов Fabric, так как они позволяют identity (или набору
identities), сделавших запрос, быть проверенными по политикам, связанных с ресурсом, требуемом для осуществления запроса.
Политики подтверждения используются, чтобы определить, была ли транзакция соответственно подтверждениа.
Политики, определенные в конфигурации канала также упоминаются как политики изменения.

Политики могут быть созданы двумя способами: как политики `Signature` или как политики `ImplicitMeta`.

#### Политики `Signature`

Политики `Signature` указывают определенные типы пользователей, которые должны подписаться для удовлетворения политики.
Пример:

```
Policies:
  MyPolicy:
    Type: Signature
    Rule: "OR('Org1.peer', 'Org2.peer')"

```

Этот код может быть истолкован так: *политика с названием `MyPolicy`
может быть удовлетворена только подписью identity с ролью "пир организации Org1"
или "пир организации Org2"*.

Политики `Signature` поддерживают произвольные комбинации `AND`, `OR` и `NOutOf`,
позволяя конструировать крайне сложные правила на подобии "Администратор
организации A и два других администратора, или 11 из 20 администраторов".

#### Политики `ImplicitMeta`

Политики ImplicitMeta агрегируют в себе результат политик, расположенных глубже в дереве конфигурации.
Эти политики поддерживают правила на подобии "Большинство администраторов организаций". Они используют
другой, но простой синтаксис:
`<ALL|ANY|MAJORITY> <sub_policy>`.

Например: `ANY` `Readers` или `MAJORITY` `Admins`.

*Заметьте, что в стандартной конфигурации политик `Admins` - те, кто контролируют административные (операционные) аспекты работы сети.
Политики, указывающие, что только Admins --- или подмножество Admins --- имеют доступ
к ресурсу, будут, как правило, управлять важными или административными аспектами сети
(например, инстанцирование чейнкода в канале). `Writers` обычно имеют права на создание proposal'ов по обновлению реестра,
но не будут иметь административные разрешения. `Readers` имеют пассивную роль. Они могут только считывать информацию.
Эти стандартные политики могут быть изменены или дополнены, например, можно добавить роли `peer` и `client` (если
вы включили `NodeOU`).*

Пример политики `ImplicitMeta`:

```
Policies:
  AnotherPolicy:
    Type: ImplicitMeta
    Rule: "MAJORITY Admins"
```

В этом примере политика `AnotherPolicy` может быть удовлетворена `MAJORITY` (большинством) `Admins`,
где `Admins` будет в дальнейшем указано в более низкоуровневой политике `Signature`.

### Где указаны ACL?

Стандартные настройки контроля доступа (Access Control) находятся в `configtx.yaml`, файле, который `configtxgen`
использует для сборки конфигурации канала.

Контроль доступа может быть обновлен двумя способами: изменением `configtx.yaml`, что будет использоваться для создания конфигурации нового канала, или
обновлением контроля доступа в конфигурации существующего канала.

## Как заданы ACL в `configtx.yaml`

ACL задается как список пар ключ-значение, где ключ - это ресурс, а значение - политика доступа.
[Пример configtx.yaml](https://github.com/hyperledger/fabric/blob/release-2.0/sampleconfig/configtx.yaml).

Два фрагмента из этого примера:

```
# ACL policy for invoking chaincodes on peer (политика ACL вызова чейнкода на пирах)
peer/Propose: /Channel/Application/Writers
```

```
# ACL policy for sending block events (политика ACL для отправки событий, связанных с блоками)
event/Block: /Channel/Application/Readers
```

Эти ACL определяют доступ к ресурсам `peer/Propose` и `event/Block`: он дан только
identities, удовлетворяющим политикам с путями `/Channel/Application/Writers` и `/Channel/Application/Readers` соответственно.

### Обновление стандартных ACL в `configtx.yaml`

В случаях, когда необходимо переопределить стандартные ACL при создании сети или
изменить ACL до создания канала, лучше всего для этого отредактировать `configtx.yaml`.

Давайте предположим, что вы хотите изменить политику ресурса `peer/Propose` с
`/Channel/Application/Writers` на `MyPolicy`.

Сначала надо определить `MyPolicy` (название может быть любым). Она будет определена в секции
`Application.Policies` файла `configtx.yaml`. Для этого примера мы создадим политику `Signature`, указывающую на `SampleOrg.admin`.

```
Policies: &ApplicationDefaultPolicies
    Readers:
        Type: ImplicitMeta
        Rule: "ANY Readers"
    Writers:
        Type: ImplicitMeta
        Rule: "ANY Writers"
    Admins:
        Type: ImplicitMeta
        Rule: "MAJORITY Admins"
    MyPolicy:
        Type: Signature
        Rule: "OR('SampleOrg.admin')"
```

Далее, отредактируем секцию `Application: ACLs` `configtx.yaml`.
Вместо этой строчки:

`peer/Propose: /Channel/Application/Writers`

Напишите эту:

`peer/Propose: /Channel/Application/MyPolicy`

После того, как поля `configtx.yaml` были изменены, `configtxgen` учтет политики и ACL при создании
транзакции создания канала. После того, как транзакция подписана одним из администраторов участника консорциума,
будет создан новый канал с данными ACL и политиками.
Как только `MyPolicy` была указана в конфигурации, на нее можно ссылаться для переопределения других ACL:

```
SampleSingleMSPChannel:
    Consortium: SampleConsortium
    Application:
        <<: *ApplicationDefaults
        ACLs:
            <<: *ACLsDefault
            event/Block: /Channel/Application/MyPolicy
```

Это позволит подписываться на события, связанные с блоками (block events), только `SampleOrg.admin`.

Если канал уже был создан и хочет использовать это ACL, ему нужно будет обновить свою конфигурацию:

### Обновление стандартных ACL существующего канала

Для этого потребуется совершить транзакцию конфигурации канала.

*Note: Транзакции конфигурации канала вовлечены в процесс, подробно в этой статье не описанный. Для более подробной информации обратитесь к статьям
[Обновление конфигурации канала](./config_update.html) и [Туториал "Добавление организации в канал"](./channel_update_tutorial.html).*

После извлечения конфигурации из конфигурационного блока, отредактируйте ее, добавив
`MyPolicy` в `Application: policies`, где уже находятся политики `Admins`, `Writers` и `Readers`.

```
"MyPolicy": {
  "mod_policy": "Admins",
  "policy": {
    "type": 1,
    "value": {
      "identities": [
        {
          "principal": {
            "msp_identifier": "SampleOrg",
            "role": "ADMIN"
          },
          "principal_classification": "ROLE"
        }
      ],
      "rule": {
        "n_out_of": {
          "n": 1,
          "rules": [
            {
              "signed_by": 0
            }
          ]
        }
      },
      "version": 0
    }
  },
  "version": "0"
},
```

Обратите внимание на поля `msp_identifer` и `role`.

К секции ACL конфига измените ACL `peer/Propose` с этой строки:

```
"peer/Propose": {
  "policy_ref": "/Channel/Application/Writers"
```

На эту:

```
"peer/Propose": {
  "policy_ref": "/Channel/Application/MyPolicy"
```

Обратите внимание: Если в конфигурации канала отсутствовали ACL, вам придется добавить все структуру ACL.

Когда конфигурация была обновлена, потребуется утвердить ее согласно стандартному процессу обновления канала.

### Удовлетворение нескольких ACL к нескольким ресурсам сразу

Если участник совершает запрос, включающий несколько вызовов к разным частям системного чейнкода, все связанные с этим ACL
должны быть удовлетворены.

Например, `peer/Propose` контролирует доступ к созданию любого proposal-запроса на канале.
Если proposal требует также доступ к двум системным чейнкодам, один из которых требует identity, удовлетворяющей `Writers`, а другой - удовлетворяющей `MyPolicy`,
то, когда участник выдвигает proposal, он должен иметь identity, удовлетворяющую и
`Writers`, и `MyPolicy`.

В стандартной конфигурации, `Writers` - политика `Signature`, правило которой - быть `SampleOrg.member`, иными словами просто быть любым участником SampleOrg.
Правило `MyPolicy` - быть администратором SampleOrg. Соответственно, чтобы удовлетворить их обеих, необходимо быть администратором SampleOrg.
Поэтому надо быть следить, чтобы ACL для proposal'ов можно было удовлетворить и чтобы ACL для propoal'а не противоречили друг другу.

<!--- Licensed under Creative Commons Attribution 4.0 International License
https://creativecommons.org/licenses/by/4.0/ -->
