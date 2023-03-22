# Быстрый старт
##### Вхождение в проект SPAR
Ссылка: https://conf.siberian.pro/pages/viewpage.action?pageId=55574249
##### Руководство по fastlane в SPAR (как сделать iOS build)
Ссылка: https://conf.siberian.pro/pages/viewpage.action?pageId=77509566
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
Авторизация пользователя проходит в два этапа, для этого необходимо запросить у пользователя номер телефона (обязательное поле, на этот номер будет создан кошелёк), дополнительный номер телефона (необязательное), имя (необязательное), идентификатор устройства (если не передать, СДК попытается подставить автоматически) и идентификатор карты лояльности (необязательный). После чего указанные данные передаются в метод `login(phone, basePhone, name, deviceId, loyaltyId)`. Сервер в ответ на запрос посылает на номер пользователя шестизначный код, который вместе с `fcm` токеном нужно передать в метод `confirmCode(code, fcmToken)`. При успешном ответе от сервера sdk сохраняет в кеше необходимые данные (публичный ключ сервера, идентификаторы сессии и пользователя). После чего пользователь считается авторизованным

```
func login(phone: String, userBasePhone: String?, name: String?, deviceId: String?, loyaltyId: String?) {
    Processing.login(userPhone: phone, userName: name, deviceId: deviceId, completion: completion)
}
 
fun confirm(smsCode: String, fcmToken: String, completion: ((Result<Void>) -> Void)? = nil) {  
    Processing.confirmCode(code: smsCode, fcmToken: fcmToken, completion: completion)
}
```

### Создание кошелька
Для создания кошелька используется метод `createAccount(referrerId: String? = null)`. Для этого пользователь должен быть авторизован в системе KD Pay. Создание аккаунта занимает некоторое время и может быть дольше 30 секунд.

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

### Привязка банка к кошельку
Для того чтобы привязать к кошельку пользователя банк, необходимо получить список банков с помощью метода `getBanks()`. Для того чтобы использовать метод пользователь должен быть авторизован и у пользователя должен быть создан аккаунт. В противном случае метод верет ошибку в Result. Далее из списка полученных банков пользователь выбирает необходимый. После чего на сервер отправляется запрос на привязку банка с помощью метода `addNewBank(bankId)`. Если сервер ответил успехом, то пользователю предлагается перейти в мобильное приложение банка, и завершить процесс привязки банка.

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

### Проверка статуса привязки банка к аккаунту пользователя
Так как привязка банка осуществляется пользователем в стороннем мобильном приложении, то при вызове метода `addNewBank` не возвращается результат привязки банка. Для того чтобы проверить статус нужно запросить аккаунт пользователя с помощью метода `getAccount`. Ориентируясь на список привязанных банков и id банка, который привязывается к аккаунту можно понять в каком статусе находится привязка банка. Если банк присутствует списке привязанных банков, но поле `isLinked` у банка равно `false` - это значит что банк в банке в процессе привязки к кошельку. В случае если поле равно `true`, значит процесс привязки банка успешен. В случае если в списке банков нет искомого, то это означает что произошла ошибка при привязке банка.

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
После привязки банка он становится банком по умолчанию. Для того чтобы изменить банк по умолчанию следует воспользоваться методом `setDefaultBank`. Для того чтобы использовать метод пользователь должен быть авторизован и у пользователя должен быть создан аккаунт, а так же привязан как минимум один банк. Для начала следует получить данные аккаунта методом `getAccount`, предложить пользователю выбрать из списка привязанных банков тот, который он хочет сделать основным. Далее передать `id` этого банка в метод `setDefaultBank(bankId)`. В результате чего в случае успеха метод вернет измененный и актуальный список привязанных банков.

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
Для быстрого создания строки, кодируемой в qr-код, достаточно использовать метод `generateQrString(loyaltyId: String)`.
Метод не требует наличия интернета, если есть предзагруженные otp-коды.
(Otp-коды нужны также для генерации строки и SDK их самостоятельно (без действий на стороне МП) запрашивает на бэке "в фоне", когда есть интернет).
Есть возможность "попросить" SDK в явном виде получить новую пачку otp-кодов для генерации QR-кода, метод
 `requestOtpCode(amount)`. Для использования методов пользователь должен быть предварительно авторизован и иметь аккаунт в системе.
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
Для платежного сервиса в сценариях необходимы формы для получения информации о пользователе. Формы динамические, то есть содержимое (состав полей) меняется в зависимости от контекста (сценария). Например, для привязывания счета необходимо заполнить форму упрощенной идентификации.
Получение формы происходит через метод `getForm(type: String?, formId: Int?, version: Int?)`. Если запрос формы произойдет без `formId`, то сервер пришлет чистую форму.
Если запрос происходит с `formId`, то SDK отправит 2 запроса на сервер:  `/form` и `/request`. Первый запрос получит форму, второй получит статус валидации для полей. SDK соединит полученные результаты и вернет форму с провалидированными полями.

*Примечание: количество полей и их состав сервер может поменять на своей стороне, форма является динамической.*

```
fun getForm(type: String?, formId: Int?, version: Int?) {
    Processing.getForm(type: type, formId: formId, version: version) { result in 
        switch result {
        case .success(let forms): 
            processSucess(forms)
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


### Получение истории операции для пользователя
Получить историю операций для пользователя можно с помощью метода `getOperations`. В данный метод заложены параметры для фильтрации операций. Бек вернет список операций, удовлетворяющих параметрам фильтрации.
Для первого запроса истории операции не нужно передавать курсор, в последующих запросах необходимо передавать курсор. Бек вернет курсор в ответе на первый запрос, если пользователь имеет больше операций, чем бек может отдать за один запрос.
Курсор (`cursor`) используется для пагинации операций.

```
func getOperations(cursor: String?, status: KDPayOperationElement.Status?, completion: @escaping (Result<KDPayOperations, KDPayError>) -> Void) {
    Processing.getOperations(
        cursor: cursor,
        type: nil,
        partner: nil,
        datetimeStart: nil,
        datetimeFinish: nil,
        status: .init(rawValue: status?.rawValue ?? "")) { [weak self] result in
            DispatchQueue.main.async {
                switch result {
                case .success(let value):
                    completion(.success(.init(value)))
                case .failure(let error):
                    let appError = KDPayError(error: error)
                    self?.handleError(error: appError)
                    completion(.failure(appError))
                }
            }
        }
}
```


### Получение конкретной операции для пользователя
Получить операцию для пользователя можно через метод `getOperation`. Важно, что id операции должен совпадать с типом операции, иначе операция будет считаться на найденной.

```
func getOperationInfo(
    id: Int,
    type: KDPayOperationElement.OperationType,
    completion: @escaping (Result<KDPayOperationElement, KDPayError>) -> Void) {
        Processing.getOperationInfo(id: id, type: type.asProcessingType) { [weak self] result in
            DispatchQueue.main.async {
                switch result {
                case .success(let value):
                    completion(.success(.init(value)))
                case .failure(let error):
                    let appError = KDPayError(error: error)
                    self?.handleError(error: appError)
                    completion(.failure(appError))
                }
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


##### Проверка доступа к кошелькам. Имеет кеширование. Возвращает результат предыдущего запроса, параллельно выполняя новый запрос. Если интервал не задан, кеш считается валидным 1 час.
`getUserExperiment(phone: String, completion: @escaping (Result<ProcessingUserExperiments>) -> Void)`
**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| UserExperiments      | Нет       | Вернет список экспериментов для данного пользователя |

##### Проверяет авторизован ли пользователь в системе по таким признакам как наличие в кеше идентификатора сессии и пользователя. А так же наличию публичного ключа сервера.
`isAuthorized: Boolean`
**Возвращает**

| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| Boolean| Нет| true - пользователь авторизован, false - не авторизован (Нет идентификатора сессии, пользователя, либо публичного ключа сервера в кеше SDK)|


##### Первый шаг авторизации пользователя в системе KD Pay. Служит для запроса смс кода на указанный в параметрах модуль.
`login(phone: String, userId: Int? = nil, basePhone: String?, deviceId: String?, name: String?, loyaltyId: String?, sessionCompletion: @escaping (String) -> Void, completion: @escaping LoginResponseCompletion)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| userPhone| String| нет | Телефон указанный пользователем |
| userName| String| да | Имя пользователя |
| deviceId| String| да | Идентификатор пользователя в системе.  Если не указан используется 64-битное число (выраженное в виде шестнадцатеричной строки), уникальное для каждой комбинации ключа подписи приложения, пользователя и устройства.  |

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
`createAccount(referrerId: String?, completion: @escaping (Result<WalletAccountResult>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| referrerId| String| да | Идентификатор реферальной системы |

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| WalletAccountResult | Нет| Ответ содержащий аккаунт пользователя в системе и текущий статус аккаунта. |


##### Получение аккаунта пользователя в системе KD Pay.
`getAccount(cached: Bool, referrerId: String? = nil, isShort: Bool, completion: @escaping (Result<UserResponse?>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| cached| Boolean| нет | Флаг для определения откуда брать данные аккаунта. true - из кеша, false - из сервера с последующим кешированием.|
| referrerId| String| да | Идентификатор реферальной системы |
| isShort| Bool| нет | Короткий вариант получения данных. Если указан `short`, то будут получены только основные данные по аккаунту. |


**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| WalletAccountResult | Нет| Ответ содержащий аккаунт пользователя в системе и текущий статус аккаунта |

##### Получение списка банков, которые можно привязать к кошельку.
`getBanks(completion: @escaping (Result<[ProcessingBank]>) -> Void)`
**Параметры**


**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| List<Bank> | Нет| Cписок банков, которые можно привязать к кошельку. |


##### Метод для установки банка по умолчанию. Выбирается из списка привязанных банков.
`addNewBank(bankId: Int, completion: @escaping (Result<Void>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| bankId| Int| нет |Идентификатор банка в системе KD Pay, выбранных из списка банков получаемых с помощью метода getBanks()|

**Исключения**
| Тип      |  Описание |
| ----------- |  ----------- |
| ErrorMessage.Client.NotAuthorized() | Сессия отсутствует или устарела |
| ErrorMessage.Client.Authorization() | Отсутствует идентификатор пользователя в системе. Необходима авторизация.  |
| ErrorMessage.Client.NoAccount() | Нет аккаунта пользователя |

##### Метод для удаления привязанного банка
`deleteBank(bankId: Int)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| bankId| Int| нет |Идентификатор банка в системе KD Pay, выбранных из списка банков получаемых с помощью метода getBanks()|

**Исключения**
| Тип      |  Описание |
| ----------- |  ----------- |
| ErrorMessage.Client.NotAuthorized() | Сессия отсутствует или устарела |
| ErrorMessage.Client.Authorization() | Отсутствует идентификатор пользователя в системе. Необходима авторизация.  |
| ErrorMessage.Client.NoAccount() | Нет аккаунта пользователя |

##### Метод для установки банка по умолчанию. Выбирается из списка привязанных банков.
`setDefaultBank(bankId: Int, completion: @escaping (Result<[ProcessingBank]>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| bankId| Int| нет | Идентификатор банка в системе KD Pay, выбранных из списка привязанных банков |


**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| List<Bank> | Нет| Список привязанных банков, с измененным банком по умолчанию. |

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
| amount| Int| нет | Количество запрашиваемых otp кодов.  По умолчанию 10.|

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
`getForm(type: String?, requestId: String?, version: String?, completion: @escaping (Result<ProcessingForm>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| type| String| да | Тип формы, которая задает определенный набор полей. |
| formId| Int| да | Идентификатор последнего актуального заполнения. |
| version| Int| да | Версия последнего актуального заполнения |

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

##### Метод проверки версии бибоиотеки. Имеет кеширование. Возвращает результат предыдущего запроса, параллельно выполняя новый запрос.
`checkLibraryVersion(cacheInterval: TimeInterval, completion: @escaping (Result<ProcessingVersionInfo>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| cacheInterval| TimeInterval aka Double| нет | Интервал проверки кэша в секундах. Если не выставлен интервал, то кеш на 1 час. |

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| ProcessingVersionInfo | нет | Возвращает результат проверки.  |

 /**
     * уведомляет сервер о показанных оповещениях ползьзователю
     * @param shownSnacks id и timestamp показанного оповещения
     * @return список id оповений с актуальным статусом
     */

##### Уведомляет сервер о показанных оповещениях ползьзователю. Возвращает список id оповещений с актуальным статусом
`snackShown(snacks: [SnackShownRequest], completion: @escaping (Result<SnackShownResponse>) -> Void)`
**Параметры**

| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| snacks| [SnackShownRequest] | нет | id и timestamp показанного оповещения |

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| SnackShownResponse | нет | Список id оповений с актуальным статусом |


#### Получение списка операций для пользователя. Возвращает курсор на предыдущий/следующий набор операций, если такие имеются.
`getOperations(
        cursor: String?,
        type: OperationElement.OperationType?,
        partner: String?,
        datetimeStart: String?,
        datetimeFinish: String?,
        status: OperationElement.Status?,
        completion: @escaping (Result<Operations>) -> Void)`
 **Параметры**
 | Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| cursor | String | да | указатель на получение списка операций |
| type | OperationElement.OperationType | да | тип операции |
| partner | String | да | фильтр партнера |
| datetimeStart | String | да | дата начала фильтрации операции |
| datetimeFinish | String | да | дата конца фильтрации операции |
| status | OperationElement.Status | да | статус операции |

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| Operations | нет| Список операций с предыдущим/следующим указателем |


#### Получение операции для пользователя
`getOperationInfo(id: Int, type: OperationElement.OperationType, completion: @escaping (Result<OperationElement>) -> Void)`
 **Параметры**
 | Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | String | нет | идентификатор операции |
| type | OperationElement.OperationType | нет | тип операции |

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| OperationElement | нет| Элемент операции |


#### Получение информации о кешбеке. Имеет перегрузку метода с параметром ключ-значения для гибкой настройки параметров.
`getCashbackInfo(prevMonth: Int, scenario: String, collapsed: Bool, source: String, completion: @escaping (Result<CashbackInfo>) -> Void)`
**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| prevMonth | Int | нет | период запроса данных |
| scenario | String | нет | место запроса данных |
| collapsed | Boolean | нет | скрытие виджетов в разделе |
| source | String | нет | источник перехода |

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| CashbackInfo | нет| Информацию о кешбеке |

####Перегрузка метода информации о кешбеке
`getCashbackInfo(params: [String: String], completion: @escaping (Result<CashbackInfo>) -> Void)`

**Параметры**
| Имя      | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| parameters | [String: String] | нет | параметры в формате ключ-значение |

**Возвращает**
| Тип      | Опциональный | Описание |
| ----------- | ----------- | ----------- |
| CashbackInfo | нет| Информацию о кешбеке |


##### Полная очистка кеша sdk. Удаляются идентификаторы сессии, пользователя, публичный ключ сервера, отп коды и номер телефона пользователя. Параллельно на сервер отправляется запрос на сброс всех активных сессий.
`logout()`


# Структура данных

#### `WalletAccountResult`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| walletAccountState | ProcessingWalletAccountState | нет | Описывает состояние аккаунта |
| walletAccount | ProcessingWalletAccount | да | Модель данных кошелька |
| banners | BannerAccount | да  | Список баннеров |
| messages | MessagesAccount | да  | Информационное сообщение о работе сервера |
| snacks | Snacks | да  | Список снекбаров |

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
| banks | [ProcessingBank] | нет  |     Список привязанных к кошельку банков |
| forms | ProcessingFormAccount? | да  | Список форм для упрощенной идентификации |


#### `ProcessingWalletStatus ` enum
| Имя | Описание |
| ----------- | --------|
| undefined | Неизвестное состояние кошелька |
| active | Кошелек активен и готов к работе |
| blocked | Кошелек заблокирован |
| closed | Кошелек закрыт |

#### `ProcessingBannersResponse`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| type | Stirng | нет | Вариант отображения баннеров, например списком, каруселью и т.п. |
| items | [ProcessingBannerItems] | да | Список баннеров |

#### `ProcessingBannerItems`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор баннера |
| type | String | нет | Тип баннера |
| params | [ProcessingBannerParams] | нет | Список параметров баннера |

#### `ProcessingBannerParams`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| name | String | нет | Имя параметра |
| value | String | нет | Значение параметра |
| textColor | String | да | Цвет значения (текста) параметра |
| type | String | да | Тип параметра |
| backgroundColor | String | да | Цвет фона параметра |

#### `ProcessingServiceStatus`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| badStatus | ProcessingServiceStatusElement | да | Сообщение о некорректной работе сервера |

#### `ProcessingServiceStatusElement`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| title | String | нет | Заголовок сообщения |
| description | Stirng | нет | Описание сообщения |

#### `ProcessingTimeZoneElement`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| title | String | нет | Заголовок сообщения  |
| text | String | нет | Описание сообщения |
| button | String | нет | Текст для кнопки |

#### `ProcessingSnackResponse`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| type | String | нет | Вариант отображения |
| items | [ProcessingSnackItems] | нет | Список снеков |

#### `ProcessingSnackItems`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор снека |
| type | Type | нет | Тип снека |
| params | [ProcessingParamsResponse] | нет | Параметры снека |

#### `ProcessingParamsResponse`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| name | String | нет | Имя параметра |
| value | String | нет | Значение параметра |

#### `ProcessingBank`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор банка в системе  |
| name | String | нет | Имя банка в системе  |
| imageLink | String | да | Ссылка на логотип банка  |
| isLinked | Bool | нет (false по умолчанию) | Состояние привязки банка. Если банк присутствует в списке, но его статус isLinked == false, это означает что банк в процессе привязки.   |
| isDefault | Bool | нет (false по умолчанию) | Банк выбран основным. С счета этого банка будут сниматься средства при платежах кошельком.  |
| isRetryPossible | Boolean | нет (false по умолчанию) | Возможность отправить запрос на привязку заново. |
| instructionUrl | String | да | Ссылка на инструкцию по привязке |

#### `ProcessingServiceStatus`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| badStatus | BadStatusServer | да | Сообщение о нестабильности сервера  |
| badTime | BadTime | да | Сообщение о некорректных настройках даты, времени и часового пояса  |

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
| nextFieldId | Int | да | Идентификатор следующего поля |
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

#### `SnacksShownStatus`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор сообщения  |
| status | String | нет | Статус сообщения  |

#### `Operations`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| operations | List<OperationElement> | нет | Список элементов операции|
| nextCursor | String | да | Указатель на следующий список операций |
| prevCursor | String | да | Указатель на предыдущий список операций |

#### `OperationElement`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор операции |
| timestamp | Long | нет | Дата и время операции |
| type | OperationsElement.OperationType | нет | Тип операции |
| status | OperationsElement.Status | нет | Статус операции |
| name | String | нет | Название операции – русский аналог type |
| imageUrl | String | нет | Изображение операции |
| merchant | String | нет | Торговец операции |
| from | FromAndToElement | нет | Откуда произведена операция |
| to | FromAndToElement | нет | Куда произведена операция |
| amount | String | нет | Сумма операции |
| currency | CurrencyElement | нет | Валюта операции |
| childOperations | List<OperationItem> | нет | Информация о дочерних операциях |
| parentOperations | List<OperationItem> | нет | Информация о родительских операциях |
| bill  | String | да | законопроект |
| externalLink  | String | да | внешняя ссылка |
| moneyFlowDirection  | OperationElement.MoneyFlowDirection | нет | поступают деньги или уходят со счета пользователя |
| additionalImageId  | String | да | Дополнительное изображение операции (например значок СБП) |
| cancelDescription  | String | да | писание причины отмены |

#### `OperationElement.OperationType ` enum
| Имя | Описание|
| ----------- | ----------- |
| Purchase | покупка  |
| Invoice | me2me пополнение |
| BankLink | пополнение для привязки |
| Refill | пополнение инициированное пользователем из приложение его банка |
| Refund | возврат |
| Unknown | неизвестная операция, если не удалось распарсить ответ бека |

#### `OperationElement.Status ` enum
| Имя | Описание|
| ----------- | ----------- |
| Canceled | отменена  |
| InProcess | в обработке |
| Success | выполнена |
| Unknown | неизвестный статус операции |

#### `OperationElement.MoneyFlowDirection ` enum
| Имя | Описание|
| ----------- | ----------- |
| Income | деньги поступают на счёт пользователя в монете - в приложении знак "+" |
| Outcome | outcome - деньги уходят со счёта пользователя в монете |
| Unknown | неизветсный статус |

#### `OperationItem`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| id | Int | нет | Идентификатор операции |
| type | OperationsElement.OperationType | нет | Тип операции |
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
| id | String | да | Идентификатор валюты |
| symbol | String | да | Символ валюты |

#### `FromAndToElement`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| name | String | нет | Торговец операции |
| description | String | да | Описание торговца |

#### `CashbackInfo`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| collapsed | Boolean | да | скрытие виджетов в разделе |
| widgets | [WidgetsData] | да | список виджетов |

#### `WidgetsData`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| name | String | да | имя |
| id | Int | да | идентификатор |
| collapsable | Boolean | да | идентификатор |
| text | String | да | текст |
| sum | String | да | сума |
| currency | String | да | валюта |
| params | [WidgetsParamData] | да | параметры |
| items | [WidgetsItemData] | да | элементы |
| left | WidgetParamLeft | да | левый элемент |
| right | WidgetParamRight | да | правый элемент |
| pointerDirection | WidgetPointerDirection | да | направление |
| currentMonth | Int | да | текущий месяц |
| onClick | CashbackOnClick | да | действие по нажатию |

#### `WidgetsParamData`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| name | String | да | имя |
| id | Int | да | имя |
| type | String | да | тип |
| value | String | да | значение |
| onClick | CashbackOnClick | да | действие по нажатию |
| fontColor | String | да | цвет текста |
| text | String | да | текст |
| sum | String | да | сума |
| currency | String | да | валюта |
| currentMonth | Int | да | текущий месяц |

#### `WidgetsItemData`
| Имя свойства | Тип | Опциональный |Описание|
| ----------- | ----------- | ----------- |--------|
| prefix | String | нет | префикс |
| sum | String | нет | сумма |
| currency | String | нет | валюта |
| cashback | String | нет | кешбек |
| fullness | Int | нет | заполненность |

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
| value | String | да | значение |
| type | String | да | тип |
