# Быстрый старт
##### Компиляция из исходников
1. Откройте в Xcode файл Processing.xcodeproj 
2. Внесите необходимые изменения.
3. Используйте для сборки фреймворка баш скрипт build.sh в корне проекта
4. В папке build появится Processing.xcframework, скопируйте его и целевой проект
5. В целевом проекте добавьте Processing.xcframework в настройках проекта General → Frameworks, Libraries, and Embded Content
6. Импортируйте модуль с помощью import Processing в коде
7. Перед использованием обязательно вызовите Processing.configure(buyerId: String) и настройте эндпоинт с помощью Processing.beta = false (по умолчанию Processing.beta == true ) 
##### Использование скомпилированный framework
1. В целевом проекте добавьте Processing.xcframework в настройках проекта General → Frameworks, Libraries, and Embded Content
2. Импортируйте модуль с помощью import Processing в коде
3. Перед использованием обязательно вызовите Processing.configure(buyerId: String) и настройте эндпоинт с помощью Processing.beta = false (по умолчанию Processing.beta == true ) 

# Сценарии использования SDK KD Pay

### Инициализация sdk
Для инициализации sdk необходимо использовать Processing.confifure(buyerId: `buyerId`) с переданным аргументом id лица, использующего процессинг sdk.
```
Processing.confifure(buyerId: `buyerId`)
```

### Авторизация пользователя в системе KD Pay
Авторизация пользователя проходит в два этапа, для этого необходимо запросить у пользователя номер телефона (обязательное поле) и имя (необязательное). После чего указанные данные передаются в метод `login(phone, name)`. Сервер в ответ на запрос посылает на номер пользователя шестизначный код, который вместе с `fcm` токеном нужно передать в метод `confirmCode(code, fcmToken)`. При успешном ответе от сервера sdk сохраняет в кеше необходимые данные (публичный ключ сервера, идентификаторы сессии и пользователя). После чего пользователь считается авторизованным 

```
func login(phone: String, name: String?, uid: String?, completion: ((Result<Int>) -> Void)? = nil) {  
    Processing.login(userPhone: phone, userName: name, uid: uid, completion: completion)
}
 
fun confirm(smsCode: String, fcmToken: String, completion: ((Result<Void>) -> Void)? = nil) {  
    Processing.confirmCode(code: smsCode, fcmToken: fcmToken, completion: completion)
}
```

### Создание кошелька
Для создания кошелька используется метод `createAccount()`. Для этого пользователь должен быть авторизован в системе KD Pay. Создание аккаунта занимает некоторое время и может быть дольше 30 секунд.

### Получение аккаунта пользователя
Получение аккаунта осуществляется с помощью метода `getAccount(cached: Bool, completion: @escaping (Result<WalletAccountResult>) -> Void)`. Для успешного получения состояния кошелька пользователь должен быть авторизован. Если клиент не авторизован в системе метод выбрасывает исключение. При отсутствии аккаунта в ответе поле `walletAccountState` будет иметь значение `none`.
```
func getAccount() {
    Processing.getAccount(cached: false) { result in 
        switch result {
        case .success(let account): 
            if case account.walletAccountState = .none {
                processAccountNotExists()
            } else
            if case account.walletAccountState = .ready {
                processSucess(account)
            }
            /// other states
        case .failure(let error):
            processNotAuthorized(error)
        }
    }
}
```

### Привязка счета к кошельку
Для того чтобы привязать к кошельку пользователя банковский счет необходимо получить список банков с помощью метода `getBanks()`. Для того чтобы использовать метод пользователь должен быть авторизован и у пользователя должен быть создан аккаунт. В противном случае метод верет ошибку в Result. Далее из списка полученных банков пользователь выбирает необходимый. После чего на сервер отправляется запрос на привязку счета с помощью метода `addNewBank(bankId)`. Если сервер ответил успехом, то пользователю предлагается перейти в мобильное приложение банка, и завершить процесс привязки счета.

```
fun selectBanksExample() {
    Processing.getBanks { result in 
        switch result {
        case .success(let banks): 
            Processing.addNewBank(bankId) { result in 
                switch result {
                case .success: 
                    showBankBindingSuccess()
                case .failure(let error):
                    showBankBindingError(error)   
                }
            }
        case .failure(let error):
            processNotAuthorized(error)
        }
    }
}
```

### Проверка статуса привязки счета к аккаунту пользователя
Так как привязка счета осуществляется пользователем в стороннем мобильном приложении, то при вызове метода `addNewBank` не возвращается результат привязки счета. Для того чтобы проверить статус нужно запросить аккаунт пользователя с помощью метода `getAccount`. Ориентируясь на список привязанных счетов и id банка, счет которого привязывается к аккаунту можно понять в каком статусе находится привязка счета. Если банк присутствует списке привязанных банков, но поле `isLinked` у банка равно `false` - это значит что счет в банке в процессе привязки к кошельку. В случае если поле равно `true`, значит процесс привязки счета успешен. В случае если в списке банков нет искомого, то это означает что произошла ошибка при привязке счета.

```
fun checkBindingStatus(cachedBankId: Int) {
    Processing.getAccount(cached: false) { result in 
        switch result {
        case .success(let account): 
            guard let bank = result.walletAccount.banks.first(where: { $0.id == cachedBankId }) else {
                processError()
                return
            }
            if bank.isLinked {
                processSucess()
            } else {
                processError()
            }
        case .failure(let error):
            processFail(error)
        }
    }
}
```

### Установить банк по умолчанию
После привязки счета этот счет становится счетом по умолчанию. Для того чтобы изменить счет по умолчанию следует воспользоваться методом `setDefaultBank`. Для того чтобы использовать метод пользователь должен быть авторизован и у пользователя должен быть создан аккаунт, а так же привязан как минимум один счет. Для начала следует получить данные аккаунта методом `getAccount`, предложить пользователю выбрать из списка привязанных счетов тот, который он хочет сделать основным. Далее передать `id` этого банка в метод `setDefaultBank(bankId)`. В результате чего в случае успеха метод вернет измененный и актуальный список привязанных счетов.

```
fun setDefaultBank() {
    Processing.getAccount(cached: false) { result in 
        switch result {
        case .success(let account): 
            guard let bank = result.walletAccount.banks.first(where: { $0.id == chossedBankId }) else {
                processError()
                return
            }
            Processing.setDefaultBank(bankId: bank.id) { result in 
                switch result {
                case .success: 
                    processSucess()
                case .failure(let error):
                    processError(error)
                }
            }
        case .failure(let error):
            processError(error)
        }
    }
}
```

### Генерация qr кода
Для быстрого создания строки кодируемой в qr код достаточно использовать метод `enerateQrString(loyaltyId: String)`. Метод не требует наличие интернета в случае если есть предзагруженные otp коды. OTP коды неявно подгружаются при использовании метода `generateQrString`, либо это можно сделать явно с помощью метода `requestOtpCode(amount)`. Для использования методов пользователь должен быть предварительно авторизован и иметь аккаунт в системе.
```
fun preloadOtp() {
    if !Processing.haveOtpCode { 
        Processing.requestOtpCode(OTP_CODE_AMOUT) 
    }
}
 
fun generateQrString(loyaltyId: String) {
    Processing.generateQrString(loyaltyId: loyaltyId) { result in 
        switch result {
        case .success(let qrString): 
            renderQrCode(qrString)
        case .failure(let error):
            processError(error)
        }
    }
}
```

### Получение формы
Получение формы происходит через метод `getForm(type: String, formId: Int?, formStatus: ProcessingFormStatus?, completion: @escaping (Result<ProcessingForm>) -> Void)`. Если запрос формы произойдет без `formId`, то сервер пришлет чистую форму.
Если запрос происходит с `formId`, то SDK отправит 2 запроса на сервер:  `/form/fields` и `/request`. Первый запрос получит форму, второй получит статус валидации для полей. SDK соединит полученные результаты и вернет форму с провалидированными полями.

*Примечание: количество полей и их состав сервер может поменять на своей стороне, форма является динамической.*

```
fun getFormFields(type: String, formId: Int?, formStatus: FormStatus) {
    Processing.getFormFields(type: type, formId: formId, formStatus: formStatus) { result in 
        switch result {
        case .success(let formFields): 
            processSucess(formFields)
        case .failure(let error):
            processError(error)
        }
    }
}
```

### Валидация полей формы
Для валидации полей используется метод `sendFormDraft(type: String, formId: Int?, version: Int?, draftFields: [Int: String], completion: @escaping (Result<ProcessingFormDraft>) -> Void)`. Первый запрос на валидацию всегда будет происходить без `formId` и `version`. Актуальные значения `formId` и `version` вернутся при первой и последующей валидации. Валидировать необходимо все заполненные поля.

*Примечание: сервер сохраняет в следующем драфте поля "вместо" предыдущих. То есть если в первой версии драфта МП отправило заполненное поле 1, а потом не прислало это поле или прислало пустое, то в этой версии драфта поле не будет сохранено на сервере.*

При `sendFormDraft(type: String, formId: Int?, version: Int?, draftFields: [Int: String], completion: @escaping (Result<ProcessingFormDraft>) -> Void)` SDK уберет пустые поля из запроса, чтобы не заваливать пользователя ошибками на пустых полях.

```
fun sendFormDraft(type: String, formId: Int?, version: Int?, draftFields: [Int: String], completion: @escaping (Result<ProcessingFormDraft>) -> Void) {  
    Processing.sendFormDraft(type: type, formId: formId, version: version, draftFields: draftFields) { result in 
        switch result {
        case .success(let formDraft): 
            processSucess(formDraft)
        case .failure(let error):
            processError(error)
        }
    }
}
```


### Отправка формы на проверку
Для валидации полей используется метод `sendFormCommit(type: String, formId: Int, version: Int?, draftFields: [Int: String], completion: @escaping (Result<ProcessingFormDraft>) -> Void)`. Первый запрос на `commit` может происходить без `formId` и `version`. Актуальные значения `formId` и `version` вернутся при первом и последующем `commit`. Отправлять необходимо все поля.

При `sendFormCommit(type: String, formId: Int, version: Int?, draftFields: [Int: String], completion: @escaping (Result<ProcessingFormDraft>) -> Void)` сервер отправит **все** поля, даже пустые, чтобы уведомить пользователя о незаполненных полях.

```
fun sendFormCommit(type: String, formId: Int, version: Int?, draftFields: [Int: String], completion: @escaping (Result<ProcessingFormDraft>) -> Void) {  
    Processing.sendFormDraft(type: type, formId: formId, version: version, draftFields: draftFields) { result in 
        switch result {
        case .success(let formCommi): 
            processSucess(formCommi)
        case .failure(let error):
            processError(error)
        }
    }
}
```





# Методы

##### Метод для проверки состояния сервиса.
`ping()` 

##### Метод устанавливает идентификатор магазина.
`configure(buyerId: String)`

##### Метод возвращает идентификатор пользователя, если имеется.
`getUserInfo: UserInfo`

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| UserInfo      | Нет       | Возвращает userId пользователя, если такой имеется |


##### Проверка доступа к кошелькам.
`getUserExperiment(phone: String, completion: @escaping (Result<ProcessingUserExperiments>) -> Void)`
**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| UserExperiments      | Нет       | Вернет значение разрешены кошельки или нет для данного пользователя в зависимости от параметра BASE_FLOW|

##### Проверяет авторизован ли пользователь в системе по таким признакам как наличие в кеше идентификатора сессии и пользователя. А так же наличию публичного ключа сервера.
`isAuthorized: Boolean`
**Возвращает**

| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| Boolean| Нет| true - пользователь авторизован, false - не авторизован (Нет идентификатора сессии, пользователя, либо публичного ключа сервера в кеше SDK)|


##### Первый шаг авторизации пользователя в системе KD Pay. Служит для запроса смс кода на указанный в параметрах модуль.
`login(userPhone: String, userName: String? = nil, uid: String?, completion: ((Result<Int>) -> Void)? = nil)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| userPhone| String| нет | Телефон указанный пользователем |
| userName| String| да | Имя пользователя |
| uid| String| да | Идентификатор пользователя в системе.  Если не указан используется 64-битное число (выраженное в виде шестнадцатеричной строки), уникальное для каждой комбинации ключа подписи приложения, пользователя и устройства.  |

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| Int| Нет| Идентификатор пользователя|


##### Второй шаг авторизации пользователя в системе KD Pay. Служит для подтверждения телефона пользователя и завершения авторизации.
`confirmCode(code: String, fcmToken: String, completion: ((Result<Void>) -> Void)? = nil)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| code| String| нет | Смс код полученный на указанный в методе login номер телефона |
| fcmToken| String| нет | Токен Firebase Messaging Service [подробнее](https://firebase.google.com/docs/cloud-messaging/ios/client). |


##### Метод формирует строку, которую впоследствии можно закодировать в qr код для оплаты на кассе. При отсутствии закерованных otp кодов, метод берет себя работу по их получению и кешированию.
`generateQRString(loyaltyId: String, completion: @escaping (Result<String>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| loyaltyId| String| нет | Идентификатор лояльности во внешней системе |


**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| String | Нет| Строка для кодирования в Qr код|


##### Запрос на создание аккаунта в системе KD Pay.
`createAccount(completion: @escaping (Result<WalletAccountResult>) -> Void)`


**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| WalletAccountResult | Нет| Ответ содержащий аккаунт пользователя в системе и текущий статус аккаунта. |


##### Получение аккаунта пользователя в системе KD Pay.
`getAccount(cached: Bool, completion: @escaping (Result<WalletAccountResult>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| cached| Boolean| нет | Флаг для определения откуда брать данные аккаунта. true - из кеша, false - из сервера с последующим кешированием.|


**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| WalletAccountResult | Нет| Ответ содержащий аккаунт пользователя в системе и текущий статус аккаунта |

##### Получение списка банков, счета которых можно привязать к кошельку.
`getBanks(completion: @escaping (Result<[ProcessingBank]>) -> Void)`
**Параметры**


**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| List<Bank> | Нет| Cписок банков, счета которых можно привязать к кошельку. |


##### Метод для установки банка по умолчанию. Выбирается из списка привязанных банков.
`addNewBank(bankId: Int, completion: @escaping (Result<Void>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| bankId| Int| нет |Идентификатор банка в системе KD Pay, выбранных из списка банков получаемых с помощью метода getBanks()|


##### Метод для установки банка по умолчанию. Выбирается из списка привязанных банков.
`setDefaultBank(bankId: Int, completion: @escaping (Result<[ProcessingBank]>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| bankId| Int| нет | Идентификатор банка в системе KD Pay, выбранных из списка привязанных банков |


**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| List<Bank> | Нет| Список привязанных счетов банков, с измененным банком по умолчанию. |

##### Метод для проверки наличия сохраненных otp кодов в кеше sdk.
`haveOtpCode: Bool`

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| Boolean | Нет| true - есть как минимум один сохраненный otp код. false - не сохраненных otp кодов. |


##### Метод для получение otp кодов с сервера с последующим сохранением в кеш sdk.
`requestOtpCodes(amount: Int = 10, completion: ((Result<Bool>) -> Void)? = nil)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| amount| Int| нет | Количество запрашиваемых otp кодов. |

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| Boolean | Нет| true - есть как минимум один сохраненный otp код. false - нет сохраненных otp кодов. |

##### Метод для получения количества оставшихся otp кодов
`remainingOtpCodes: Int`

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| Int | Нет| количество оставшихся otp кодов |

##### Запрашивает  указанное в параметре количество otp кодов, если их оставшееся количество валидных меньше 6
`requestOtpCodesIfRequired(amount: Int = 10)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| amount| Int| нет | Количество запрашиваемых otp кодов. |

##### Метод получения формы упрощенной идентификации
`getForm(type: String, formId: Int?, formStatus: ProcessingFormStatus?, completion: @escaping (Result<ProcessingForm>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| type| String| нет | Тип формы, которая задает определенный набор полей. |
| formId| Int| да | Идентификатор последнего актуального заполнения. |
| formStatus| ProcessingFormStatus| да | Статус последнего актуального заполнения. |

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| ProcessingForm | Нет| Форма для упрощенной идентификации пользователя |


##### Метод для отправки черновика формы
`sendFormDraft(type: String, formId: Int?, version: Int?, draftFields: [Int: String], completion: @escaping (Result<ProcessingFormDraft>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| type| String| нет | Тип формы, которая задает определенный набор полей. |
| formId| Int| да | Идентификатор последнего актуального заполнения. |
| version| Int | да | Версия последнего актуального заполнения. |
| draftFields| [Int: String] | нет | Идентификатор валидируемого поля и его значение. |

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| ProcessingFormDraft | Нет| Актуальная информация формы и список невалидных полей |


##### Метод для коммита формы
`sendFormCommit(type: String, formId: Int, version: Int?, draftFields: [Int: String], completion: @escaping (Result<ProcessingFormDraft>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| type| String| нет | Тип формы, которая задает определенный набор полей. |
| formId| Int| да | Идентификатор последнего актуального заполнения. |
| version| Int | да | Версия последнего актуального заполнения. |
| draftFields| [Int: String] | нет | Идентификатор валидируемого поля и его значение. |

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| ProcessingFormDraft | Нет| Актуальная информация формы и список невалидных полей |

##### Метод проверки версии бибоиотеки
`checkLibraryVersion(cacheInterval: TimeInterval, completion: @escaping (Result<ProcessingVersionInfo>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| cacheInterval| TimeInterval aka Double| нет | Интервал проверки кэша в секундах. Если интервал 0, то кеш не используется. |

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| ProcessingVersionInfo | нет| Возвращает результат проверки.  |


##### Полная очистка кеша sdk. Удаляются идентификаторы сессии, пользователя, публичный ключ сервера, отп коды и номер телефона пользователя.
`logout()`


# Структура данных

#### `WalletAccountResult`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| walletAccountState | ProcessingWalletAccountState | нет | Описывает состояние аккаунта |
| walletAccount | ProcessingWalletAccount | да | Модель данных кошелька |


#### `ProcessingWalletAccountState` enum
| Имя | Описание |
| ----------- | --------|
| none |  Нет аккаунта пользователя |
| pending | Аккаунта в процессе создания |
| ready | Аккаунт существует и готов к работе |
| banned | Аккаунт забанен |


#### `ProcessingWalletAccount`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор аккаунта  |
| balance | Double | нет ( 0 по умолчанию) | Баланс пользователя |
| currency | String | нет | Строковый код валюты |
| status | ProcessingWalletStatus | нет (Undefined по умолчанию) | Состояние кошелька |
| banks | [ProcessingBank] | нет  |     Список привязанных к кошельку счетов |
| forms | ProcessingFormAccount? | да  | Список форм для упрощенной идентификации |


#### `ProcessingWalletStatus ` enum
| Имя | Описание |
| ----------- | --------|
| undefined | Неизвестное состояние кошелька |
| active | Кошелек активен и готов к работе |
| blocked | Кошелек заблокирован |
| closed | Кошелек закрыт |


#### `ProcessingBank`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор банка в системе  |
| name | String | нет | Имя банка в системе  |
| imageLink | String | да | Ссылка на логотип банка  |
| isLinked | Bool | нет (false по умолчанию) | Состояние привязки счета. Если банк присутствует в списке, но его статус isLinked == false, это означает что счет в процессе привязки.   |
| isDefault | Bool | нет (false по умолчанию) | Счет банка выбранный основным. С этого счета будут сниматься средства при платежах кошельком.  |


#### `ProcessingFormAccount`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| type | String | нет | Тип формы, задает конкретный набор полей  |
| status | ProcessingFormStatus | да | Статус формы последнего актуального заполнения. Пустое поле или нет поля – пользователь не заполнял форму |
| formId | Int | да | Идентификатор формы последнего актуального заполнения |
| isRequired | Bool | нет | Необходимо ли пользователю заполнить/дозаполнить форму |
| description | String | да | Текст, который можно показать пользователю |
| version | Int | да | Версия формы последнего актуального заполнения |

#### `ProcessingFormStatus` enum
| Имя | Значение |Описание|
| ----------- | ----------- | ----------- |
| draft | draft | Форма заполнялась, но не была отправлена |
| commit | commit | Форма была отправлена, но не запушена |
| processing | processing | Форма в процессе обработки |
| invalid | invalid | Форма заполнена неверно |
| success | success | Форма успешно заполнена |
| notExists | not_exists | Форма отсутствует |
| unknown | unknown | Форма неизвестна |


#### `ProcessingForm`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | да | Идентификатор формы. Может его не быть, если форму пользователь получает впервые |
| version | ProcessingFormStatus | да | Статус формы последнего актуального заполнения. |
| pageCount | Int | нет | Количество страниц |
| title | String | да | Заголовок формы |
| pages | Dictionary<Int, ProcessingFormPage> | нет | Словарь страниц сформированный по `FormPage.id` идентификаторам |
| visibilityChecks | [ProcessingVisibilityCheck] | нет | Список проверки видимости |


#### `ProcessingFormPage`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор страницы |
| number | Int | нет | Номер страницы |
| name | String | да |     Имя страницы |
| description | String | да | Описание страницы |
| groups | Dictionary<Int, ProcessingFormGroup> | нет | Словарь групп сформированный по `ProcessingFormGroup.id` |
| submitButtonText | String | да | Текст на кнопке отправки страницы формы |

#### `ProcessingVisibilityCheck`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| dependantFieldId | Int | нет | Идентификатор зависимого поля |
| parentFieldId | Int | нет | Идентификатор родительского поля. У группы зависимых элементов родительский идентификатор совпадает. |
| operation | ProcessingVisibilityCheckOperation | да | Операция проверки видимости |
| value | String | да | Значение проверки видимости |


#### `ProcessingVisibilityCheckOperation ` enum
| Имя | Описание|
| ----------- | ----------- |
| equals | Проверка на равенство. `ProcessingVisibilityCheckOperation.check(field: ProcessingFormField, value: String) -> Bool` |
| notEqual | Проверка на неравенство. `ProcessingVisibilityCheckOperation.check(field: ProcessingFormField, value: String) -> Bool` |
| unknown | Всегда возвращает true.|


#### `ProcessingFormGroup`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор группы |
| number | Int | нет | Номер группы |
| pageId | Int | нет | Идентификатор страницы |
| name | String | да | Имя группы |
| description | String | да | Описание группы |
| fields | Dictionary<Int, ProcessingFormField> | нет | Словарь полей сформированный по `ProcessingFormField.id` |


#### sealed `ProcessingFormField`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор поля |
| groupId | Int | да | Идентификатор группы |
| isRequired | Bool | нет | Требуется ли для заполнения По умолчанию всегда true |
| isVisible | Bool | нет | Видимость поля По умолчанию всегда  true |
| value | String | да | Значение поля |
| initalValue | String | да | Изначальное значение поля |
| validationResult | ProcessingValidationResult | да | Результат валидации поля |

#### `ProcessingRadioGroup` : ProcessingFormField
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор поля |
| groupId | Int | да | Идентификатор группы |
| isRequired | Bool | нет |     Требуется ли для заполнения По умолчанию всегда true |
| choices | Dictionary<Int, RadioButton> | нет |     Словарь переключателей сформированный по `RadioButton.id` |
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
| id | Int | нет | Идентификатор поля |
| groupId | Int | да | Идентификатор группы |
| isRequired | Bool | нет | Требуется ли для заполнения. По умолчанию всегда true |
| placeholder | String | да | Заполнитель |
| inputType | ProcessingEditTextInputType | нет | Тип поля |
| mask | String | да | Маская для поля |
| minLength | Int | да | Минимальное количество символов |
| maxLength | Int | да | Максимальное количество символов |
| hideSymbols | Bool | нет | Скрывать символы. По умолчанию false |

#### `ProcessingEditTextInputType  ` enum
| Имя | Описание|
| ----------- | ----------- |
| text | Текстовое поле |
| date | Поле даты |
| number | Числовое поле |


#### `ProcessingValidationResult`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| isValid | Bool | нет | Состояние валидности поля  |
| isRequired | Bool | нет | Требуется ли для заполнения  |
| errorMessage | String | да | Сообщение об ошибке  |


#### `ProcessingFormDraft`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| requestId | Int | нет | Актуальный идентификатор формы |
| status | ProcessingStatusDraft | нет | Статус драфта формы |
| version | Int | нет | Актуальная версия формы |
| details | [ProcessingDraft] | да | Список невалидных полей и их описание |

#### `ProcessingDraft`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| fieldId | Int | нет | Идентификатор поля |
| errorCode | Stirng | нет | Код ошибки валидации формы |
| description | Stirng | нет | Описание |
| userMessage | Stirng | нет | Описание ошибки для пользователя |


#### `ProcessingStatusDraft   ` enum
| Имя | Описание|
| ----------- | ----------- |
| error | В форме имеются поля, не прошедшие валидацию |
| ok | Все поля в черновике валидны |



#### `ProcessingUserExperiments` enum
| Имя свойства |Описание|
| ----------- |--------|
| normal | Кошельки доступны |
| notAvailable(hasQrCodes: Bool) | Кошельки недоступны |

#### sealed `UserInfo` enum
| Имя свойства |Описание|
| ----------- |--------|
| exist(userId: Int) | Возвращается, если пользователь авторизован |
| notExist | Возвращается, если пользователь `не` авторизован |

#### `ProcessingVersionInfoStatus` enum
| Имя свойства |Описание|
| ----------- |--------|
| actual | Версия библиотеки актуальная, обновления не требуются. |
| minor | Доступно небольшое обновление. |
| major | Доступно обновление. |
| critical | Требуется обновление, использование кошельков невозможно. |

#### `ProcessingVersionInfo`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| current | ProcessingVersionInfoCurrent | нет | Информация о текущей версии sdk |
| actual | ProcessingVersionInfoActual | да | Информация об актуальной версии sdk, если есть |

#### `ProcessingVersionInfoCurrent`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| status | ProcessingVersionInfoStatus | нет | Статус текущей версии |
| description | String | нет | Описание текущей версии |

#### `ProcessingVersionInfoActual`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| value | String | нет | Актаульная версия (вида 1.0.0) |
| storeLink | String | да | Ссылка на мп в сторе|
| description | String | нет | Описание актуальной версии |
