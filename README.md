
## Постановка задачи
Требования к заданию изложены [здесь](./TaskDescription.md)
## Описание архитектуры решения
Ниже представлена компонентная схема приложения<br>
Это классическая трехслойная архитектура Spring Web MVC, состоящая из трех ключевых компонентов: Controller, Service, 
Repository.<br>  
![](./appscheme.svg)

<b>OperationsController</b>:

- обеспечивает экспозицию необходимых эндпонитов для обращения к ним клиентского приложения (FRONT),
- извлекает из получаемых HTTP запросов данные, необходимые для выполнения перевода денег с карты на карту (транзакций), 
- вызывает и получает ответы о результатах вызовов из компонента OperationService,
- формирует HTTP ответ во FRONT на основании данных о выполнении/ошибке выполнения транзакции, полученных от 
OperationService, а так же в случае возникновения ошибки в других компонентах приложения.<br>
OperationsController и OperationsService обмениваются друг с другом данными в виде DTO, подготовленных для всех типов 
запросов и ответов от одного сервиса к другому.

<b>OperationsService</b>:
- формирует и выполняет запрос на выполнение транзакции к "внешнему" сервису IPSP API. 
- получает от внешнего сервиса IPSP API результат выполнения транзакции и формирует ответ контроллеру OperationsController
в требуемым им формате.<br> 
- формирует и выполняет запрос к OperationsRepository для записи транзакции в лог транзакций в памяти и записи ее в
файл журнала транзакций.
Выполнение транзакций состоит из двух этапов, выполняемых последовательно, каждый из которых реализован отдельным методом.
Первый (`OperationService.transfer`) - это обработка первоначального запроса на авторизацию транзакции, второй -
(`OperationService.confirmOperation`) - это обработка запроса на верификацию кода подтверждения от клиента, в случае
успешного завершения первого этапа.<br>Сервис IPSP API, точнее его заглушка, реализован компонентом `AcquiringService`.
Он эмулирует поведение реального платежного сервиса, чтобы наше приложение можно было запустить и проверить его работу
в режиме prod (по сути - это mock).

<b>OperationsRepository</b>:
- записывает транзакцию в лог транзакций в памяти, а также предоставляет методы для получения транзакции по ее id из
лога и метод для обновления статуса транзакции.
- записывает транзакцию в файл журнала транзакций. <br>

В дополнении к основным компонентам описанным выше, реализован компонент LogAspect. Он определяет правила генерации 
и формирования содержания сообщений выводимых в лог консоли во время работы приложения в дополнении к стандартному 
логу консоли Spring Boot. Так же с помощью настройки в `application.properties` настроена запись лога консоли в файл.

Полный набор данных карт, достаточный для проведения переводов, нигде в приложении не хранится. В логе транзакций в
памяти хранятся только: идентификатор операции, номера карт, дата и время транзакции, сумма, валюта, результат согласно
полям класса `Transaction`. В файле журнала транзакций записывается та же информация в формате csv:<br>
`3,2022-04-05T01:13:19.256945500,1111-1111-1111-1111,1111-1111-1111-1111,100000.00,RUR,success`<br>
Полная информация о передаваемых в приложение данных карт выводится только в логе консоли и только если приложение 
собрано/запущено в режиме отладки, то есть со значением свойства `application.profile.dev` = `true`.

Ниже представлена диаграмма последовательностей процесса обработки транзакции:

![](./sequencediag.svg)
## Описание сборки и запуска приложения
### Сборка и запуск клиентского Web приложения (FRONT)
Согласно условиям задания, клиентское приложение (разработанное за рамками данного проекта) должно быть запущено
в docker контейнере используя docker-compose. Ниже описано создание и процедура запуска FRONT приложения:
- Выполнить шаги по клонированию репозитория [FRONT](https://github.com/serp-ya/card-transfer). 
Если планируется также запускать FRONT непосредственно в вашей host OS, необходимо выполнить построение 
зависимостей на вашей host OS. Это быстрее и удобнее делать UNIX системе, так как в этом случае достаточно установить
Node.js и "собрать" приложение. В случае же с OS Windows, для сборки необходимо обеспечить на машине набор 
"build tools", в который входит python и Visual Studio Build Tools, поскольку эти инструменты необходимы
на OS Windows для компиляции модулей Node.js. Эти инструменты устанавливаются достаточно долго и занимают на диске 
около 3G. 
- Скачать docker image Node.js. В проекте использован официальный образ `14-alpine3.15`.
- Скопировать `DockerfileFRONT` и `.dockerignore` в папку проекта фронта - /card-transfer/ (содержащую файл 
`package.json`). Файл `.dockerignore` исключает копирование в образ c host OS директории /node_modules/, содержащей 
уже скомпилированные модули.
- Собрать docker образ выполнив команду<br> 
`docker build -f DockerfileFRONT -t card-transfer-app .`<br> 
Полученный образ можно запустить без docker-compose командой<br> 
`docker run -it --rm -p 3000:3000 --name card-transfer card-transfer-app`<br>
### Сборка и запуск RESTful сервиса, разработанного в рамках данного проекта
- Установить значение `application.profile.dev=true` в файле `application.properties`
- Собрать приложение, выполнив `mvn clean package`
- Перейти корневую папку проекта и собрать docker образ выполнив команду в корневой папке проекта<br>
`docker build -f DockerfileRESTAPP -t rest-service-app .`<br>
Полученный образ можно запустить без docker-compose командой<br>
`docker run -it --rm -p 5500:5500 --name rest-service-app rest-service-app`<br>
### Запуск обоих приложений в docker-compose
Для запуска обоих приложений в docker-compose выполнить команду:<br>
`docker-compose up`<br>
(Файл с описанием docker-compose находится в корневой папке проекта `docker-compose.yml`)
## Описание тестов
Для тестирования приложения разработаны unit тесты, проверяющие работу методов с использованием mock, 
а так же интеграционные тесты с использованием testcontainers. Интеграционные тесты используют docker образ 
приложения.<br>
Интеграционные тесты находятся в классе `MoneyTransferServiceApplicationTests.java`. Примеры
запросов в формате вcтроеного в Intellij IDEA инструмента выполнения запросов собраны в файле `test-requests.http`
в папке test.






