# Проект “Планировщик задач”. 🕊 (@lynxio).

[Тг пост](https://t.me/zhukovsd_it_chat/53243/186874)

[Код проекта](https://github.com/lynxiox/planboard)

---

## Planboadr-api

### JwtAuthenticationFilter и JwtSevice

Классический Jwt фильтр “из youtube”.

1. Нарушает SRP, т.к содержит следующие ответственности:
    1. Получение заголовка и его значения:
        
        ```java
        String authHeader = request.getHeader("Authorization");
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }
        String jwt = authHeader.substring(7);
        ```
        
        Причем обработка значения заголовка упрощена до минимума - достаточно только то, что это есть префикс “Bearer “, а формат самого токена перед тем, как попасть уже в JwtService никак предварительно не проверяется. Соответственно, в `JwtService` эту ответственность возьмет на себя метод получения claims, что, как не сложно догадаться, также нарушает SRP. 
        
    2. Получить нужный claim:
        
        ```java
        String userEmail = jwtService.extractUsername(jwt, true);
        ```
        
    3. Работа с аутентификацией и контекстом:
        
        ```java
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        if (userEmail != null && authentication == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(userEmail);
        
            if (jwtService.isValid(jwt, userDetails, true)) {
        
                String role = jwtService.extractRole(jwt, true);
                List<SimpleGrantedAuthority> authorities = List.of(new SimpleGrantedAuthority("ROLE_" + role));
                UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,
                        authorities
                );
                token.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
        
                SecurityContextHolder.getContext().setAuthentication(token);
            } else {
                response.setStatus(HttpStatus.UNAUTHORIZED.value());
                response.getWriter().write("Invalid access token");
            }
        }
        ```
        
        При этом в этой части также собраны разные действия, размывающие границы ответственности метода фильтра.
        
2. Есть несколько вариантов улучшения логики работы с заголовком:
    
    ```java
    String authHeader = request.getHeader("Authorization");
    
    if (authHeader == null || !authHeader.startsWith("Bearer ")) {
        filterChain.doFilter(request, response);
        return;
    }
    String jwt = authHeader.substring(7);
    ```
    
    1. Избавиться от магических строк и чисел:
        - "Authorization" → HttpHeaders.AUTHORIZATION.
        - "Bearer " → константа BEARER_TOKEN_PREFIX.
        - 7 → BEARER_TOKEN_PREFIX.length().
        
        Код по сути тот же самый, но проще будет поддерживать, если вдруг что-то где-то поменяется.
        
    2. Проверить префикс можно с помощью `org.springframework.util.StringUtils`:
        
        ```java
        private static final BEARER_PREFIX = "bearer ";
        
        public void doFilterInternal(...) {
            String authHeader = request.getHeader(HttpHeaders.AUTHORIZATION);
        		
            if (!StringUtils.startsWithIgnoreCase(authHeader, BEARER_PREFIX) {
                filterChain.dofilter(request, response);
                return;
            }
        }
        ```
        
        Меньше ручной работы.
        
    3. Использовать регулярное выражение, которое сразу будет учитывать наличие префикса Bearer и соответствующий формат токена (для подписанного токена - это строка, разделенная на три части точками).
    В таком случае проверка заголовка сводится к примерно следующему виду:
        
        ```java
        private static final Pattern TOKEN_PATTERN = 
        				Pattern.compile("^Bearer (?<token>[a-zA-Z0-9-._~+/]+=*)$", Pattern.CASE_INSENSITIVE);
        
        private String authHeader = HttpHeaders.AUTHORIZATION;
        
        private String resolveToken(HttpServletRequest request) {
            String authorization = request.getHeader(this.authHeader);
        		
            if (!StringUtils.startsWithIgnoreCase(authorziation, "bearer") {
                // если значение заголовка не начинается с bearer, то fail-fast
            }
            Matcher matcher = TOKEN_PATTERN.matcher(authorziation);
            if (!matcher.matches()) {
            		// если значение не соответствует регулярному выражению - выдаем ошибку
            }
            return matcher.group("token");
        
        }
        ```
        
        Более гибкий вариант, гарантирующий соблюдение указанного формата.
        
3. Каждый запрос к api требует обращения к базе данных:
    
    ```java
    UserDetails userDetails = userDetailsService.loadUserByUsername(userEmail);
    ```
    
    Это можно представить как дополнительный уровень гарантии того, что в subject указано имя пользователя, который уже есть в базе данных (зарегистрирован). Но с учетом того, что данный функционал отдает токен после успешной аутентификации, то если токен не был перехвачен, ключи подписи не были скомпрометированы, то в subject априори будет имя пользователя, который есть в базе. Поэтому в данном случае обращение к бд являются лишними накладными расходами. И можно положиться на две вещи: а) проверка подписи (делает jsonwebtoken); б) проверка срока годности (тоже делает jsonwebtoken, когда обрабатывает его).
    
4. `@NonNull` - из `io.micrometer.common.lang`. Лучше взять аналог из Spring, т.к все же это основа проекта.
5. Методы работы с токеном в jsonwebtoken могут выбрасывать исключения, обработку которых я не увидел ни в фильтре, ни в `JwtService`. Из этого может следовать 
6. `jwtService.extractUsername(jwt, true)` - тип `boolean` не очень хорошо показывает себя качестве параметра метода, потому что не предоставляет никакого контекста при чтении кода. И необходимо погружаться в детали реализации метода, чтобы понять, что означает это `true`.
Вызов этого метода примечателен еще тем, что делает последующую проверку `JwtService::isValid` практически бессмысленной. Во-первых, в текущем виде получение имени пользователя сопряжено с парсингом токена. В этом случае, если у токена есть TTL, то по умолчанию будет осуществляться проверка на срок годности (метод `parse()` из `DefaultJwtParser`).
7. `jwtService.isValid(jwt, userDetails, true)`:
    1. Повторный вызов `extractUsername()` - токен уже был обработан. Это бессмысленное дублирование уже проделанной работы.
    2. `UserDetails` в качестве параметра выглядит неуместно в контексте обработки токена. Таким образом работа с токенами у тебя тесно переплетается со Spring, и не получится переиспользовать этот функционал без изменений исходников там, где Spring-а нет.
    3. `isTokenExpierd()` - как уже говорил, если у токена есть TTL, то по умолчанию парсер проверит, что срок годности не вышел. К тому, что опять стоит отметить, что в очередной раз будет осуществляться процесс обработки токена, потому что “под капотом” `isTokenExpired()` в конечном итоге полагается на `extractAllClaims()`.
    
    Из всего выше перечисленного особое внимание я бы уделил именно проблеме повторных обработок токена в рамках *одного* запроса. Обращение к бд можно списать на “я художник я так вижу”. А копипаст логики работы с токеном из youtube - не желанием усложнять пет-проект. И вот с последним я бы все же посоветовал не лениться и попытаться сделать как следует. Тем более, что с нуля писать не надо - нужно просто разделить работу с библиотекой jsonwebtoken и Spring Security по соответствующим уровням абстракции.
    

### Исключения

1. Для своих исключений по умолчанию должно быть три конструктора:
    - С параметром `String` - сообщение об ошибке.
    - С параметром `Throwable`, представляющим собой причину данного исключения.
    - C параметрами `String` и `Throwable`, позволяющими указать сообщение об ошибке и причину исключения.
    
    Второй и третий вариант нужны, когда ты низкоуровневые исключения оборачиваешь в какие-то свои например.
    
2. `GlobalExceptionHandler` - хочется здесь видеть более интересный объект ответа. Может быть свой пользовательский класс создать для ошибок или использовать `ProblemDetail` (RFC 9457).

### Сущности и ДТО

1. `public class User implements UserDetails` - не надо смешивать пользователя с точки зрения предметной области бизнес логики и логики работы с учетными данными в контексте безопасности.
2. `@OneToMany(…, fetch = FetchType.LAZY)` - для всех `…ToMany` ассоциаций `LAZY` - это `FetchType` по умолчанию и явно прописывать его не требуется.
3. `@ManyToOne(fetch = FetchType.EAGER)` - для всех `…ToOne` ассоциаций это `FetchType` по умолчанию и явно прописывать его не требуется.
4. Для ДТО использую рекорды.

### Контроллеры

1. В чем идея дублировать аннотации маппинга в контроллерах и в OpenAPI интерфейсах, которые они реализовывают?
2. Хоть REST и не запрещает использовать глаголы в энд-поинтах, если они указаны в конечном сегменте пути (RPC стиль), но в данном случае этого можно было не делать, т.к фреймворк сможет сопоставить требуемый функционал с HTTP методом запроса.
    
    ```java
    @PatchMapping("/{id}/update") // -> @PatchMapping("/{id}")
    @DeleteMapping("/{id}/delete") // -> @DeleteMapping("/{id}")
    ```
    

### Сервис

1. В `AuthService` метод `singup()`:
    
    ```java
    User user = userMapper.toUser(registrationUserDto);
    user.setPassword(passwordEncoder.encode(registrationUserDto.getPassword()));
    ```
    
    В данном контексте маппинг дто в сущность подразумевает то, что в сущности пароль будет представлен в зашифрованном виде. Если объявить метод с логикой шифрования в самом `UserMapper`, то это может стать нарушением SRP.
    
    - Понадобится создать маркерную аннотацию. Например, что-нибудь типа такого:
        
        ```java
        **@Qualifier // из мапстракта**
        @Target({ElementType.TYPE, ElementType.METHOD})
        @Retention(RetentionPolicy.CLASS)
        public @interface EncodedMapping {
        }
        ```
        
    - Создаем класс, отвечающий за кодировку пароля. Над методом кодировки ставим аннотацию, которую создали ранее:
        
        ```java
        @Component
        public class PasswordEncodingMapper {
        
            private static final PasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
        
            @EncodedMapping
            public static String encode(String value) {
                return passwordEncoder.encode(value);
            }
        }
        ```
        
        Здесь простой пример того, как может быть использоваться Bcrypt алгоритм шифрования, но это сопряжено с нарушением OCP. Подумай как это можно исправить.
        
    - В самом маппере добавляем необходимые атрибуты в аннотации `@Mapper` и `@Mapping`:
        
        ```java
        @Mapper(componentModel = MappingConstants.ComponentModel.SPRING,
        			**uses = PasswordEncodingMapper.class**)
        public interface UserMapper {
        
            @Mapping(target = "id", ignore = true)
            **@Mapping(source = "password", target = "password", qualifiedBy = EncodedMapping.class)**
            User mapToEntity(UserDto userDto);
        }
        ```
        
2. `user.setRole(Role.USER)` - если роль `USER` задается по умолчанию после регистрации, то это тоже можно сделать в `UserMapper`. Один из вариантов - с помощью `@AfterMapping` метода.
3. Роли никак не участвуют в бизнес-логике.
4. `public interface UserService extends UserDetailsService` - то же самое, как и в сущностях.
5. `TaskServiceImpl`.
    1. updateTask():
        - Ставший уже классикой лишний запрос в бд для проверки на соответствие уникальности:
            
            ```java
            if (StringUtils.hasText(requestUpdateTaskDto.getTitle()) &&
                    !requestUpdateTaskDto.getTitle().equals(task.getTitle()) &&
                    taskRepository.existsByTitleAndUser(requestUpdateTaskDto.getTitle(), user)) {
                throw new TaskAlreadyExistsException("Task with this title already exists for this user");
            }
            ```
            
            Если в ходе каких-то манипуляций с данными в бд будет нарушено ограничение на уникальность, то СУБД выбросит исключение. Поэтому достаточно обработать его (в Spring это DataIntegrityViolationException) и не делать запросов для предварительной проверки.
            
        - При наличии `@Transactional` сущность `Task` будет находится в состоянии managed до тех пор пока транзакция не завершится. В таком случае Hibernate выполнит синхронизацию состояния объектной модели с реляционной при коммите. Это произойдет быстро, потому что в методе отсутствует какая-либо логика, способная затянуть процесс. Таким образом вызов `TaskRepository::save` может быть лишним. Т.е это не ошибка, но просто на заметку.
    2. Динамический апдейт:
        
        ```java
            private void updateTaskFields(Task task, RequestUpdateTaskDto dto) {
                if (dto.getTitle() != null && !Objects.equals(task.getTitle(), dto.getTitle())) {
                    task.setTitle(dto.getTitle());
                }
                if (dto.getDescription() != null && !Objects.equals(task.getDescription(), dto.getDescription())) {
                    task.setDescription(dto.getDescription());
                }
                if (dto.getIscompleted() != null) {
                    task.setIscompleted(dto.getIscompleted());
                    task.setCompletedAt(dto.getIscompleted() ? LocalDateTime.now() : null);
                }
            }
        ```
        
        Если у сущности появятся новые поля, то придется изменять этот метод. Самый просто вариант при наличии MapStruct - это в соответствующем маппере создать метод для обновления сущности в процессе конвертации из дто:
        
        ```java
        @Mapper(componentModel = MappingConstants.ComponentModel.SPRING,
        				nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
        public interface TaskMapper {
        
        		void updateTaskFromDto(RequestUpdateTaskDto dto, @MappingTarget Task task);
        }
        ```
        
        В сервисе просто используем этот метод:
        
        ```java
        @Transactional
        public ResponseTaskDto updateTask(User user, Long id, RequestUpdateTaskDto requestUpdateTaskDto) {
        taskRepository.findByIdAndUser(id, user)
        				.map(task -> taskMapper.updateTaskFromDto(dto, task))
        				.map(updatedTask -> taskMapper.taskToResponseTaskCreateDto(updatedTask))
        				.orElseThrow(() -> new TaskNotFoundException());
        }
        ```
        
    3. `taskMapper.taskToResponseTaskCreateDto(task)` - название метода может ввести в заблуждение. Дто у тебя называется `ResponseTaskDto`, а значит и метод пусть лучше ассоциируется с этим: `toResponseTaskDto()`.

## planboard-scheduler

1. `MessageService`.
Единственный метод `generateDailyReport()` собирает данные по завершенным и незавершенным задачам. Подумай как сделать так, что при добавлении новой логики (например, сбор данных по не слишком просроченным задачам) не пришлось изменять данный метод.

## Messaging

Для асинхронного общения между сервисами по ТЗ выбрана Kafka. Spring предоставляет чуть более высокоуровневую абстракцию над непосредственно `KafkaProducer`, `KafkaConsumer` - `KafkaTemplate`. В использовании последнего нет ничего плохого, но это не предел. Так, например, Spring Cloud Stream и Spring Cloud Functions модули предоставляют более высокоуровневую работу, не привязанную к какому-то конкретному брокеру. Это не потребует каких-то более углубленных знаний Kafka (или другого выбранного провайдера) - достаточно будет базовых, потому что никаких особых настроек для брокера не требуется и хватит того, что Kafka дает из коробки.

При этом, если возникнет необходимость перейти с одного брокера на другой, то при наличии поддержки от фреймворка, все что потребуется - это поменять зависимости и настройки брокера в application.properties/yml. Никаких низкоуровневых вещей как:

```java
Map<String, Object> configProps = new HashMap<>();
configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapAddress);
configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
return new DefaultKafkaProducerFactory<>(configProps);
```

В коде же при отправки сообщений из конкретного места (т.е если при создании задачи отправляется в брокер сообщение) используется `StreamBridge`, а если необходимо в “потоковом” стиле обработать входящие данные и сформировать какой-нибудь, например, ответ - то Spring Functions позволяют это сделать с минимальным объемом работы. При этом можно использовать функциональные интерфейсы из стандартной библиотеки. Это позволит в том числе отказаться уже от устаревающего `@KafkaListener` в сервисе-потребителе.

Единственное, что в Spring Functions мне лично не нравится - это то, что эти функции необходимо объявлять как Bean Definition-ы через `@Bean`.

### Разное

- Не используй wildcard (”*”) ни с обычными, ни со статическими импортами.
- Все, что связано с исключительными ситуациями, надо логировать.
- docker:
    - почитай про многослойные Jar файлы и как это можно использовать в Dockerfile. Можно использовать jib плагин для создания образов.
    - есть сервисы, с которыми внешний мир никак не взаимодействует. Например, с базой данных могу общаться только имеющиеся микросервисы, поэтому пробрасывать порт контейнера на порт хоста не надо.
    - когда один сервис зависит от другого, то надо помимо depends_on указать также и условие (сервис запущен, сервис “здоров” и т.д).

## Микросервисная архитектура

Поскольку этот проект представляет собой лишь знакомство с миром микросервисов, и поэтому многих вещей он не требует, но я считаю важным все же донести информацию, которую следует знать в том числе и перед тем, как сесть за реализацию седьмого задания.

1. Первое, что хочется отметить - это то, что ты полностью разделила проекты. Никаких упрощений в виде многомодульных проектов и т.д. Это правильно.
2. Основная причина для перехода на микросервисы - это необходимость быстро и качественно горизонтально масштабироваться, чего с монолитами добиться крайне сложно. Это означает, что каждый твой микросервис может быть представлен в общей системе в виде нескольких работающих в одно и то же время экземпляров. И это накладывает дополнительные сложности.
    1. Так, например, у тебя отсутствует централизованный источник для конфигурации. Сейчас ты этого не замечаешь, но достаточно попробовать запустить N экземпляров любого из твоих сервисов, и если потребуется что-то изменить (например, изменится имя пользователя базы данных), то придется это делать для каждого имеющегося экземпляра сервиса. Централизованное хранилище решает эту проблему. И если мы остаемся в рамках возможностей Spring, то Spring Cloud Configuration Server предоставляет возможность использовать разные варианты хранения конфигураций, которые будут подхватывать все имеющиеся экземпляры микросервисов: можно хранить в файлах на диске, можно хранить в classpath, можно хранить в git и т.д.
    Другой момент - это хранение секретов/чувствительных данных. Для этого можно присмотреться к Vault, с которым у Spring также есть интеграция (у Spring Config Server в том числе, причем можно указать сразу и git и vault в качестве источников для соответствующих значений). Чтобы сервисы автоматически могли подтягивать изменения в конфигурации - есть Spring Cloud Bus.
    2. Следующий момент - это Service Discovery. Горизонтальное масштабирование подразумевает то, что система в целом должна знать, какие экземпляры каких сервисов доступны здесь и сейчас, и как до них достучаться (какие порты и т.д). Для этого в Spring можно взять Spring Cloud Netfilx (Eureka).
    3. Далее, это балансировщик нагрузки, который в координации с сервисом обнаружения служб (Eureka) способен распределять запросы по разным экземплярам сервиса. В седьмом задании, вроде бы нет микросервисов, между которыми бы было синхронное общение через например `RestClient`, и поэтому данный аспект делегируется той же Kafka.
    4. Паттерны, связанные с отказоустойчивостью. Для начала стоит ознакомиться с Circuit Breaker и Fallback pattern. Первый позволяет системе адаптироваться в случае деградации каких-либо экземпляров сервисов: повышение времени отклика, превышение таймаута в принципе, количество попыток достучаться до какого-либо экземпляра и т.д. Второй паттерн позволяет в случае неудачного запроса использовать альтернативный источник данных (в том числе отдать просто какой-нибудь дефолтный ответ).
    5. Gateway. Является неотъемлемой частью микросервисной архитектуры, позволяя централизовать сквозную логику, которая должна распространяться на разные микросервисы. Самый простой пример - это security. Так вместо того, чтобы в каждом микросервисе дублировать одну и ту же логику - это делается в одном месте, поскольку изначально все запросы будут проходить через gateway, где к ним и будут применяться все требуемые правила безопасности.
    6. Observability. Крайне важный аспект микросервиса, который сочетает в себе сбор метрик, агрегацию логов и трассировку. В совокупности это позволяет отслеживать состояние системы в целом, просматривать логи того или иного микросервиса в одном месте и отслеживать ход выполнения запросов через все сервисы. Для этого можно использовать: Actuator, Prometheus, Grafana (включая Loki и Alloy - современный аналог Promtail) и OpenTelemetry (устаревающие аналоги Spring Cloud Sleuth + Zipkin).
    Сперва можно было бы поэкспериментировать с трассировкой запросов вручную. Для этого потребуется понять, как можно отслеживать запрос, который будет путешествовать между разными микросервисами. Т.е нужно определиться с тем, как идентифицировать запрос в отношении всей системы целиком и в отношении каждого отдельного микросервиса, через который он проходит.

Я намеренно опущу здесь тему работы с данными (CQRS, Saga и т.д) и OAuth2, поскольку все, что упомянул ранее не потребует много времени для реализации, включая время на изучение вопроса. Но касательно работы с данным, то это добавит слишком много сложностей и крайне бы затянуло реализацию; касательно OAuth2/OIDC, то работа с токенами вручную является одним из пунктов ТЗ, поэтому готовыми инструментами типа Keycloak пользоваться не будем.
Стоит также сделать небольшую ремарку в отношении Spring Cloud стека. Поскольку многое из перечисленного выше делегируется или облачным сервисам или например Docker swarm, K8S, то даже минимальное погружение в Spring Cloud может показать лишней тратой времени, тем не менее, повторюсь, я постарался собрать те моменты, которые стоит попробовать, и на что не уйдет много времени.
