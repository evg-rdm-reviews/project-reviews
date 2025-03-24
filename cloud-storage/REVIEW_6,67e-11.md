# Проект “Облачное хранилище файлов”. Андрей (tg: @MrShoffen).

[Код проекта](https://t.me/zhukovsd_it_chat/53243/182198)

[Тг пост](https://github.com/MrShoffen/cloud-storage-rest-api)

---

## Логирование

1. Реализовано с помощью AOP. Рабочий подход, но добавляет соответствующие накладные расходы, поэтому главное тут не злоупотреблять.
2. Pointcut-ы лучше вынести из `@Aspect` класса в отдельное место.
3. В подавляющем большинстве используется уровень INFO, что не есть хорошо - логирование не бесплатный инструмент.
    1. Например, это логи уровня DEBUG максимум:
        
        ```java
        @AfterReturning(value = "loadingUserFromRepoPointcut(username)", returning = "user")
        public void afterLoadingUser(JoinPoint joinPoint, String username, UserDetails user) {
            String className = joinPoint.getTarget().getClass().getSimpleName();
            String method = joinPoint.getSignature().getName();
            log.info("{} : {} :username[{}]: SUCCESS : User[{}] ", className, method, username, user);
        }
        ```
        
        Логи, связанные с какими-то проблемами - это ERROR или WARN (в одном месте ты использовал WARN как полагается). Причем по умолчанию ставим ERROR, а потом думаем можно ли ослабить до WARN.
        
        ```java
        @AfterThrowing(value = "storageOperationsWithTwoArgs()", throwing = "ex")
        ```
        

## Security

Были две попытки сделать аутентификацию - через псевдо-кастомный фильтр, он же декорированный `UsernamePasswordAuthenticationFilter` на минималках. Не требует отдельного контроллера.

```java
@Value("/api/v1/auth/login") String defaultFilterProcessesUrl
!request.getMethod().equals("POST")

private static final AntPathRequestMatcher AUTH_PATH_REQUEST_MATCHER =
        new AntPathRequestMatcher(AUTH_PATH_SEGMENT, HttpMethod.POST.name());
```

- “Главный плюс - глубокая интеграция в Spring Security, мало ручной работы. Магия происходит сама”:
    - Глубокая интеграция - это не интеграция, а использование Spring Security напрямую.
    - Мало ручной работы - можно еще меньше.
    
    Данный подход может иметь место быть, если существует четкое требование - обрабатывать запрос аутентификации с телом в формате JSON. Но не вижу проблем в том, чтобы отправлять данные аутентификации в виде x-www-form-urlencoded.
    Сам же REST стиль не лучшим образом сочетается с наличием сессий на стороне сервера, даже когда эти сессии хранятся в стороннем сервисе (redis). Тем не менее требование к наличию сессии есть, поэтому мы не можем полностью от них избавиться в пользу того же JWT. Но можем сделать шаг в эту сторону. Все, что для этого требуется - поменять дефолтные `SessionIdResolver` с `CookieHttpSessionIdResolver` на `HeaderHttpSessionIdResolver` и политику создания сессий на `IF_REQUIRED`. Для аутентификации можем использовать HttpBasic (имеем полное право). Что это дает - по сути получились те же яйца только в профиль, но:
    
    - Не надо делать собственный фильтр.
    - Не надо (но можно) делать собственный энд-поинт и контроллер соответственно. Любой запрос без заголовка Authorization и X-Auth-Token будет остановлен. Зарос с Authorization с верными учетными данными в ответ вернет также X-Auth-Token со значением в виде идентификатора сессии. Любой запрос с X-Auth-Token не требует Authorization заголовка и доступ к ресурсу будет обрабатываться уже на уровне сессии.
    - Не надо заниматься установкой контекста в сессию.
    - Не надо объявлять `SecurityContextRepository`.
    - Не надо (но можно) создавать пользовательский `AuthenticationSuccessHandler`.
    - Неудачную аутентификацию можно обработать с помощью `AuthenticationEntryPoint`, который в данном случае лучше подходит, чем `AuthenticationFailureHandler`.
- v2_controller_auth:
    - “Рест контроллеры с аутентификацией более легко читаются” - это заблуждение, потому что текущая логика максимально тривиальна, выполнена в “youtube” стиле и работает на стандартных процессах самого фреймворка. К тому же контроллеры “читаться” не должны в принципе. Максимум, что можно “прочитать” в контроллере - это то, кому он делегирует логику.
    - “Приходится вручную управлять процессом аутентификации” - за процесс аутентификации отвечает `AuthenticationProvider`. В данном случае используются стандартные средства фреймворка, поэтому никакого управления вручную не происходит. Возможно, речь шла про необходимость вручную добавлять контекст в сессию.
    - “Логика аутентификации слишком "размазывается" между фильтрами и контроллерами” - это так, поэтому аутентификация/авторизация осуществляется не на уровне сервлета, контроллера или интерцептора, а на уровне фильтров, делегируя процесс аутентификации `AuthenticationManager`, который подбирает `AuthenticationProvider`, который и отвечает за логику аутентификации.
    - “внедрение Spring Session с Redis усложняет процесс” - на самом деле в текущих обстоятельствах логика максимально примитивная - добавить в сессию атрибут с соответствующим названием. Но справедливости ради стоит отметить, что если сделать все с соблюдением SRP, то задача станет сложнее. Я не хочу сильно фокусировать внимание на SOLID, иначе это займет очень много времени, плюс всегда можно аргументировать тем, что это пет-проект (но я бы так лучше не делал :)).
        - Когда в `LoginController` мы вызываем `SecurityContextService::saveAuthToContext`, но по факт мы сохраняем контекст в атрибуты сессии. Путаница из-за названия происходит и от того, что у метода не одна ответственность. В данном виде он берет пустой контекст, действительно устанавливает туда `Authentication` и завершающее действие, как писал - добавление контекста в сессию.
        Я понимаю, откуда этот код, но там, где ты его взял - целью является показать концепцию, и чистый код там не приоритет.
        
        Про SOLID и другие моменты, связанные с чистотой кода, сделаю акцент еще в одном месте (storage), чтобы у тебя была возможность самостоятельно провести работу над ошибками. Переделывать не надо - просто подумать над тем, что нужно инкапсулировать в отдельный класс, что может заслуживать дополнительный уровень абстракции, что может ограничиваться просто выносом логики в отдельный вспомогательный метод.
        

## Сессия

Здесь смущают два момента. Первое в `SessionService` это:

```java
session.setAttribute(SPRING_SECURITY_CONTEXT_KEY, context);
RedisIndexedSessionRepository.RedisSession redSession = (RedisIndexedSessionRepository.RedisSession) session;

sessionRepository.save(redSession);
```

При наличии `SecurityContextService`. Главное здесь то, что ты обновляешь данные в сессии. И это как раз плавно выводит нас на второй момент  - обновление учетных данных.

В случае с паролем, ты закономерно аннулируешь сессию, но при изменении имени пользователя - просто контент сессии обновляешь. Имя пользователя - это часть учетных данных этого пользователя, и любые манипуляции с этим должны также повлечь за собой необходимость сделать логин. Хотя как правило имя пользователя не то, чтобы часто дают изменять. Плюс стоит запомнить, что учетные данные (имя пользователя, в каком бы видео оно не было представлено, и пароль) - это скорее отдельно стоящая абстракция, которую мы лишь ассоциируем с тем или иным пользователем. Именно поэтому в Spring мы можем наблюдать то, что фреймворк в контексте аутентификации/авторизации сам изначально пытается абстрагироваться от конечного пользователя за счет наличия контрактов UserDetails и UserDetailsService. Именно по этой причине в сессии по большому счету хранится только имя пользователя без всего того, что может быть присуще пользователю с точки зрения бизнес логики (например, avatarUrl или storagePlan).

## Storage

В этом разделе, чтобы не повторяться, я соберу большую часть замечаний, которые могут быть актуальны и для других частей программы.

- Первое и оно же главное, работа с `MinioClient` - это не репозиторий. В репозитории (или в дао) мы работаем с данными из бд (sql, nosql). Если к нам данные приходят с другого внешнего сервиса (”напрямую” через `Web/RestClient/Template` или через готовые клиенты от соответствующих API), то это слой сервисов и соответственно не стоит не смешивать это с контекстом репозитория.
- Увидел кэширование, что соответствует REST требованиям. Реализовано в виде декоратора, хотя у фреймворка есть готовое решение в декларативном стиле.
- `MinioOperations` - ты поделил предметную область на “папки” и “файлы” - хорошо. Отсюда возникла идея попробовать стратегию, но в Spring можно сделать лучше, потому что в текущем виде `MinioOperationResolver` нарушает OCP. Плюс, в качестве *альтернативы* - если “протянуть” разделение на “папки” и “файлы” вплоть до слоя контроллеров, то стратегия может и не понадобиться - основной момент здесь - необходимость каждый раз вызывать `resolve()` в `MinioRepository` не потребуется.
    - Метод `String::replaceFirst` принимает на вход регулярное выражение. Ты же в качестве этого аргумента передаешь строку.
        
        ```java
        return presignedObjectUrl.replaceFirst(endpoint, "")
                .replaceFirst("/" + bucket + "/", "");
        ```
        
        Понятно, что вряд ли endpoint или “/user-files/ могут соответствовать какому-либо регулярному выражению, но подобная привычка может сыграть злую шутку в будущем, поэтому обращай внимание на документацию используемого API.
        
    - Не мог пройти мимо настолько очевидного дублирования:
        
        ```java
        items.stream()
              .map(item -> StorageObjectResponse.builder()
                      .name(extractSimpleName(item.objectName()))
                      .path(extractRelativePath(item.objectName()))
                      .isFolder(item.isDir())
                      .lastModified(item.lastModified())
                      .size(item.size())
                      .build())
              .toList(); 
        ```
        
        Еще и с нарушением SRP вдобавок.
        
    - Обработка исключений try-catch внутри лямбды - некрасивый код:
        
        ```java
        protected List<Item> findItemsWithPrefix(String prefix, boolean recursive) {
            Iterable<Result<Item>> objects = minioClient.listObjects(
                    ListObjectsArgs.builder()
                            .bucket(bucket)
                            .recursive(recursive)
                            .prefix(prefix)
                            .build()
            );
        
            return StreamSupport.stream(objects.spliterator(), false)
                    .map(result -> {
                        try {
                            return result.get();
                        } catch (Exception e) {
                            throw new RuntimeException(e);
                        }
                    })
                    .toList();
        }
        ```
        
        Вынеси в отдельный вспомогательный метод эту логику и теперь можно будет даже использовать `@SneakyThrows` при желании:
        
        ```java
        public List<Item> findItemsWithPrefix(String prefix, boolean recursive) {
            Iterable<Result<Item>> objects = minioClient.listObjects(
                    ListObjectsArgs.builder()
                            .bucket(BUCKET)
                            .recursive(recursive)
                            .prefix(prefix)
                            .build()
            );
        
            return StreamSupport.stream(objects.spliterator(), false)
                    .map(this::getItem)
                    .toList();
        }
        
        @SneakyThrows
        private Item getItem(Result<Item> result) {
            return result.get();
        }
        ```
        
        Но, честно говоря, я не фанат `@SneakyThrows`. Как по мне - это тот случай, когда lombok способен создать больше проблем, чем удобств.
        
- `MinioFolderOperations`.
    - `objectStats()`:
        - `findFirst().get()` - **никогда** не вызывай `get()` без предварительных проверок.
    - `deleteObjectByPath()`:
        
        ```java
        .forEach(del -> {
        });
        ```
        
        Зря оставил лямбду пустой. В процессе могут возникнуть те же исключения и по хорошему их надо как-то обрабатывать. На худой конец, добавить логирование.
        
    - Метод `readObject()`:
    Интересный метод, который вобрал в себя все ошибки, которые можно найти по всему проекту.
        - Множество магических чисел/строк - все необходимо было вынести в константы.
        - Нарушение SRP - метод создает новый поток, берет i/o стрим из MinioOperations и архивирует данные.
        - `PipedInput/OutputStream` - использованы по назначению, но однозначно стоит узнать об альтернативах, потому что эти I/O классы имеют плохую производительность и сопряжены с различными ошибками (вплоть до deadlock). Если вкратце - использовать не рекомендуется.
        - скачивание папки реализовано в виде операции в новом потоке через `new Thread()`. Лучше используй `ExecutorService`/`CompletableFuture`/`@Async` (Spring).
        - Дополнительная работа с внутренним буфером (при передачи данных из `inputStream` от MinioClient в `zipOut`) выполнена вручную.
            
            ```java
            byte[] buffer = new byte[8192];
            int len;
            while ((len = inputStream.read(buffer)) > 0) {
                zipOut.write(buffer, 0, len);
            }
            ```
            
            При подобной ручной работе есть большая вероятность допустить ошибки. В таком случае можно рассмотреть в качестве альтернативы метод `transferTo()`, если используемые в `InputStream` дефолтный размер буфера (16 KB) устраивает: 
            
            ```java
            inputStream.transferTo(zipOut);
            ```
            
            В любом случае, всегда стоит обратить внимание на используемый API и реализованные в нем решения. Как минимум, если ты все же хочешь делать что-то похожее вручную, то возможно в документации будут описаны связанные с этим функционалом проблемы, о которых ты можешь не знать и соответственно никак их не учесть в своем решении.
            
- `MinioRepository`:
    - Предварительные проверки для разных действий `ensureObjectExists()` и `ensureObjectNotExists()` - стоит убедиться, что нижележащие методы используемого API (в данном случае это `MinioClient`) не позволяют выполнить соответствующую логику, уложившись в один запрос, или как минимум не позволяют обойтись без подобных предварительных запросов. По аналогии с СУБД, когда выполняется сперва запрос в БД для проверки на то, что запись уже существует, и только потом выполняется новый запрос на добавление или обновление записи, которая если имеет ограничение на уникальность, то достаточно обработать соответствующее исключение. Когда я пишу “стоит убедиться”, то я знаю, что для `MinioClient` это актуально.
    - Методы копирования и перемещения объектов являются хорошим примером того, как можно исправить дублирование (если допустим рефакторить твою стратегию и резолвер будет сложно):
        
        ```java
        @Override
        public void copy(String sourcePath, String targetPath) throws StorageObjectNotFoundException, StorageObjectAlreadyExistsException {
            MinioOperations operations = operationResolver.resolve(sourcePath);
        
            ensureObjectExists(sourcePath, operations);
            ensureObjectNotExists(targetPath, operations);
        
            operations.copyObject(sourcePath, targetPath);
        }
        
        @Override
        public void move(String sourcePath, String targetPath) throws StorageObjectNotFoundException, StorageObjectAlreadyExistsException {
            MinioOperations operations = operationResolver.resolve(sourcePath);
        
            ensureObjectExists(sourcePath, operations);
            ensureObjectNotExists(targetPath, operations);
        
            operations.copyObject(sourcePath, targetPath);
            operations.deleteObjectByPath(sourcePath);
        }
        
        private void ensureObjectExists(String path, MinioOperations operations) {
            if (!operations.objectExists(path)) {
                throw new StorageObjectNotFoundException("'%s' не существует в исходной папке"
                        .formatted(extractSimpleName(path)));
            }
        }
        
        private void ensureObjectNotExists(String path, MinioOperations operations) {
            if (operations.objectExists(path)) {
                throw new StorageObjectAlreadyExistsException("'%s' уже существует в целевой папке"
                        .formatted(extractSimpleName(path)));
        
            }
        }
        ```
        
        Нетрудно заметить, что методы `copy()` и `move()` практически полностью идентичны друг другу. Отличие заключается в том, что метод `move()` выполняется дополнительное действие удаления объекта в изначальной локации.
        Здесь подойдут функциональный интерфейсы из стандартной библиотеки. Точнее один - `Consumer`. Выделим общую часть во вспомогательный метод, который будет в качестве параметра принимать `Consumer<MinioOperations>`:
        
        ```java
        private void performOperation(String sourcePath, String targetPath, Consumer<MinioOperations> operation) 
                throws StorageObjectNotFoundException, StorageObjectAlreadyExistsException {
            MinioOperations operations = operationResolver.resolve(sourcePath);
            
            ensureObjectExists(sourcePath, operations);
            ensureObjectNotExists(targetPath, operations);
            
            operation.accept(operations);
        }
        ```
        
        Методы `copy()` и `move()` теперь выглядят так:
        
        ```java
        public void copy(String sourcePath, String targetPath)
                throws StorageObjectNotFoundException, StorageObjectAlreadyExistsException {
            performStorageOperation(sourcePath, targetPath,
                    ops -> ops.copyObject(sourcePath, targetPath));
        }
        
        public void move(String sourcePath, String targetPath)
                throws StorageObjectNotFoundException, StorageObjectAlreadyExistsException {
            Consumer<MinioOperations> copyOperation = ops -> ops.copyObject(sourcePath, targetPath);
            Consumer<MinioOperations> deleteOperation = ops -> ops.deleteObjectByPath(sourcePath);
            
            performOperation(sourcePath, targetPath, copyOperation.andThen(deleteOperation));
        }
        ```
        
        Подумай, можно ли применить подобный подход к остальным методам класса, и что можно улучшить в показанном мной примере в том числе.
        
- `UserStorageService`:
    - `user.getStoragePlan().getCapacity() * 1024 * 1024 * 1024` - вычисление повторяется. Можно (если не усложнять) вообще было вынести расчет емкости хранилища в соответствии с выбранным планом в сам `StoragePlan`.
    - `files.stream().map(MultipartFile::getSize).reduce(0L, Long::sum)` - в данном случае подобные вычисления можно (и нужно) заменять на более производительные аналоги:
        
        ```java
         files.stream()
            .mapToLong(MultipartFile::getSize)
            .sum();
        ```
        
        Операцию редукции `reduce()` рекомендуется использовать только тогда, когда другие варианты не подходят и когда в процессе редукции все промежуточные объекты неизменяемые.
        
        ```java
        static BigInteger factorial(int n) {
            return IntStream.rangeClosed(1, n)
                          .mapToObj(BigInteger::valueOf)
                          .reduce(BigInteger.ONE, BigInteger::multiply);
        }
        ```
        
        Объекты `BigInteger` неизменяемые. При умножении двух `BigInteger` получаем новый `BigInteger`, и никаких более простых способов сделать такую редукцию на `BigInteger` нет.
        
    - Почему методы `getAllSubPaths()` и `getFullPath()` статические? Intellij может по умолчанию в результате операции extract method создавать статические методы без какой-либо особой на то причины. Поэтому стоит внимательно смотреть, что получилось в результате и удалять модификатор `static` там, где он не требуется.
    - Метод `split("/")` из той же оперы, что и `replaceFirst()` - принимает на вход регулярное выражение. Если не хочется прибегать к использованию регулярок, то можно сделать так:
        
        ```java
        Pattern.compile("/", Pattern.LITERAL).split(path);
        ```
        
- `StorageController`.
/api/v1/resources - слишком абстрактно. Ресурсами может быть что угодно - пользователи, файлы, папки, хранилище и т.д. Когда нет домена, то альтернативой может быть: /api/v1/storage. Но сам по себе “storage” может ассоциироваться с хранилищем клиента (архетип ресурса, не связано с бизнес логикой). Поэтому, чтобы не было диссонанса можно было бы уточнить контекст, что это cloud-storage например. Но опять же - это слишком обобщенно и по сути тесно связано с основной предметной областью. Т.е это лучше подходит для суб-доменного имени: например, cloud-storage.restapi.com.
    
    Таким образом, мы снова приходим к тому, что контроллеры тоже можно было бы поделить соответствующим образом. И за не имением покупать доменное имя, то чтение энд-поинтов также было бы более прозрачным: /api/v1/folder(s) и /api/v1/file(s).
    

## User

1. Не увидел функционала создания корневого каталога для пользователя. Возможно уже просто глаз замылен, но при наличии уже `@TransactionalEventListener` в пакете session, я ожидал здесь чего-то подобного. Вообще, считаю, что создание корневой папки при регистрации пользователя - это один из интересных функционалов, который требует продумать следующие варианты развития событий:
    - Пользователь зарегистрировался, но в процессе создания папки случилась исключительная ситуация. Как поступить в таком случае? Какие-то повторные попытки?
    - Пользователь не смог зарегистрироваться, и в таком случае папка не должна быть создана. Решается за счет того же `@TransactionalEventListener`.

## Разное

- Опять вижу статический импорт с wildcard, хотя писал тебе о нем в ревью на погоду. Wildcard не используется ни со статическим, ни с обычным импортом. Чтобы идея не делала такие импорты за тебя - можно настроить этот момент в Editor → Code Style → Java в свойства Class count to use import with “*” и Names count to use static import with “*” установить большие значения.
- Наличие магических строк, для которых, если не хочется делать свои константы/классы с константами, как правило есть готовое решение у фреймворка.
- Смешение разных форматов ответов об ошибках - то `ProblemDetail`, то свой пользовательский формат (например, `StorageOperationResponse`).
- Смешение разных локализаций в проекте:
    
    ```java
    generateProblemDetail(UNAUTHORIZED,"Неправильное имя пользователя или пароль");
    generateProblemDetail(BAD_REQUEST, exception.getMessage());
    ```
    
    По умолчанию английский, и `MessageSource` + `ResouceBundle` для локализации сообщений.
    
- `@Mapper(componentModel = "spring")` - для “spring” есть константа.
