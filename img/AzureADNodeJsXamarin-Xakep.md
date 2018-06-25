# Сквозная аутентификация мобильного приложения и backend

В современных условиях использование мобильных устройств для доступа к корпоративным системам является востребованным и уже абсолютно типовым сценарием. Например, сотрудник, находясь в офисе или же за пределами корпоративной сети, должен иметь возможность с мобильного телефона оперативно получить данные о последних сделках, состоянии склада, статусе выполнения заказа и пр. В многих случаях подобная информация хранится в backend-системах внутри периметра предприятия. Соответственно, возникают задачи:

1. аутентификации корпоративного пользователя в мобильном приложении;
2. обеспечения прозрачного доступа пользователя, успешно прошедшего проверку, к необходимым backend-системам.

В одной из [предыдущих](https://xakep.ru/2018/06/08/azure-mfa/) публикаций рассматривалось решение первой задачи на основе [Azure Active Directory](https://docs.microsoft.com/en-us/azure/active-directory/active-directory-whatis) (Azure AD) для мобильных приложений, созданных с помощью [Xamarin Forms](https://www.xamarin.com/forms). Здесь же мы обсудим реализацию второй задачи – защищенного доступа к backend с использованием результатов аутентификации из п. 1.

Действующие лица и исполнители:

- В качестве системы аутентификации и управления токенами, как уже очевидно, будет выступать Azure AD. Предполагается, что для каждого пользователя, нуждающегося в мобильном доступе, в Azure AD создана учетная запись. Если необходимо использовать учетные записи локальной службы AD предприятия, можно настроить различные [варианты синхронизации](https://docs.microsoft.com/en-us/azure/active-directory/choose-hybrid-identity-solution) между AD и Azure AD.
- Мобильным приложением послужит пример кода на Xamarin из упомянутой выше статьи. Там же вы найдете все шаги по регистрации приложения в Azure AD.
- Backend-система будет представлять собой простой REST API сервис, написанный на Node.js и запущенный на CentOS. Это сервис поддерживает в базе данных MongoDB некий список абстрактных задач (а по сути, просто строк) и реализует три операции со списком: получить список задач для пользователя (listTasks), добавить задачу (createTasks), удалить задачу (removeTasks).

Как и для Xamarin в случае мобильного приложения, для Node.js доступна реализация библиотеки [Azure Active Directory Authentication Libraries](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-authentication-libraries) (ADAL), которая возьмет на себя всю рутину по взаимодействию с Azure AD, как то: защищенное обращение к облачным endpoints, проверка сертификатов, вызовы сервисов многофакторной аутентификации, если необходимо, и пр.


## Процесс аутентификации на базе Azure AD

Существует [несколько](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-authentication-scenarios#native-application-to-web-api) сценариев аутентификации с использованием Azure AD. В данном случае нас интересует вариант доступа нативного приложения к Web API, который описывается следующей диаграммой:

![Authentication flow](https://github.com/ashapoms/AzureADNodeJsXamarin/blob/master/img/native_app_to_web_api.png)

Однако при использовании библиотек подобным ADAL некоторые шаги скрыты от разработчика, поэтому ниже я дам несколько упрощенное описание, которое тем не менее отражает основную логику происходящего. Итак:

1. Пользователь запускает приложение на своем смартфоне. Это приложение обращается к Azure AD и как бы говорит: «Я – такое-то приложение, вот мой Application ID (он же по-другому называется Client ID). Дай мне токен для доступа к Web API вот с таким Application ID». Эти Application ID генерируются в процессе регистрации мобильного приложения и Web API в Azure AD.
2. Первое, что делает Azure AD, получив такой запрос, аутентифицирует пользователя, запустившего приложение. На экране телефона появляется окно для ввода логина и пароля. Если на пользователя распространяется политика многофакторной аутентификации, после корректного ввода пароля задействуется дополнительный фактор, например, на телефон высылается SMS с кодом.
3. После успешной аутентификации пользователя Azure AD проверяет, зарегистрированы ли в ее базе Application IDs мобильного приложения и Web API, иными словами, «знает ли» она эти приложения.
4. Если все в порядке, Azure AD проверяет, а разрешен ли доступ мобильному приложению к запрашиваемому Web API.
5. Если доступ разрешен, Azure AD формирует токен в формате JWT, в поле **audience** токена копирует содержимое поля **App ID URI** из регистрационных настроек Web API, подписывает токен и возвращает мобильному приложению.
6. Мобильное приложение обращается к Web API, поместив в Authorization header полученный от Azure AD токен. Поскольку токен подписан, но по умолчанию не зашифрован, предполагается, что взаимодействие между мобильным приложением и Web API осуществляется по HTTPS.
7. Получив запрос, Web API проверяет подпись и срок действия токена. Сертификат для проверки подписи доступен с публичного endpoint Azure AD.
8. Если токен в порядке, Web API сравнивает значение поля **audience** токена со своей конфигурационной информацией (файл *config.js* в нашем случае). Если найдено совпадение, Web API «понимает», что токен действительно предназначен для него, и что можно предоставлять запрашиваемую информацию.

Теперь давайте посмотрим, как эти шаги реализуются в коде и настройках Azure AD. Полностью, включая код мобильного приложения, пример доступен в моем [репозитории](https://github.com/ashapoms/AzureADNodeJsXamarin) на GitHub.


## Настройка сервиса REST API на Node.js

Оставляю за кадром процесс инсталляции Node.js и MongoDB, здесь нет никаких особенностей. Замечу лишь, что в рассматриваемом примере обе компоненты развернуты на одном хосте, и для MongoDB использовались установки и endpoints по умолчанию. Предполагаю также, что на хост с Node.js и MongoDB скопирован код из моего репозитория из папки [RestApiServer/node-server/](https://github.com/ashapoms/AzureADNodeJsXamarin/tree/master/RestApiServer/node-server).

Первым делом регистрируем Web API в Azure AD. Если нет учетной записи и доступа к [Microsoft Azure](https://azure.microsoft.com/), можно легко получить бесплатную пробную учетную запись [здесь](https://azure.microsoft.com/ru-ru/free/). В рамках пробной подписки большое число сервисов предоставляется бесплатно на год.

На портале управления Azure заходим в **Azure Active Directory** -> **App registrations** и щелкаем **New application registration**.

![Application registration](https://github.com/ashapoms/AzureADNodeJsXamarin/blob/master/img/AzureADAppRegistration.PNG)

На странице **Create** даем приложению более-менее понятное имя, например *NodeJSBearerRestServer*, в поле **Application type** выбираем *Web app / API*, в поле **Sign-on URL** задаем  http://localhost:3000 (в нашем примере REST API слушает по порту 3000, можно конечно использовать другой порт) и нажимаем **Create**.

![Create WebApi](https://github.com/ashapoms/AzureADNodeJsXamarin/blob/master/img/CreateWebApiApp.PNG)

Регистрация выполнена. Для последующей настройки сервиса REST API нам потребуется некоторая информация из Azure AD. Щелкаем на только что созданном приложении, нажимаем **Settings**, копируем и где-нибудь сохраняем значения двух важных параметров: **Application ID**

![ApplicationID](https://github.com/ashapoms/AzureADNodeJsXamarin/blob/master/img/ApplicationID.PNG)

и **App ID URI**.

![AppIDURI](https://github.com/ashapoms/AzureADNodeJsXamarin/blob/master/img/AppIDURI.PNG)

Кроме того, нам пригодится идентификатор нашего каталога Azure AD, так называемый **Tenant ID** (или по-другому **Directory ID**). Найти его можно в разделе **Azure Active Directory** -> **Properties**.

![TenantID](https://github.com/ashapoms/AzureADNodeJsXamarin/blob/master/img/TenantID.PNG)

Теперь можно перейти непосредственно к приложению Node.js. В нашем примере сервис REST API построен с использованием Restify, а задачи аутентификации решаются с применением популярного middleware [Passport](http://www.passportjs.org/). Microsoft, в свою очередь, разработала [passport-azure-ad](https://github.com/AzureAD/passport-azure-ad) – плагин для Passport, реализующий интеграцию нодовских приложений с Azure AD. Это не единственный способ подружить Node.js с Azure AD, но, с моей точки зрения, наиболее простой и удобный.

Все настройки, связанные с Azure AD, находятся в файле *config.js*. Для нашего сценария принципиальны три параметра, которые мы и рассмотрим.

С помощью первого параметра **identityMetadata** мы говорим нашему приложению, как получить необходимые метаданные об Azure AD (вспомните диаграмму процесса аутентификации): где находится Authorization Endpoint, где находится Token Endpoint, откуда загрузить сертификат для проверки токенов и пр. Если Web API обслуживает запросы от приложений только одного тенанта, что характерно для корпоративной среды (одна компания – один тенант), то параметр должен выглядеть так:

```
identityMetadata: 'https://login.microsoftonline.com/<<Insert_your_tenant_ID_here>>/.well-known/openid-configuration'
```

В эту строку надо подставить идентификатор тенанта (Directory ID), скопированный на одном из предыдущих шагов. Если же ваш Web API является мультитенантным приложением, то значение должно быть иным:

```
identityMetadata: ' https://login.microsoftonline.com/common/.well-known/openid-configuration'
```

Во второй параметр **clientID** копируем значение **Application ID**, полученное при регистрации Web API в Azure AD.

```
clientID: '<<Insert client ID of your REST API Server here>>'
```

Строго говоря, этот шаг не принципиален, если к Web API обращается нативное приложение, потому что пользователь уже прошел аутентификацию с его помощью, и токен уже получен. Но если предполагается, что к серверу могут обращаться пользователи и через браузер, и такие запросы также должны быть аутентифицированы Azure AD, то в коде Web API необходимо предусмотреть редирект в Azure AD. На какой URL перенаправлять для аутентификации сервер уже знает из **identityMetadata**. Остается в редирект добавить свой **clientID**, чтобы Azure AD понимала, на какой URL вернуть пользователя после успешной проверки.

Выглядит это примерно так:
1. Azure AD получает редирект-запрос, в котором присутствует **clientID** перенаправившего Web API.
2. Azure AD отображает страницу аутентификации, пользователь вводит логин и пароль.
3. Если все в порядке, Azure AD находит в своей базе зарегистрированное приложение с **Application ID** равным полученному в запросе **clientID**, в настройках приложения находит поле **Reply URLs** и перенаправляет браузер на URL, указанный в этом поле.

Наконец, третий параметр, который нас интересует, audience. Сюда мы копируем полученное при регистрации значение **App ID URI**.

```
audience: '<<Insert App ID URI of your REST API Server here>>'
```

Напомню, получив токен от мобильного приложения и проверив его подпись и срок действия, сервер сравнивает значение поля **audience** из токена со значением, которое прописано в данном параметре. Если значения совпадают, значит токен действительно предназначен для сервера, а не какого-то иного приложения.

Теперь заглянем в *app.js*. Здесь, пожалуй, требуют пояснения два момента. Во-первых, при инициализации сервера следующий код создает стратегию аутентификации на основе рассмотренных параметров:

```
// Let's start using Passport.js

server.use(passport.initialize()); // Starts passport
server.use(passport.session()); // Provides session support

var bearerStrategy = new OIDCBearerStrategy(options,
    function(token, done) {
        log.info(token, 'was the token retreived');
        if (!token.oid)
            done(new Error('oid is not found in token'));
        else {
            owner = token.oid;
            done(null, token);
        }
    }
);

passport.use(bearerStrategy);
```

Во-вторых, вот так выглядят хендлеры, которые задействуют настроенную стратегию при каждом запросе к Web API, проверяя с ее помощью передаваемый в запросе токен.

```
// Handlers with protection
server.get('/api/tasks/:owner', passport.authenticate('oauth-bearer', {
    session: false
}), listTasks);
server.post('/api/tasks/:owner/:Text', passport.authenticate('oauth-bearer', {
    session: false
}), createTask);
server.del('/api/tasks/:owner/:Text', passport.authenticate('oauth-bearer', {
    session: false
}), removeTask);
```

Вся рутинная работа выполняется методами passport-azure-ad, вам об этом заботиться не нужно.

Теперь, когда все готово, можно запустить наш сервер. Для более удобочитаемого логирования я использую библиотеку [bunyan](https://github.com/trentm/node-bunyan).

```
$ node server.js | bunyan
```

Для работы с Web API необходимо обращаться на endpoint http://localhost:3000/api/tasks

Осталось посмотреть на настройки мобильного приложения и код, который, собственно, формирует запрос к Web API и предает токен.


## Настройка мобильного приложения Xamarin

Прежде всего, мобильное приложение тоже должно быть зарегистрировано в Azure AD. Эта процедура подробно описана в статье, на которую я уже неоднократно ссылался.

Далее необходимо предоставить разрешение мобильному приложению на доступ к Web API. Для этого в Azure AD заходим в свойства мобильного приложения, нажимаем **Settings** -> **Required permissions** -> **Add** -> **Select an API**. В открывшемся списке выбираем (можно воспользоваться поиском) *NodeJSBearerRestServer* (или то имя, которое вы присвоили вашему Web API при регистрации), нажимаем **Select**. В разделе **SELECT PERMISSIONS** ставим галочку напротив выбранного сервера, нажимаем **Select** и затем **Done**.

![AzureADPermissions](https://github.com/ashapoms/AzureADNodeJsXamarin/blob/master/img/AzureADPermissions.PNG)

Перейдем к коду приложения. Чтобы мобильное приложение понимало, для какого сервиса необходимо запрашивать токен в Azure AD и куда потом с этим токеном обращаться, используются два статических поля **graphResourceUri** и **RestUri** (файл *MainPage.xaml.cs*). В эти поля необходимо прописать соответственно **App ID URI** и endpoint только что настроенного и запущенного сервера.

![XamarinApp](https://github.com/ashapoms/AzureADNodeJsXamarin/blob/master/img/XamarinApp.PNG)

Получение токена происходит в методе *btnLogin_Clicked()*, который можно найти там же в файле *MainPage.xaml.cs*.

```
private async void btnLogin_Clicked(object sender, EventArgs e)
        {
            try
            {
                var auth = DependencyService.Get<IAuthenticator>();
                authResult = await auth.Authenticate(authority, graphResourceUri, clientId, returnUri);
                string userUpn = authResult.UserInfo.DisplayableId.ToString();
                int userAt = userUpn.IndexOf('@');
                userLogin = userUpn.Substring(0, userAt);
                var userName = authResult.UserInfo.GivenName + " " + authResult.UserInfo.FamilyName;
                lblUserName.Text = userName;
                lblMessage.Text = "Access Token: " + authResult.AccessToken.ToString();
                await DisplayAlert("User Login", userLogin, "Ok", "Cancel");
                AuthenticationContext ac = new AuthenticationContext(authority);
                var cachedTokens = ac.TokenCache.ReadItems();
                lblMessage.Text = "Cached Tokens: " + cachedTokens.ToString();
                btnLogout.IsEnabled = true;
            }
            catch (Exception ex)
            {
                await DisplayAlert("Error", ex.Message, "Ok");
            }
        }
```

Если аутентификация прошла успешно, полученный токен сохраняется в **authResult**.

Далее в коде можно найти три метода (*btnCallRestAdal_Clicked()*, *btnAddTaskRestAdal_Clicked()*, *btnDeleteTaskRestAdal_Clicked()*), которые получают, добавляют и удаляют задачи для конкретного пользователя соответственно. И во всех трех случаях обращение к Web API выглядит однотипно – формируется нужный URI, в заголовок запроса добавляется полученный токен, вот так:

```
client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", authResult.AccessToken);
```

После чего вызывается метод, реализующий требуемый запрос (GET, POST или DELETE). Например, для получения списка задач пользователя в методе *btnCallRestAdal_Clicked()* к URL сервера добавляется логин пользователя (его можно найти в токене) и вызывается метод *GetAsync()*.

```
        private async void btnCallRestAdal_Clicked(object sender, EventArgs e)
        {

            if (!(authResult is null))
            {
                client.MaxResponseContentBufferSize = 256000;
                var uri = new Uri(string.Format(RestUri + userLogin, string.Empty));
                client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", authResult.AccessToken);

                try
                {
                    var response = await client.GetAsync(uri);
                    if (response.IsSuccessStatusCode)
                    {
                        lblMessage.Text = await response.Content.ReadAsStringAsync();
                    }
                }
                catch (Exception ex)
                {
                    lblMessage.Text = $"ERROR: {ex.Message}";
                }
            }
            else
                lblMessage.Text = "Please press Login to Azure button first";
        }
```


## Тестирование сквозной аутентификации

Мы сконфигурировали и запустили сервис на Node.js, сконфигурировали мобильное приложение, и, если теперь его запустить, можно протестировать весь процесс сквозной аутентификации. Ниже приведены скриншоты, которые отражают логин пользователя и получение его списка задач.

![Mob01](https://github.com/ashapoms/AzureADNodeJsXamarin/blob/master/img/Mob01.PNG)
![Mob102](https://github.com/ashapoms/AzureADNodeJsXamarin/blob/master/img/Mob102.PNG)
![Mob103](https://github.com/ashapoms/AzureADNodeJsXamarin/blob/master/img/Mob103.PNG)
![Mob104](https://github.com/ashapoms/AzureADNodeJsXamarin/blob/master/img/Mob104.PNG)
![Mob105](https://github.com/ashapoms/AzureADNodeJsXamarin/blob/master/img/Mob105.PNG)

В заключение отмечу, получив токен, сервер может быть уверен, что запрашивающий ресурсы пользователь успешно прошел аутентификацию. Более того, сервер понимает, какой конкретно пользователь к нему обращается – имя пользователя и логин доступны в токене. А это означает, что дальше разработчик может реализовать в Web API любую логику авторизации для любой используемой подсистемы, будь то MongoDB, SQL Server, SAP или что-то еще.