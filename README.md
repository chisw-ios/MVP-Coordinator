# MVP+Coordinator

Скелет проекта на основе модульной архитектуры MVP + C. В основу заложена микросервисная идея, где каждый юзер флоу будет отдельным сервисом. 
В основе работы всех депенденсей лежит принцип DependencyInjection. 
Директории в проекте делиться на следующие слои:
* ApplicationLayer - с самыми основными сервисами
* BusinessLogicLayer - где у нас размещаются модели для нетворкинга и базы данных, а также сервисы для каждого флоу. 
    * Models - директория с моделями для нетворкинга и базы данных
    * Services - директория для сервисов и основного класса Services (некий сервис локатор)
        * AdditionalServices - общие дополнительные сервисы (BiometricsService, FileService)
        * DataBase - подготовленная сетап базы данных - RealmSwift
        * ModuleServices - сервисы для работы с отдельными флоу
        * Networking - директория с классами для нетворкинг коммуникации
* CoreLayer:
    * Autogenerated - автогенерируемые файлы ресурсов c помощью SwiftGen (локализация, цвета, картинки, шрифты) 
    * BaseClases - базовые класы (cell, controller)
    * Configurations - файлы конфигурации для настроек разных схем (development, staging, production)
    * Constants - для хранения глобальных констант
    * Coordinator - с базовыми классами для координатора и роутинга.
    * Extension - разнообразные полезные расширения для классов 
    * Generics - дженерики (например для работы с датосорсами таблиц и коллекций)
    * Helpers - разнообразные помощники (Keychain, Logger, Utils)
    * Transitions - для обработки кастомных переходов и анимаций
* PresentationLayer - слой для всех флоу с модулями и контроллерами
* Resources - для хранения основных ресурсов (локализации, шрифтов, картинок, цветов 🌷)
* Supporting Files - системные, сопровождающие файлы

![Folder Skeleton](/Assets/folderSkeleton.png)

Жизненный цикл приложения начинается с SceneDelegate, где стартует основная сущность DependencyProvider, которая запускает все наши сервисы и основной координатор. (ApplicationLayer/) 
Паттерн координатора позволяет облегчить и декопозировать роутинг всех экранов внутри приложения. (CoreLayer/Coordinator/)
Основной координатор ApplicationCoordinator, с помощью LaunchInstructor определяет точку входа приложения, на LoginFlow или MainFlow и так далее. 
Для примера в демо проекте есть два флоу Auth и Main, и соответственно сервисы AuthService и UserService (BusinessLogicLayer/Services) которые обслуживают флоу. 
Класс Services (BusinessLogicLayer/Services) отвечает за инициализацию нетворкинга, хранилища и сервисов для всех флоу. 

Каждый модуль состоит из стандарта архитектуры MVP (Model-ViewController-Presenter) и в дополнению к модулю или userFlow идет свой координатор.
Есть основной ApplicationCoordinator который содержит в себе под координаторы. Каждый из них наследуется от BaseCoordinator, который содержит в себе массив всех координаторов. 
Для того что бы собрать основные координаторы для точек входа его необходимо описать в протоколе CoordinatorFactoryProtocol
Внизу есть схема которая показывает каким образом у нас разветвляются подкоординаторы.

![Main Coordinator](/Assets/coordinatorMain.jpg)

Для удобства создания MVPC модулей в корневой директории есть темплейт Xcode_Template_MVPC. Добавляем и пользуемся.
Далее обсудим PresentationLayer.
Разберем модуль на примере Auth модуля: 
* Auth.storyboard - рекомендуется использовать не больше 5 экранов в одном файле. Если больше создаем новые сториборды с связями-релейшенами, во избежании конфликтов при слиянии веток. Основное правило: один человек - один флоу - один сторибоард.
* AuthViewControllerFactory - некий ассемблер, билдер или фабрика, которая ответственна за сборку модулей (контроллеров с презентами)
* AuthCoordinator - ответственный за роутинг между всеми экранами модуля по средством передачи событий из контроллера в презентер, а затем в координатор. Идея основана на блоках, описание работы чуть ниже в презентере. 
* Login submodule: 
    * LoginViewController - отвечает за работу с экраном (UI, ViewLifeCycle)
    * LoginPresenter - обработка бизнес логики из сервисов, подготовка данных для отображения
        * Доступ к каждому презенту идет через протокол LoginPresenterProtocol, который описывает все действия которые контроллер может провести с презентом
        * Каждый презентер описывает роуты по которым может ходить модуль LoginPresenterRoutes. Это структура с блоками, которую мы инициализируем в координаторе и там же и проводит роутинг.
        
Ниже приведена схема каким образом могут быть связаны экраны с координаторами. 

![Coordinator modules](/Assets/coordinatorModule.jpg)

В качестве менеджера зависимостей используется Cocoapods. Как вариант можно воспользоваться альтернативой SPM. Но к сожалению некоторые зависимости все еще не поддерживают SPM.

Отдельно хотелось бы остановится на директории Networking. За основу используем Alamofire.
* EndPointType - протокол конструктор для запросов
* Networker - класс с описанием основных запросов (MultipartData, RequestInterceptor, parsers)
* Requester - директория с фабриками реквестов для каждого из флоу. Разбивается на отдельные фабрики во избежании массивных конструкций.
    * AuthRequester - фабрика реквестов для регистрации, авторизации, забыл пароль и тд.
    * И так каждый флоу по отдельности
* TokenStorage - сервис для хранения токенов и secure информации (KeychainSwift)
Модели для нетворкинга парсятся по Codable и лежат в директории BusinessLogicLayer/Models


В проекте настроены три схемы для сборки приложений с разными ключами и входящими данными. 
Зачем это нужно? Например для разных окружений может использоваться разные урлы для бека, бандлы, названия приложения, а так же разнообразные ключи и токены для разных сервисов. 
Все это можно настроить через конфигурационные файлы в директории CoreLayer/Configurations

![Schemes](/Assets/schemes.png)

Подытожив выделяем основные правила для маштабирования архитектуры. 
* Стараемся следовать принципам SOLID.
* Каждый сервис должен иметь свою зону отвественности: например Auth сервис будет отвечать за работу конкретно с авторизацией, регистрацией и тд. User сервис будет отвечать конкретно за работу с данными юзера, не важно нетворкинг это или база данных. 
* Каждый координатор работает только со своим флоу, размер файла не превышает 500 строк. В противном случае создаем SubCoorinator по прицнипу работы основного координатора со своими сабкоординаторами.
* Следуя декомпозиции стараемся не перегружать сервисы, презентеры и контроллеры. Где можно разбиваем на отдельные сущности и сервисы. 
* Для приватности отдельных классов работаем через протоколы. 
