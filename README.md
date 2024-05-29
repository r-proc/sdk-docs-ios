# Kd Pay Sdk Ios

IOS SDK for KD_pay

Для интеграции SDK в приложение необходимо сохранить сборку данного фреймворка. Чтобы получить сборку при первой установке, либо после внесения изменений, необходимо в терминале перейти в папку  SDK и зупстить команду sh build.sh, для того чтобы запустить работу скрипта, запускающего сборку данного фреймворка.
После чего, в папке build появится папка Processing.xcframework

Для того, чтобы этот файл корректно работал в проекте после внесенных изменений, необходимо сделать следующее:
1.  Добавить/заменить (показать в Finder) папку Processing.xcframework в корневой папке приложения
2.  Необходимо добавить Processing в Frameworks, Libraries, and Embedded Content и выбрать Embed Without Signing.
3.  import Processing

# Сценарии использования SDK KD Pay

### Инициализация sdk
Для инициализации Processing необходимо вызвать Processing.configure с  **обзязательными** параметрами:
`buyerId` установливает идентификатор магазина.
`appVersion` устанавливает версию вашего приложения.

Ниже перечислены необязательные параметры:
`screenSize` устанавливает размер, разрешение вашего экрана. Используется в запросах к беку. В зависимости от него для некоторых устройств могут приходить иные картинки, тексты.
`fcmToken` - токен Firebase Messaging Service [подробнее](https://firebase.google.com/docs/cloud-messaging/ios/client?hl=ru). 
`deviceId`- идентификатор устройства в системе.
 
```
Processing.configure(
   buyerId: buyerId,
   appVersion: appVersion,
   screenSize: screenSize,
   fcmToken: AppUserDefaultsManager.firebaseRegistrationToken,
   deviceId: deviceId)
```

Также при запуске приложения можно вызвать следующие методы и утсановить свойства для получения/установления данных:

```
public class func setRetryParams(retryCount: Int = 3, retryDelay: Double = 0.1)
```
Бета или прод сервер, prod если false, иначе true. По умолчанию бета = true. Используются разные keychain для беты и прод, поэтому invalidate на одном не стирает данные для другого. Также отличается baseHost в url запросах.
```
public static var beta: Bool = true 
```

Проверка на авторизованность
```
var isAuthorized: Bool {
    processingEntry.isAuthorized
}
```
Получение базовой версии для API, используется в запросах, например, "/v2/" сразу после baseHost (
```
public class func getBaseVerion() -> String {
    processingEntry.baseVersion
}
```  
Получение версии SDK,  используется в url запросах в Header с ключом X-Version
```
public class func getSdkVersion() -> String {
    sdkVersion
}
```
Обновление токена пользователя на сервере.
```
public class func updateFcmToken(_ token: String?) {
   processingEntry.fcmToken = token
}
```
Проверка доступа к кошелькам. Имеет кеширование. Возвращает результат предыдущего запроса, параллельно выполняя новый запрос. Если интервал не задан, кеш считается валидным 1 час
```
public class func getUserExperiment(phone: String,
                                    deviceId: String,
                                    loyaltyId: String,
                                    cacheInterval: TimeInterval? = nil,
                                    completion: @escaping (Result<[ProcessingUserExperiments]>) -> Void) 
```
Проверка статуса sdk, т.е. его версии и возможности использования с этой версией. Проверяется текущая версия sdk.
```
public class func checkLibraryVersion(cacheInterval: TimeInterval, completion: @escaping (Result<ProcessingVersionInfo>) -> Void)
```
Получение состояние аккаунта в системе KD Pay. Имеется информация о баннерах, story, привязанных банках, которые можно использовать до входа пользователя в KD Pay.
```
public class func getUserPreview(source: String, manzanaUserId: String, loyaltyId: String, completion: @escaping (Result<UserPreviewResponseData>) -> Void)
```

### Авторизация пользователя в системе KD Pay
                                
Авторизация пользователя проходит в два этапа, для этого необходимо запросить у пользователя:
   - userPhone: номер телефона, указанный пользователем (обязательное поле, на этот номер будет создан кошелёк),
   - userBasePhone: дополнительный номер телефона в основном приложении (необязательное),
   - userName: имя (необязательное),
   - uid: идентификатор устройства (если не передать, СДК попытается подставить автоматически)
   - loyaltyId: идентификатор карты лояльности (необязательный). 
        
После чего указанные данные передаются в метод `login`. Сервер в ответ на запрос посылает на номер пользователя шестизначный код, который вместе с `fcm` токеном нужно передать в метод `confirmCode`. При успешном ответе от сервера sdk сохраняет в кеше необходимые данные (публичный ключ сервера, идентификаторы сессии и пользователя). После чего пользователь считается авторизованным
```
func login(userPhone: String, userBasePhone: String?, userName: String? = nil, uid: String?, loyaltyId: String?, device: Device, completion: ((Result<Int>) -> Void)? = nil) {
        processingEntry.login(phone: userPhone, basePhone: userBasePhone, deviceId: uid, name: userName, loyaltyId: loyaltyId, device: device, completion: completion)
    }
```
Подтверждение регистрации пользователя с помощью ОТП кода (СМС)
```
func confirmCode(code: String, fcmToken: String, referrerId: String?, completion: ((Result<UserResponse?>) -> Void)?) {
        processingEntry.userConfirm(fcmToken: fcmToken, phoneCode: code, referrerId: referrerId, completion: completion)
    }

```
<img src="https://github.com/r-proc/sdk-docs-ios/assets/93093046/80b1b2e9-6523-4dd3-921d-abfe51e441ea" width="262" height="568">


### Ссылки на связанные документы:
Согласие на обработку персональных и биометрических данных и пользовательское соглашение -
https://storage.yandexcloud.net/slyanov-s3/gdpr_consent_v4.html

Публичная оферта - 
https://storage.yandexcloud.net/slyanov-s3/wallet_public_offer24.html

### Публичный ключ для генерации QR-кода

SDK получает публичный ключ и сохраняет его в кеше после авторизации пользователя (метод `confirmCode`). Этот публичный ключ хранится в кеше и считается валидным, пока в заголовке любого метода от сервера не вернется `Need-Update-Server-key: true`, после чего SDK  асинхронно дернет ручку обновления публичного ключа и при успешном запросе значение в кеше будет заменено на новый публичный ключ. Это происходит незаметно для пользователя и весь процесс инкапсулирован внути SDK. Пользователь SDK извне никак не может повлиять на этот процесс.

### Получение аккаунта пользователя

Получение аккаунта осуществляется с помощью метода `getAccount`. Для успешного получения состояния кошелька пользователь должен быть авторизован, должен подтвердить смс и сдк . Если клиент не авторизован в системе метод выбрасывает ошибку.
```
public class func getAccount(cached: Bool, referrerId: String? = nil, completion: @escaping (Result<UserResponse?>) -> Void) {
    processingEntry.account(cached: cached, referrerId: referrerId, completion: completion)}
```   

В UserResponse в поле bankLinkType приходит тип привязки банков : рекуррент/me2me. 

<img src="https://github.com/r-proc/sdk-docs-ios/assets/93093046/412383e9-db78-4de3-841d-d2e282e9881b"  width="262" height="568">

### Onboarding
В ответе UserResponse приходит  поле required_screens с массивом форм/экранов, которые можно показать перед отображением кошелька, например "need_bank_link"

<img src="https://github.com/r-proc/sdk-docs-ios/assets/93093046/9b37415d-e85a-462f-8274-ceb63fbb2e38"  width="262" height="568">

### Привязка банка к кошельку
Для того чтобы привязать к кошельку пользователя банк, необходимо получить список банков с помощью метода `getBanks`. Для того чтобы использовать метод пользователь должен быть авторизован и у пользователя должен быть создан аккаунт. В противном случае метод вызовет ошибку.
```
public class func getBanks(completion: @escaping (Result<[ProcessingBank]>) -> Void) {
        processingEntry.accountBanks(completion: completion)
    }
```
В ответе метода `getAccount` есть информация о типе платежного сервиса (bankLinkType): рукуррент/me2me. Метод `getBanks` в SDK учитывает это информацию и возвращает список банков в зависимости от данного типа.

<img src="https://github.com/r-proc/sdk-docs-ios/assets/93093046/7ca7b14c-0706-462f-8d9a-6d745614de0a"  width="262" height="568">

Далее из списка полученных банков пользователь выбирает необходимый. После чего на сервер отправляется запрос на привязку банка с помощью метода `addNewBank`.

```
public class func addNewBank(bankId: Int, fundingSum: String?, completion: @escaping (Result<AccountBankLinkResponse>) -> Void) {
        processingEntry.accountBankLink(bankId: bankId, fundingSum: fundingSum, completion: completion)
    }
```  
Если сервер ответил успехом, то пользователю предлагается перейти в мобильное приложение банка и завершить процесс привязки банка.

Возвращается AccountBankLinkResponse с id привязки банка, ссылкой с кастомной схемой выбранного банка "bankID://sub.nspk.ru/..."  и временем ожидания подписки expireTimeout.

Данную ссылку необходимо открыть в МП для перехода в мобильное приложение банка, если оно установлено на девайсе.

 <img src="https://github.com/r-proc/sdk-docs-ios/assets/93093046/db8dc338-42d9-4ed3-b475-4dfecb73f546"  width="262" height="568">


### Проверка статуса привязки банка к аккаунту пользователя
Так как привязка банка осуществляется пользователем в стороннем мобильном приложении, то при вызове метода `addNewBank` не возвращается результат привязки банка. 

Для того, чтобы проверить статус в течение времени ожидания подписки (expireTimeout), нужно запросить метод `getBankLinkStatus`,  где в ответе возвращается статус и finalStoryId.

```
public class func getBankLinkStatus(completion: @escaping (Result<AccountBankLinkStatus>) -> Void) {
        processingEntry.getBankLinkStatus(completion: completion)
    }
```
Статус привязки банка:
pending - ожидание привязки банка в приложении банка, 
success - банк привязан успешно

<img src="https://github.com/r-proc/sdk-docs-ios/assets/93093046/f5906d61-72f6-47ec-9fa8-c4c110f6b759"  width="262" height="568"> 

По полученному finalStoryId отображаются экраны со story (Информация об оплате в кошелька с помощью привязанного банка)

<img src="https://github.com/r-proc/sdk-docs-ios/assets/93093046/59ab130b-65f0-42b5-aee9-867b747d38b5"  width="262" height="568"> 
<img src="https://github.com/r-proc/sdk-docs-ios/assets/93093046/1f324f42-56ac-4c5f-8cd5-d89f2d94ba01"  width="262" height="568">
<img src="https://github.com/r-proc/sdk-docs-ios/assets/93093046/41167685-8025-4389-b864-77b22084d350"  width="262" height="568"> 

### Получение привязанных банков

При возврате на главный экран кошелька и вызове метода `getAccount` в поле banks возвращаются банки, которые успешно привзязаны, либо которые в процессе привязки.

Ориентируясь на список привязанных банков и id банка, который привязывается к аккаунту можно понять в каком статусе находится привязка банка. Если банк присутствует списке привязанных банков, но поле `isLinked` у банка равно `false` - это значит что банк в процессе привязки к кошельку. В случае если поле равно `true`, значит процесс привязки банка успешен. В случае если в списке банков нет искомого, то это означает что произошла ошибка при привязке банка.

<img src="https://github.com/r-proc/sdk-docs-ios/assets/93093046/2a336778-d9f6-458d-80b0-22ef606e859e"  width="262" height="568"> 

Если в методе `getAccount` возвращается bankLinkType - recurrecnt, то список привязанных банков находится в массиве recurrentBanks, если me2me, то в account в поле banks

### Установить банк по умолчанию

После привязки банка он становится банком по умолчанию. Для того чтобы изменить банк по умолчанию следует воспользоваться методом `setDefaultBank`. Для того чтобы использовать метод пользователь должен быть авторизован и у пользователя должен быть создан аккаунт, а так же привязан как минимум один банк. Для начала следует получить данные аккаунта методом `getAccount`, предложить пользователю выбрать из списка привязанных банков тот, который он хочет сделать основным. Далее передать `id` этого банка в метод `setDefaultBank(bankId)`. В результате чего в случае успеха метод вернет измененный и актуальный список привязанных банков.

```
public class func setDefaultBank(bankId: Int, completion: @escaping (Result<[ProcessingBank]>) -> Void) {
        processingEntry.accountBank(id: bankId, completion: completion)
}
```
<img src="https://github.com/r-proc/sdk-docs-ios/assets/93093046/fa65d106-97bb-47bd-a3f9-73c23897c568"  width="262" height="568"> 

### Генерация qr кода

Для быстрого создания строки, кодируемой в qr-код, достаточно использовать метод `generateQRString`.
Для использования метода пользователь должен быть предварительно авторизован и иметь аккаунт в системе.
 
 Метод setNumberOfCodes устанавливает кол-во ОТП кодов, по умолчанию их  10
 
```
public class func generateQRString(loyaltyId: String, completion: @escaping (Result<String>) -> Void) {
    processingEntry.generateQRString(idLoyalty: loyaltyId, completion: completion)
}
    
public class func setNumberOfCodes(number: Int) {
    processingEntry.numberOfCodes = number
}
```
<img src="https://github.com/r-proc/sdk-docs-ios/assets/93093046/5cec424e-1887-4447-9c54-b818e7cfd394"  width="262" height="568"> 


### Получение формы
Для платежного сервиса в сценариях me2me необходимы формы для получения информации о пользователе. Формы динамические, то есть содержимое (состав полей) меняется в зависимости от контекста (сценария). Например, для привязывания счета необходимо заполнить форму упрощенной идентификации.
Получение формы происходит через метод `getForm(type: String?, requestId: Int?, version: Int?)`. Если запрос формы произойдет без `requestId`, то сервер пришлет чистую форму.
Если запрос происходит с `requestId`, то SDK отправит 2 запроса на сервер:  `/form` и `/request`. Первый запрос получит форму, второй получит статус валидации для полей. SDK соединит полученные результаты и вернет форму с провалидированными полями.

*Примечание: количество полей и их состав сервер может поменять на своей стороне, форма является динамической.*

```
public class func getForm(type: String?, requestId: String?, version: String?, completion: @escaping (Result<ProcessingForm>) -> Void) {
    processingEntry.userForm(type: type, requestId: requestId, version: version, completion: completion)
}

```

### Валидация полей формы
Для валидации полей используется метод `sendFormDraft(type: String, formId: Int?, version: Int?, draftFields: Dictionary<Int, String>)`. Первый запрос на валидацию всегда будет происходить без `formId` и `version`. Актуальные значения `formId` и `version` вернутся при первой и последующей валидации. Валидировать необходимо все заполненные поля.

*Примечание: сервер сохраняет в следующем драфте поля "вместо" предыдущих. То есть если в первой версии драфта МП отправило заполненное поле 1, а потом не прислало это поле или прислало пустое, то в этой версии драфта поле не будет сохранено на сервере.*

При `sendFormDraft(type: String, formId: Int?, version: Int?, draftFields: Dictionary<Int, String>)` SDK уберет пустые поля из запроса, чтобы не заваливать пользователя ошибками на пустых полях.

```
public class func sendFormDraft(type: String, formId: Int?, version: Int?, draftFields: [Int: String], completion: @escaping (Result<ProcessingFormDraft>) -> Void) {
    processingEntry.userFormDraft(fields: draftFields, type: type, requestId: formId, version: version, completion: completion)
}
```


### Отправка формы на проверку
Для валидации полей используется метод `sendFormCommit(type: String, formId: Int?, version: Int?, draftFields: Dictionary<Int, String>)`. Первый запрос на `commit` может происходить без `formId` и `version`. Актуальные значения `formId` и `version` вернутся при первом и последующем `commit`.
Отправлять необходимо все поля.

При `sendFormCommit(type: String, formId: Int?, version: Int?, draftFields: Dictionary<Int, String>)` сервер отправит **все** поля, даже пустые, чтобы уведомить пользователя о незаполненных полях.

```
public class func sendFormCommit(type: String, formId: Int, version: Int?, draftFields: [Int: String], completion: @escaping (Result<ProcessingFormDraft>) -> Void) {
    processingEntry.userFormCommit(fields: draftFields, type: type, requestId: formId, version: version, completion: completion)
}

```

### Получение истории операции для пользователя
Получить историю операций для пользователя можно с помощью метода `getOperations`. В данный метод заложены параметры для фильтрации операций. Бек вернет список операций, удовлетворяющих параметрам фильтрации.
Для первого запроса истории операции не нужно передавать курсор, в последующих запросах необходимо передавать курсор. Бек вернет курсор в ответе на первый запрос, если пользователь имеет больше операций, чем бек может отдать за один запрос.
Курсор (`cursor`) используется для пагинации операций.

```
public class func getOperations(
    cursor: String?,
    type: OperationElement.OperationType?,
    partner: String?,
    datetimeStart: String?,
    datetimeFinish: String?,
    status: OperationElement.Status?,
    completion: @escaping (Result<Operations>) -> Void) {
    processingEntry.getOperations(
        cursor: cursor,
        type: type,
        partner: partner,
        datetimeStart: datetimeStart,
        datetimeFinish: datetimeFinish,
        status: status,
        completion: completion)
}
```

<img src="https://github.com/r-proc/sdk-docs-ios/assets/93093046/9318f10a-0df7-45e0-86a3-1b0a3586ca06"  width="262" height="568"> 


### Получение конкретной операции для пользователя
Получить операцию для пользователя можно через метод `getOperationInfo`. Важно, что id операции должен совпадать с типом операции, иначе операция будет считаться на найденной.

```
public class func getOperationInfo(id: Int, type: OperationElement.OperationType, completion: @escaping (Result<OperationElement>) -> Void) {
  processingEntry.getOperationInfo(id: id, type: type, completion: completion)
}
```

# Методы

## GENERAL LIBRARY FUNCTIONS

##### Конфигурация Processing SDK.
```
public class func configure(
        buyerId: String,
        appVersion: String,
        screenSize: String?,
        fcmToken: String?,
        deviceId: String?) 
```
**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| buyerId| String| нет | Идентификатор пользователя|
| appVersion | String| нет | Версия приложения |
| screenSize | String| да | Pазмер, разрешение экрана. Используется в запросах к беку. В зависимости от него для некоторых устройств могут приходить иные картинки, тексты.|
| fcmToken| String| да | Токен Firebase Messaging Service [подробнее](https://firebase.google.com/docs/cloud-messaging/ios/client?hl=ru). |
| deviceId | String| да | Идентификатор устройства в системе.|

##### Версия SDK.
`func getSdkVersion() -> String`

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| String      | Нет       | Вернет строку с версией SDK|


##### Обновление токена пользователя на сервере.
`func updateFcmToken(_ token: String?) `

**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| token| String| да | Токен Firebase Messaging Service [подробнее](https://firebase.google.com/docs/cloud-messaging/ios/client?hl=ru). |


##### Метод для проверки состояния сервиса.
`func ping(completion: ((Result<Bool>) -> Void)? = nil)`

**Сompletion**

| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
|(Result\<Bool\>) -> Void| да | Коллбэк с результатом пинга. В  .success передается true, если статус ответа == "ok" или false в противном случае. |


##### Проверка статуса sdk, т.е. его версии и возможности использования с этой версией. Проверяется текущая версия sdk.
`func checkLibraryVersion(cacheInterval: TimeInterval, completion: @escaping (Result<ProcessingVersionInfo>) -> Void)`

**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| cacheInterval| TimeInterval| нет |Время кэширования статуса процессинга в секундах. 0 - нет кэширования |

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<ProcessingVersionInfo\>) -> Void| нет| callback с информацией о версии|

##### Первый шаг авторизации пользователя в системе KD Pay. Служит для запроса смс кода на указанный в параметрах номер телефона.
`func login(userPhone: String, userBasePhone: String?, userName: String? = nil, uid: String?, loyaltyId: String?, device: Device, completion: ((Result<Int>) -> Void)? = nil)`

**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| userPhone| String| нет | Номер телефона, указанный пользователем |
| userBasePhone | String| да | Телефон пользователя в основном приложении |
| userName| String| да | Имя пользователя |
| uid| String| да |UUID устройства |
| loyaltyId | String| да | Номер карты лояльности |


**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<Int\>) -> Void)| Да| Сallback с идентификатором пользователя|


##### Второй шаг авторизации пользователя в системе KD Pay. Служит для подтверждения регистрации пользователя с помощью ОТП кода (СМС) и завершения авторизации.
`func confirmCode(code: String, fcmToken: String, referrerId: String?, completion: ((Result<UserResponse?>) -> Void)?)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| code| String| нет | Смс (ОТП) код, полученный на указанный в методе login номер телефона |
| fcmToken| String| нет | Токен Firebase Messaging Service [подробнее](https://firebase.google.com/docs/cloud-messaging/ios/client?hl=ru). |
| referrerId| String| да | Идентификатор реферальной системы |

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<UserResponse?\>) -> Void| Да| Сallback с аккаунтом пользователя в системе и текущий статус аккаунта|

##### Проверка на авторизованность. Пользователь авторизован, если у него есть userId, sessionId и serverPublicKey. Нужно учитывать, что эта проверка локальная и не может определить актуальность сессии. Ее определит любой запрос на бэк (c userId) и вернет ошибку .notAithorized
`isAuthorized: Bool`

**Возвращает**

| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| Bool| Нет| true - пользователь авторизован, false - не авторизован (Нет идентификатора сессии, пользователя, либо публичного ключа сервера в кеше SDK)|

##### Проверка доступности кошельков для пользователя. Имеет кеширование.
```
getUserExperiment(phone: String,
                  deviceId: String,
                  loyaltyId: String,
                  cacheInterval: TimeInterval? = nil,
                  completion: @escaping (Result<[ProcessingUserExperiments]>) -> Void)`
```
**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| phone| String| нет | Номер телефона пользователя |
| deviceId | String| нет | Идентификатор устройства в системе. |
| loyaltyId | String| нет | Номер карты лояльности |
| cacheInterval| TimeInterval| да | Интервал обновления кэша значений эксперимента в секундах, т.е. кэш будет пытаться обновиться через cacheInterval секунд. Если не передавать значение, то по-умолчанию 1 час. |

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| Result\<[ProcessingUserExperiments]\>) -> Void| Нет|Сallback с массивом экспериментов для данного пользователя, isAvailable == true - доступ есть, false - доступа нет|

##### Выход и удаление всех данных пользователя.
`func logout(completion: ((Result<Void>) -> Void)? = nil)`

**Сompletion**
| Тип                     | Опциональный | Описание |
| ----------------------- | ----------- | ----------- |
| (Result\<Void\>) -> Void| Да| Сallback успешного / ошибочного запроса|

## USERS
##### Получение аккаунта пользователя в системе кошелька. Пользователь должен быть авторизован, должен подтвердить смс и сдк должно получить всю необходимую информацию с сервера, иначе будет ошибка.
`func getAccount(cached: Bool, referrerId: String? = nil, completion: @escaping (Result<UserResponse?>) -> Void)`

**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| cached| Bool| нет | Флаг для определения откуда брать данные аккаунта. true - из кеша, false - из сервера с последующим кешированием.|
| referrerId| String| да | Идентификатор реферальной системы |


**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<UserResponse?\>) -> Void | нет| Сallback, содержащий аккаунт пользователя в системе |


##### Получение состояния аккаунта в системе KD Pay. Имеется информация о баннерах, story, привязанных банках, которые можно использовать до входа пользователя в KD Pay.
`func getUserPreview(source: String, manzanaUserId: String, loyaltyId: String, completion: @escaping (Result<UserPreviewResponseData>) -> Void)`

**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| source| String| нет | Источник перехода|
| manzanaUserId| String| нет | Идентификатор пользователя |
| loyaltyId | String| нет | Номер карты лояльности |

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<UserPreviewResponseData\>) -> Void| Нет| Сallback, содержащий аккаунт пользователя в системе |

##### Возвращает информацию о пользователе для Профиля.
`func getUserInfo(completion: @escaping (Result<UserInfoResponse>) -> Void)`

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<UserInfoResponse\>) -> Void  | Нет       | Данные по профилю пользователя |

##### Возвращает информацию о пользователе для Профиля (детализация).
`func getUserInfoDetails(completion: @escaping (Result<UserInfoDetailsResponse>) -> Void)`

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<UserInfoDetailsResponse\>) -> Void | Нет       |Сallback c данными по профилю пользователя (детализация)|

### BANKS

##### Получение информации фондирования (для платежного сервиса me2me).
`func getUserFunding(completion: @escaping (Result<UserFundingResponse>) -> Void)`

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<UserFundingResponse\>) -> Void | Нет| Сallback с информацией для фондирования:  сумма, кнопки c суммами, баннеры    |


##### Получение списка банков, которые можно привязать к кошельку.
`func getBanks(completion: @escaping (Result<[ProcessingBank]>) -> Void)`


**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| Result\<[ProcessingBank]\>) -> Void | Нет| Сallback со списком банков, которые можно привязать к кошельку. |

##### Метод для установки банка по умолчанию. Выбирается из списка привязанных банков.
`func setDefaultBank(bankId: Int, completion: @escaping (Result<[ProcessingBank]>) -> Void)`

**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| bankId| Int| нет |Идентификатор банка в системе KD Pay, выбранных из списка банков получаемых с помощью метода getBanks()|


**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<[ProcessingBank]\>) -> Void| Нет| Сallback со списком привязанных банков, с измененным банком по умолчанию. |


##### Метод для добавления банка к кошельку. Выбирается из списка привязанных банков.
`func addNewBank(bankId: Int, fundingSum: String?, completion: @escaping (Result<AccountBankLinkResponse>) -> Void)`

**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| bankId| Int| нет |Идентификатор банка в системе KD Pay, выбранных из списка банков получаемых с помощью метода getBanks()|
| fundingSum| String| да |В случае me2me сумма пополнения|

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<AccountBankLinkResponse\>) -> Void| Нет| Сallback с результатом добавления  банка к кошельку |

##### Метод для плучения статуса банковской связи.
`func getBankLinkStatus(completion: @escaping (Result<AccountBankLinkStatus>) -> Void)`

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<AccountBankLinkStatus\>) -> Void)| Нет|Сallback с результатом привязки банка. |


##### Метод для удаления привязанного банка
`func deleteBank(bankId: Int, completion: @escaping (Result<Void>) -> Void)`

**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| bankId| Int| нет |Идентификатор банка в системе KD Pay, выбранных из списка банков получаемых с помощью метода getBanks()|


**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
|(Result\<Void\>) -> Void)| Нет| Сallback с результатом удаления банка. |


## QR CODES GENERATION

##### Метод формирует строку, которую впоследствии можно закодировать в qr код для оплаты на кассе. 
`func generateQRString(loyaltyId: String, completion: @escaping (Result<String>) -> Void)`
**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| loyaltyId| String| нет | Идентификатор лояльности во внешней системе |


**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<String\>) -> Void) | Нет| Сallback со строкой для кодирования в Qr код|

##### Устанавливает количество OTP кодов для QR кода, по умолчанию - 10.
`func setNumberOfCodes(number: Int) `
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| number| Int| да | количество OTP кодов для QR кода |


## FORMS & VALIDATION

##### Метод получения формы упрощенной идентификации УПРИД
`func getForm(type: String?, requestId: String?, version: String?, completion: @escaping (Result<ProcessingForm>) -> Void)`

**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| type| String| да | Тип формы, который задает определенный набор полей (приходит в account). |
| requestId| String| да | Идентификатор последнего актуального заполнения (id драфта формы). |
| version| String| да | Версия последнего актуального заполнения |

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<ProcessingForm\>) -> Void) | Нет|Сallback с формой для упрощенной идентификации пользователя |


##### Метод для отправки черновика формы
`func sendFormDraft(type: String, formId: Int?, version: Int?, draftFields: [Int: String], completion: @escaping (Result<ProcessingFormDraft>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| type| String| нет | Тип формы, который задает определенный набор полей. |
| formId| Int| да | Идентификатор последнего актуального заполнения(id драфта формы). |
| version| Int | да | Версия последнего актуального заполнения. |
| draftFields|  Dictionary\<Int: String\> | нет | Идентификатор валидируемого поля и его значение. |


**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<ProcessingFormDraft\>) -> Void) | Нет|Сallback с актуальной информацией формы и список невалидных полей |


##### Метод для коммита формы (окончательное подтверждение).
`func sendFormCommit(type: String, formId: Int, version: Int?, draftFields: [Int: String], completion: @escaping (Result<ProcessingFormDraft>) -> Void)`

**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| type| String| нет | Тип формы, который задает определенный набор полей. |
| formId| Int| нет | Идентификатор последнего актуального заполнения(id драфта формы). |
| version| Int | да | Версия последнего актуального заполнения.|
| draftFields| Dictionary\<Int: String\>  | нет | Идентификатор валидируемого поля и его значение. |

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<ProcessingFormDraft\>) -> Void | Нет| Сallback с актуальной информацией формы и список невалидных полей |


##### Метод для подсказки для полей формы
`func getFormSuggestions(fieldId: Int, value: String, completion: @escaping (Result<ProcessingFormFieldSuggestionResponse>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный | Описание            |
| ----------- | ----------- | ----------- |---------------------|
| fieldId| Int| да | Идентификатор поля. |
| value| String| да | Значение поля.     |

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| Result\<ProcessingFormFieldSuggestionResponse\>) -> Void | Нет|Сallback со списоком подсказок |

## OPERATIONS

#### Получение списка операций для пользователя. Возвращает курсор на предыдущий/следующий набор операций, если такие имеются.
`func getOperations(
        cursor: String?,
        type: OperationElement.OperationType?,
        partner: String?,
        datetimeStart: String?,
        datetimeFinish: String?,
        status: OperationElement.Status?,
        completion: @escaping (Result\<Operations\>) -> Void)`
        
**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| cursor | String | да | указатель на получение списка операций, номер страницы |
| type | OperationElement.OperationType | да | тип операции |
| partner | String | да | фильтр партнера |
| datetimeStart | String | да | дата начала фильтрации операции |
| datetimeFinish | String | да | дата конца фильтрации операции |
| status | OperationElement.Status | да | фильр по статусу операции |

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<Operations\>) -> Void | нет| Сallback со списоком операций с предыдущим/следующим указателем |


#### Получение операции для пользователя
`func getOperationInfo(id: Int, type: OperationElement.OperationType, completion: @escaping (Result<OperationElement>) -> Void`
**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | String | нет | идентификатор операции |
| type | OperationElement.OperationType | нет | тип операции |

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<OperationElement\>) -> Void | нет|  Сallback с элементом операции |


#### Получение информации о кешбеке. Имеет перегрузку метода с параметром ключ-значения для гибкой настройки параметров.
`func getCashbackInfo(prevMonth: Int, scenario: String, collapsed: Bool, source: String, completion: @escaping (Result<CashbackInfo>) -> Void)`

**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| prevMonth | Int | нет | период запроса данных |
| scenario | String | нет | место запроса данных |
| collapsed | Bool | нет | скрытие виджетов в разделе |
| source | String | нет | источник перехода |

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<CashbackInfo\>) -> Void | нет| Сallback с информацией о кешбеке |

#### Перегрузка метода информации о кешбеке
`func getCashbackInfo(params: [String: String], completion: @escaping (Result<CashbackInfo>) -> Void)`

**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| parameters | Dictionary\<String, String\> | нет | параметры в формате ключ-значение |

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<CashbackInfo\>) -> Void | нет| Сallback с информацией о кешбеке |

## COMMON

##### Уведомляет сервер о показанных оповещениях ползьзователю. Возвращает список id оповений с актуальным статусом
`func snackShown(snacks: [SnackShownRequest], completion: @escaping (Result<SnackShownResponse>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| snacks| Array\<SnackShownRequest\> | нет | Массив показанных оповещений с id и timestamp |

**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<SnackShownResponse\>) -> Void | нет| Модель со списоком id оповещений с актуальным статусом |


#### Получение элементов для блока часто задаваемых вопросов.
`func getUserFaq(completion: @escaping (Result<UserFaq>) -> Void)`


**Сompletion**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| (Result\<UserFaq\>) -> Void | нет| Сallback со списком элементов  |



# Структура данных
## Version Info

#### `ProcessingVersionInfo`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| current | ProcessingVersionInfoCurrent | нет | Информация о текущей версии sdk  |
| actual | ProcessingVersionInfoActual | да | Информация об актуальной версии sdk, если есть |

#### `ProcessingVersionInfoCurrent`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| status | ProcessingVersionInfoStatus | нет | Статус текущей версии  |
| description | String | нет | Описание текущей версии |

#### `ProcessingVersionInfoActual`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| value | String | нет | Актаульная версия  |
| storeLink | String | да | Ссылка на мп в AppStore |
| description | String | нет | Описание актуальной версии |

#### `ProcessingVersionInfoActual` enum, String
| Имя свойства | Описание|
| ----------- |--------|
| actual | Актаульная версия  |
| minor | Обновление не требуется |
| major | Необходимо обновиться и предлагаем отправиться пользователю в AppStore за обновлением (ссылка на стор приходит с сервера) |
| critical | Критический статус, использование кошелька невозможно. Необходимо обновиться (ссылка на стор приходит с сервера)|

## Account

#### `UserResponse`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| requiredScreens | Array\<String\> | да | Список форм, которые должны отображатся перед страницей кошелька |
| account | Array\<Account\> | нет | Масиив моделей данных кошелька |
| messages | ProcessingServiceStatus | нет | Информационное сообщение о работе сервера  |
| snacks | ProcessingSnackResponse | да | Список снекбаров |
| banners | ProcessingBannersResponse | да | Список баннеров|
| unreadSnackIds | Array\<Int\> | да |  |
| recurrentBanks | Array\<ProcessingBank\> | да | Список привязанных к кошельку банков - рекуррентов |
| bankLinkType | String | да | Тип привязки банков рекуррент/me2me |
| forms | Array\<ProcessingFormAccount\> | нет | Список форм для упрощенной идентификации  |
| services | Array\<Services\> | да | Список сервисов  |

#### `UserPreviewResponseData`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| data | UserPreviewResponse | да | аккаунт пользователя в системе |

#### `UserPreviewResponse`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| banners | ProcessingBannersResponse | да | Список баннеров|
| account | Array\<Account\> | да | Масиив моделей данных кошелька |
| unreadSnackIds | Array\<Int\> | да |  |
| requiredStory | Int | да |  |
| messages | ProcessingServiceStatus | нет | Информационное сообщение о работе сервера  |
| bankLinkType | String | да | Состояние привязки банка |
| recurrentBanks | Array\<ProcessingBank\> | да | Список привязанных к кошельку банков - рекуррентов |
    
#### `Account` 
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор аккаунта  |
| status | String | нет| Состояние кошелька |
| isNeedBalance | Bool | нет  | Флаг отображать ли баланс|
| updatedAt | Double | да | |
| banks | Array\<ProcessingBank\>| нет  |    Список привязанных к кошельку банков |
| balance | Double | нет ( 0 по умолчанию) | Баланс пользователя |
| currency | String | нет | Строковый код валюты |

#### `UserInfoResponse` 
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| user | UserInfoUser | нет | Данные по профилю пользователя |
| banners | ProcessingBannersResponse | нет| Список баннеров|
| progress | ProgressResponse | нет  | Флаг отображать ли баланс|

#### `UserInfoUser` 
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| phone | String | нет | Номер телефона пользователя |

#### `UserInfoDetailsResponse` 
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| fields | Array\<UserInfoDetailsField\> | нет | Поля с детализированной информацией по профилю |

#### `UserInfoDetailsField` 
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор поля |
| label | ProcessingBannersResponse | нет| Имя поля|
| value | ProgressResponse | нет  | Значение|


## Banks
#### `ProcessingBank`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор банка в системе  |
| name | String | нет | Имя банка в системе  |
| imageLink | String | да | Ссылка на логотип банка  |
| isLinked | Bool | нет (false по умолчанию) | Состояние привязки банка. Если банк присутствует в списке, но его статус isLinked == false, это означает что банк в процессе привязки.   |
| isDefault | Bool| нет (false по умолчанию) | Банк выбран основным. С счета этого банка будут сниматься средства при платежах кошельком.  |
| isRetryPossible | Bool | да  | Возможность отправить запрос на привязку заново. |
| instructionUrl | String | да | Ссылка на инструкцию по привязке |
| type | String | да  | Тип привязки банка |
| linkStatus | String | да | Статус привязки банка  |

#### `AccountBankLinkResponse`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| status | String | нет |  Статус запроса на привязку банка к кошельку  |
| data | Array<AccountBankData> | да | Информация для привязки банка к кошельку |

#### `AccountBankData`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| link | AccountBankLink | нет |  Информация для привязки банка к кошельку  |

 #### `AccountBankLink`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | String | нет |  Идентификатор привязки банка  |
| url | String | да | Ссылка для привязки банка |
| expireTimeout | Int | нет | Время ожидания  |

#### `AccountBankLinkStatus`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| data | AccountLinkStatus | нет |  Информация по выполняющейся привязке банка к кошельку  |

#### `AccountLinkStatus`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| status | String | нет |  Результат привязки |
| finalStoryId | Int | да |  id story, которая будет показана после завешения метода привязки |

#### `UserFundingResponse`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| banner | ProcessingBannerItems | да |  Список баннеров  |
| buttons | Array\<UserFundingResponse\> | нет | Список кнопок с суммами пополнения  |
| sum | UserFundingSumItem | нет | Информация о сумме пополнения  |

#### `UserFundingSumItem`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| defaultSum | String | нет | Сумма по умолчанию  |
| min | String | нет | Минимальная сумма поплнения  |
| max | String | нет | Максимальная сумма поплнения  |
| currency | String | нет | Строковый код валюты  |

#### `UserFundingButtons`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| sum | String | нет | Сумма пополнения  |
| currency | String | нет | Строковый код валюты  |


## Common
#### `ProcessingServiceStatus`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| badStatus | ProcessingServiceStatusElement | да |   |
| badTime | ProcessingTimeZoneElement | да | |

#### `ProcessingServiceStatusElement`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| title | String | нет |  Заголовок |
| description | String | нет | Описание |

#### `ProcessingTimeZoneElement`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| title | String | нет |  Заголовок |
| text | String | нет | Заголовок|
| button | String | нет | Имя кнопки|

#### `ProcessingUserExperiments` enum 
| Case | Описание|
| ----------- | ----------- |
| smallQR(isAvailable: Bool) | Доступн ли пользователю данный эксперимент|
| bonusChange(isAvailable: Bool) | Доступн ли пользователю данный эксперимент |
| stories(isAvailable: Bool) |  Доступн ли пользователю данный эксперимент|
| bankLinkFunding(isAvailable: Bool) |  Доступн ли пользователю данный эксперимент|
| scanGo(isAvailable: Bool) | Доступн ли пользователю данный эксперимент |
| spar20(isAvailable: Bool) | Доступн ли пользователю данный эксперимент |


## Snack
#### `ProcessingSnackResponse`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| type | String | нет  |  Тип |
| items | Array\<ProcessingSnackItems\> | нет |  Список оповещений |

#### `ProcessingSnackItems`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет  |  Тип | Идентификатор оповещения 
| type | String| нет |  Тип оповещения |
| params | Array\<ProcessingParamsResponse\> | нет |  Ответ с параметрами name, value, type |

#### `ProcessingParamsResponse`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| name | String | нет  |  Тип | Имя 
| value | String| нет |  Значение |
| type | String | да |  Тип |

## Banners

#### `ProcessingBannersResponse`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| type | Stirng | нет | Вариант отображения баннеров, например списком, каруселью и т.п. |
| items | Array\<ProcessingBannerItems\> | да | Список баннеров |

#### `ProcessingBannerItems`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор баннера |
| type | String | да | Тип баннера |
| params | Array\<ProcessingBannerParams\> | да | Список параметров баннера |

#### `ProcessingBannerParams`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| name | String | нет | Имя параметра |
| value | String | нет | Значение параметра |
| textColor | String | да | Цвет значения (текста) параметра |
| type | String | да | Тип параметра |
| backgroundColor | String | да | Цвет фона параметра |

## Forms

#### `ProcessingFormAccount`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| type | String | нет | Тип формы, задает конкретный набор полей  |
| isRequired | Bool | нет | Необходимо ли пользователю заполнить/дозаполнить форму |
| description | String | да | Текст, который можно показать пользователю |
| status | ProcessingFormStatus | да | Статус формы последнего актуального заполнения. Пустое поле или нет поля – пользователь не заполнял форму |
| formId | Int | да | Идентификатор формы последнего актуального заполнения |
| version | Int | да | Версия формы последнего актуального заполнения |

#### `ProcessingFormStatus` enum String
| Case | Описание|
| ----------- | ----------- |
| draft | Форма заполнялась, но не была отправлена |
| commit | Форма была отправлена |
| processing | Форма в процессе обработки |
| invalid | Форма заполнена неверно |
| unknown | Форма неизвестна |
| success | Форма успешно заполнена |
| notExists | Форма отсутствует |
| new | Форма неизвестна |


#### `ProcessingForm`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | да | Идентификатор формы. Его может не быть, если форму пользователь получает впервые |
| version | Int | да | Версия формы. |
| status | ProcessingFormStatus | нет | Статус формы последнего актуального заполнения. |
| pageCount | Int | нет | Количество страниц |
| title | String | да | Заголовок формы |
| pages | Dictionary\<Int, ProcessingFormPage\> | нет | Словарь страниц сформированный по `FormPage.id` идентификаторам |
| visibilityChecks | Array\<ProcessingVisibilityCheck\> | нет | Список проверки видимости |

| pagesNumbered | Dictionary\<Int, ProcessingFormPage\> | нет | Словарь страниц сформированный по Array<ProcessingFormPage.number> номерам |
| allGroups | Dictionary\<Int, ProcessingFormGroup\>  | нет | Словарь групп сформированный по Array<ProcessingFormGroup.id> идентификаторам |
| allGroupsNumbered | Dictionary\<Int, ProcessingFormGroup\> | нет | Словарь групп сформированный по Array<ProcessingFormGroup.number> номерам |
| allFields | Dictionary\<Int, ProcessingFormField\> | нет | Все поля формы по Array<ProcessingFormField.id> идентификаторам|


#### `ProcessingFormPage`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор страницы |
| number | Int | нет | Номер страницы |
| name | String | да |     Имя страницы |
| description | String | да | Описание страницы |
| groups | Dictionary\<Int, ProcessingFormGroup\>| нет | Словарь групп сформированный по `FormGroup.id` |
| submitButtonText | String | да | Текст на кнопке отправки страницы формы |


#### `ProcessingFormGroup`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор группы |
| number | Int | нет | Номер группы |
| pageId | Int | нет | Идентификатор страницы |
| name | String | да | Имя группы |
| description | String | да | Описание группы |
| fields | Dictionary\<Int, ProcessingFormField\> | нет | Словарь полей сформированный по `FormField.id` |


#### `ProcessingFormField`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор поля |
| number | Int | нет | Позиция |
| groupId | Int | да | Идентификатор группы |
| isRequired | Bool | нет | Требуется ли для заполнения. По умолчанию всегда true |
| isVisible | Bool | нет | Видимость поля. По умолчанию всегда true |
| value | String | да | Значение поля |
| initalValue | String | да | Изначальное значение поля |
| validationResult | ProcessingValidationResult | да | Результат валидации поля |



#### `ProcessingValidationResult`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| isValid | Bool | нет | Состояние валидности поля  |
| isRequired | Bool | нет | Требуется ли для заполнение  |
| errorMessage | String | да | Сообщение об ошибке  |



#### `ProcessingRadioGroup` : FormField
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|

| choices | Dictionary\<Int, RadioButton\> | нет |   Словарь переключателей сформированный по `RadioButton.id` |
| placeholder | String | да | Заполнитель |

#### `RadioButton`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор кнопки переключателя |
| radioGoupId | Int | нет | Идентификатор группы переключателя |
| placeholder | String | нет | Описание кнопки переключателя |
| value | String | нет | Значение, уходящее в группу переключателя |
| selected | String | нет | Состояние переключателя. По умолчанию false. |

#### `ProcessingEditText` : ProcessingFormField
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| placeholder | String | да | Заполнитель |
| inputType | EditTextInputType | нет | Тип поля |
| mask | String | да | Маска для поля |
| minLength | Int | да | Минимальное количество символов |
| maxLength | Int | да | Максимальное количество символов |
| hideSymbols | Bool | нет | Скрывать символы. По умолчанию false |
| nextCursor | Int | да | Идентификатор следующего поля |
| hasSuggestion | Bool | нет |  Проверка на наличие в поле подсказки |

#### `ProcessingVisibilityCheck`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| dependantFieldId | Int | нет | Идентификатор зависимого поля |
| parentFieldId | Int | нет | Идентификатор родительского поля. У группы зависимых элементов родительский идентификатор совпадает. |
| operation | ProcessingVisibilityCheckOperation | нет | Операция проверки видимости |
| value | String | нет | Значение проверки видимости |


#### `ProcessingVisibilityCheckOperation ` enum
| Имя | Описание|
| ----------- | ----------- |
| equal | Проверка на равенство. `field.value == value` |
| notEqual | Проверка на равенство. `field.value != value` |
| unknown | Всегда возвращает true. |


#### `ProcessingFormDraft`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| requestId | Int | нет | Актуальный идентификатор формы |
| status | ProcessingStatusDraft | нет | Статус драфта формы |
| version | Int | нет | Актуальная версия формы |
| details | Array\<ProcessingDraft\> | да | Список невалидных полей и их описание |

#### `ProcessingStatusDraft`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| fieldId | Int | нет | Идентификатор поля |
| errorCode | Stirng | нет | Код ошибки валидации формы |
| description | Stirng | нет | Описание |
| userMessage | Stirng | нет | Описание ошибки для пользователя |

#### `StatusDraft   ` enum
| Имя | Описание|
| ----------- | ----------- |
| error | В форме имеются поля, не прошедшие валидацию |
| ok | Все поля в черновике валидны |

#### `ProcessingFormFieldSuggestionResponse`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| data | ProcessingFormFieldSuggestion | нет | Данные подсказок |

#### `ProcessingFormFieldSuggestion`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| suggestions | Array\<ProcessingFormFieldSuggestionResponseElement\> | нет | Список подсказок|

#### `ProcessingFormFieldSuggestionResponseElement`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор подсказки  |
| value | String | нет | Значение подсказки  |

## COMMON

#### `SnackShownRequest`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор оповещения  |
| timestamp | Int | нет |  Timestamp оповещения |

#### `SnackShownResponse`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| data | Data | нет | Данные сообщения |

#### `Data`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| snacks | Array\<SnackShownStatus\> | нет | Список оповещений |

#### `SnackShownStatus`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор оповещения  |
| status | String | нет | Статус оповещения  |

## Operations

#### `Operations`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| operations | Array\<OperationElement\> | нет | Список элементов операции|
| nextCursor | String | да | Указатель на следующий список операций |

#### `OperationElement`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор операции |
| timestamp | Int | нет | Дата и время операции |
| type | OperationType | нет | Тип операции |
| status | StatusStatus | нет | Статус операции |
| name | String | нет | Название операции – русский аналог type |
| imageUrl | String | нет | Изображение операции |
| merchant | String | нет | Торговец операции |
| from | FromAndToElement | нет | Откуда произведена операция |
| to | FromAndToElement | нет | Куда произведена операция |
| amount | String | нет | Сумма операции |
| currency | CurrencyElement | да | Валюта операции |
| childOperations | Array\<OperationItem\> | нет | Информация о дочерних операциях |
| parentOperations | Array\<OperationItem\> | нет | Информация о родительских операциях |
| bill  | String | да | чек |
| externalLink  | String | да | внешняя ссылка |
| moneyFlowDirection  | MoneyFlowDirection | нет | поступают деньги или уходят со счета пользователя |
| additionalImageId  | String | да | Дополнительное изображение операции (например значок СБП) |
| cancelDescription  | String | да | писание причины отмены |

#### `OperationElement.OperationType ` enum
| Имя | Описание|
| ----------- | ----------- |
| purchase | покупка  |
| invoice | me2me пополнение |
| bankLink | пополнение для привязки |
| refill | пополнение инициированное пользователем из приложение его банка |
| refund | возврат |
| withdrawal | withdrawal|
| unknown | неизвестная операция, если не удалось распарсить ответ бека |

#### `OperationElement.Status ` enum
| Имя | Описание|
| ----------- | ----------- |
| canceled | отменена  |
| inProcess | в обработке |
| success | выполнена |

#### `OperationElement.MoneyFlowDirection ` enum
| Имя | Описание|
| ----------- | ----------- |
| income | деньги поступают на счёт пользователя в монете - в приложении знак "+" |
| outcome | outcome - деньги уходят со счёта пользователя в монете |

#### `OperationItem`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор операции |
| type | OperationElement.OperationType | нет | Тип операции |
| name | String | нет | Имя операции |
| imageUrl | String | нет | Изображение операции |
| amount | String | нет | Сумма операции |
| moneyFlowDirection  | OperationElement.MoneyFlowDirection | нет | поступают деньги или уходят со счета пользователя |
| merchant | String | нет | Торговец операции |
| currency | CurrencyElement | нет | Валюта операции |
| additionalImageId  | String | да | Дополнительное изображение операции (например значок СБП) |

#### `CurrencyElement`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | String | нет | Идентификатор валюты |
| symbol | String | нет | Символ валюты |

#### `FromAndToElement`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| name | String | нет | Торговец операции |
| description | String | да | Описание торговца |

#### `CashbackInfo`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| collapsed | Bool | нет  | скрытие виджетов в разделе |
| widgets | Array\<WidgetsData\> | нет | список виджетов |

#### `WidgetsData`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| name | WidgetsName | да | имя |
| id | Int | да | идентификатор |
| collapsable | Bool | да | идентификатор |
| text | String | да | текст |
| sum | String | да | сумма |
| currency | String | да | валюта |
| params | Array\<WidgetsParamData\> | да | параметры |
| items | Array\<WidgetsItemData\> | да | элементы |
| left | WidgetParamLeft | да | левый элемент |
| right | WidgetParamRight | да | правый элемент |
| pointerDirection | WidgetPointerDirection | да | направление |
| currentMonth | Int | да | текущий месяц |
| onClick | CashbackOnClick | да | действие по нажатию |

#### `WidgetsParamData`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| name | WidgetsParamName | да | имя |
| id | Int | да | идентификатор |
| type | WidgetsParamType | да | тип |
| value | String | да | значение |
| onClick | CashbackOnClick | да | действие по нажатию |
| fontColor | String | да | цвет текста |
| currency | String | да | валюта |
| currentMonth | Int | да | текущий месяц |

#### `WidgetPointerDirection` enum
| Имя |Описание|
| ----------- | ----------- |
| left |Налево |
| right | Направо |

#### `WidgetsParamName` enum, String
| Имя |Описание|
| ----------- | ----------- |
| text |текст |
| title | заголовок |
| description |описание |
| right |  |
| background |фон |
| sum | сумма |
| rightElement | элемент справа |
    

#### `WidgetsParamType` enum
| Имя |Описание|
| ----------- | ----------- |
| regular | |
| medium |  |
| bold |  |
| button |  |
| open |  |
| imageUrl |  |


#### `WidgetParamLeft`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| sum | String | нет | сумма |
| currency | String | нет | валюта |
| text | String | нет | текст |

#### `WidgetParamRight`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| value | String | нет | значение |
| text | String | нет | текст |

#### `CashbackOnClick`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| value | String | нет | значение |
| type | String | нет | тип |

#### `WidgetsItemData`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| prefix | String | нет | префикс |
| sum | String | нет | сумма |
| currency | String | нет | валюта |
| cashback | String | нет | кешбек |
| fullness | Int | нет | заполненность |
