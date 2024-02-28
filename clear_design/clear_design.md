# Ясный дизайн
## 1. Введение
В данной работе попрактикуемся в мышлении на уровне дизайна приложения.

Можно выделить три уровня "думания" над программой:
- уровень времени выполнения (runtime). Как себя ведет программа в процессе выполнения -- какие состояния, какие процессы в конкретный момент времени;
- уровень исходного кода, конкретной реализации. Какие классы, методы, переменные, константы, типы данных, и т.д. есть в программе. Как это все взаимодействует на уровне понимания кода;
- уровень дизайна/проектирования/логики системы -- абстрактное описание системы в целом и в частностях, спецификация системы.

Возьмем несколько приложений для анализа.

## 2. Пример 1. Приложение "Утилиты для геометрических фигур"
### 2.1. Текущее состояние
На текущий момент реализован абстрактный класс `Shape` с методом `CalculateArea()`, от которого наследуются классы `Circle` и `Triangle`. В классе `Triangle` отдельно реализован метод `IsRightTriangle()`.

В спецификации описаны создание фигур и определение их площадей. Для треугольника описано определение прямоугольности.
Подразумевается что для фигур может быть множество различных утилит, не только вычисление площади и определение прямоугольности.  
Также возможно множество разных фигур, кроме указанных.  

### 2.2. Рекомендации по улучшению дизайна
Можно улучшить дизайн реализовав фабрику (фабричный метод) для создания фигур. Также можно типизировать виды измерений (угол, сторона, радиус), а также наборы параметров для фигур разного типа. Это позволит использовать ad-hoc полиморфизм.

### 2.3. Решение
Виды измерений:
```csharp
public readonly struct Radius
{
    public double Value { get; }

    public Radius(double value)
    {
        Value = value;
    }
}
public readonly struct Side
{
    public double Value { get; }

    public Side(double value)
    {
        Value = value;
    }
}
// etc.
```
Тип фигуры:
```csharp
public enum ShapeType
{
    Circle,
    Triangle
}
```
Набор параметров для различных фигур:
```csharp
public struct CircleParameters : IShapeParameters
{
    public readonly ShapeType Type => ShapeType.Circle;
    public Radius Radius;
}
public struct TriangleParameters : IShapeParameters
{
    public readonly ShapeType Type => ShapeType.Triangle;
    public Side? SideA;
    public Side? SideB;
    public Side? SideC;
    public Angle? AngleAB;
    public Angle? AngleAC;
    public Angle? AngleBC;
}
```
Реализуем фабрику:
```csharp
public class ShapeFactory
{
    private static Circle CreateShape(CircleParameters shapeParameters)
    {
        return new Circle(shapeParameters.Radius);
    }

    private static Triangle CreateShape(TriangleParameters shapeParameters)
    {
        if (
            shapeParameters.SideA != null &&
            shapeParameters.SideB != null &&
            shapeParameters.SideC != null
        )
        {
            return new Triangle((Side)shapeParameters.SideA, (Side)shapeParameters.SideB, (Side)shapeParameters.SideC);

        }

        throw new NotImplementedException("The creation of a triangle is implemented only on the basis of three sides.");
    }
}
```
И получаем следующий достаточно ясный дизайн в коде:
```csharp
CircleParameters circleParameters = new{
    Radius = new Radius(10)
};

TriangleParameters triangleParameters = new{
    SideA = new Side(3),
    SideB = new Side(4),
    SideC = new Side(5)
};

Shape circle = ShapeFactory.CreateShape(circleParameters);
Shape triangle = ShapeFactory.CreateShape(triangleParameters);

// Calculate area:
double circleArea = circle.CalculateArea();
double triangleArea = triangle.CalculateArea();

// Checking the squareness of a triangle:
bool isRightTriangle = triangle.IsRightTriangle();
```
## 3. Пример 2. Приложение "Model of blockchain currency"
### 3.1. Текущее состояние
Приложение представляет собой модель технологии блокчейн.
Позволяет выполнить транзакцию пополнения баланса пользователя, а также перевод валюты другому пользователю.

Существуют следующие классы:
- `Account` - эмуляция аккаунта пользователя. Содержит данные о двух пользователях - "bob" и "alice";
- `Transaction` – пополнение баланса или перевод валюты между пользователями;
- `Block` – список транзакций;
- `BlockTree` – дерево объектов типа `Block`;
- `BlockTreeNode` - узел дерева `BlockTree`;
- `BlockChain` – самая длинная цепочка объектов типа `Block` в `BlockTree`.

Пример проведения транзакции:
```php
// create 100 coins and transfer them to Bob
$trx = new Transaction();
$trx->setType(Transaction::TYPE_EMISSION);
$trx->setTo("bob");
$trx->setAmount(100);

$componentsSignature = array(
    $trx->getId(),
    $trx->getType(),
    $trx->getFrom(),
    $trx->getTo(),
    $trx->getAmount(),
);

$signature = MD5(implode(':', $componentsSignature));
$trx->setSignature($signature);

$block = new Block();
$block->addTransaction($trx);

$blockChain = new BlockChain();
$blockChain->addBlock(null, $block);

// Bob transfer 50 coins to Alice
$trx = new Transaction();
$trx->setType(Transaction::TYPE_TRANSFER);
$trx->setFrom("bob");
$trx->setTo("alice");
$trx->setAmount(50);

$componentsSignature = array(
    $trx->getId(),
    $trx->getType(),
    $trx->getFrom(),
    $trx->getTo(),
    $trx->getAmount(),
);

$signature = MD5(implode(':', $componentsSignature));
$trx->setSignature($signature);

$block = new Block();
$block->addTransaction($trx);

$blockChain->addBlock(1, $block);

echo 'Alice: ' . $blockChain->getBalance('alice') . "\n";
echo 'Bob: ' . $blockChain->getBalance('bob') . "\n";
```

### 3.2. Рекомендации по улучшению дизайна
На данный момент необходимо два типа транзакций - `EmissionTransaction` и `TransferTransaction`. Тип `Transaction` стоит сделать базовым для них. Необходимые параметры возможно лучше передать в конструктор.

Стоит создать специальный интерфейс `ITransactionSignature` и уже объекты данного интерфейса отправлять в метод `$trx->setSignature($signature)` если есть необходимость в разных типах подписи. В противном случае подпись можно инкапсулировать в классе `Transaction`. Здесь прослеживается неоднозначность дизайна.  
Также алгоритм `MD5` стоит заменить на более безопасный.  

Стоит также инкапсулировать логику создания блока в отдельный сервис.

### 3.3. Решение
**Разбираемся с типами транзакций.**  
```php
class EmissionTransaction extends Transaction
{
    // some properties and methods
}
class TransferTransaction extends Transaction
{
    // some properties and methods
}

// create 100 coins to Bob
$userTo = new Account("bob");
$amount = 100;
$trx = new EmissionTransaction($userTo, $amount);

// Bob transfer 50 coins to Alice
$userFrom = $userTo;
$userTo = new Account("alice");
$amount = 50;
$trx = new TransferTransaction($userFrom, $userTo, $amount);
```
**Разбираемся с подписью.**  
Инкапсулируем генерацию подписи в `Transaction`:
```php
class Transaction
{
    // some properties

    public function __construct($userFrom, $userTo, $amount, $type)
    {
        $this->type = $type;
        $this->from = $userFrom;
        $this->to = $userTo;
        $this->amount = $amount;
        $this->generateSignature();
    }

    private function generateSignature() {
        $components = [$this->id, $this->type, $this->from, $this->to, $this->amount];
        $this->signature = hash('sha256', implode(':', $components));
    }

    // some methods
}
```
**Инкапсулируем логику.**  
Инкапсулируем создание и добавление блока в цепочку блоков в отдельный сервис:
```php
$blockChain = new BlockChain();
$transactionService = new BlockChainTransactionService($blockChain);

// create 100 coins to Bob
$emissionTrx = new EmissionTransaction(new Account("bob"), 100);
$blockChain = $transactionService->runTransaction($emissionTrx);

// Bob transfer 50 coins to Alice
$transferTrx = new TransferTransaction(new Account("bob"), new Account("alice"), 50);
$blockChain = $transactionService->runTransaction($emissionTrx);

echo 'Alice: ' . $blockChain->getBalance('alice') . "\n";
echo 'Bob: ' . $blockChain->getBalance('bob') . "\n";
```

## 4. Пример 3. Приложение для йогических практик
### 4.1. Текущее состояние
Приложение находится в начальном этапе разработки. Представляет собой набор сущностей, в которых описаны основные поля и доменная логика.

Например в классе `User` есть методы:
- `public bool CheckPassword(string passwordToCheck)`;
- `private static string HashPassword(string password)`;
- `public Kriya? GetKriyaByName(string name)`;
- `public static void StartKriya(TimerSet timerSet, Statistic statistic)`;
- и т. п.

Пока не сформирована точка входа в приложение и другая логика, обеспечивающая запуск и работу приложения. Есть только набор сущностей и функционал, связанный с ними.

Поэтому есть хорошая возможность проверить существующий код и детализировать дизайн.

Специфицированы некоторые процессы:
- `UserCreateKriya` -- пользователь создает крию;
- `UserDeleteKriya` -- пользователь удаляет крию;
- `UserUpdateKriya` -- пользователь обновляет крию;
- `UserStartKriya` -- пользователь начинает крию;
- `UserGetKriya` -- пользователь получает крию;
- `UserGetKriyaList` -- пользователь получает список крий;
- `UserStopKriya` -- пользователь останавливает крию;
- `UserPauseKriya` -- пользователь приостанавливает крию;
- `UserResumeKriya` -- пользователь возобновляет крию;
- `KriyaFinished` -- крия завершается;
- `UserCreateTimerSet` -- пользователь создает набор таймеров;
- `UserDeleteTimerSet` -- пользователь удаляет набор таймеров;
- `UserUpdateTimerSet` -- пользователь обновляет набор таймеров;
- `UserGetStatistic` -- пользователь получает статистику.

### 4.2. Рекомендации по улучшению дизайна
По спецификации понятна часть дизайна. Но по коду не совсем понятно что делать, как использовать.  
В спецификации и в коде еще не хватает части логики. Не хватает точки входа в приложение.  
Не достаточно разграничены зоны ответственности. Доменная логика описана в сущностях.  

### 4.3. Решение
**Дописываем дизайн.**  
Приложению потребуются такие возможности, как:
- конфигурация БД
- авторизация и аутентификация
- роутинг
- кэширование
- логирование
- локализация

Необходимо сделать так, чтобы в файле-точке входа приложения легко подключались эти необходимые сервисы по типу "луковичной" архитектуры по мере их создания (или отключались при необходимости).

Главная страница приложения будет представлять собой дашбоард с настраиваемым отображением различных элементов.
Например, элемент `CurrentKriya` и элемент `Statistic`.

Данные статистики будут браться из кэша. При изменении статистики кэш должен обновляться.

**Выносим логику из сущностей в отдельные сервисы.**  
Создаем общие сервисы:
- `FileSystemService`
- `TimeProviderService`
- `HashService`

Создаем сервисы в контексте логики домена:
- `TimerService`
- `KriyaService`
- `UserService`
- `StatisticService`
- `CacheService`

**Дополняем логику приложения.**  
Создадим точку входа в приложение `Application` и разместим в нем код, который конфигурирует и инициализирует приложение со всеми необходимыми сервисами.
```csharp
class Application
{
    public Application()
    {
        var appBuilder = new ApplicationBuilder();

        appBuilder.Services.addDatabase(new DatabaseConfiguration("connectionConfig"));
        appBuilder.Services.addAuthentication(new AuthenticationService("authenticationConfig"));
        appBuilder.Services.addRouting(new RoutingService("routingConfig"));
        appBuilder.Services.addLocalization(new LocalizationService("localizationConfig"));
        appBuilder.Services.addCache(new CacheService("cacheConfig"));
        appBuilder.Services.addLogging(new LoggingService("logConfig"));

        var app = builder.Build();
        app.run();
    }
}
```
Реализуем логику подключения элементов на дашбоарде.
```csharp
    var kriyaService = new KriyaService();
    var statisticService = new StatisticService();
    var cacheService = new CacheService();
    var userService = new UserService();

    var dashboard = new Dashboard();
    dashboard.AddElement(new CurrentKriyaElement(kriyaService, userService.CurrentUser));
    dashboard.AddElement(new StatisticElement(statisticService, cacheService, userService.CurrentUser));

    dashboard.Display();
```

## 3. Выводы

В ходе данной работы мы рассмотрели несколько проектов и постарались сделать так, чтобы в коде был максимально виден дизайн, что делает его более читаемым, слабо связанным и невосприимчивым к ошибкам.

В достижении ясного дизайна в коде мы использовали разные возможности языка для приближения к более декларативному и короткому коду. Это инкапсуляция, наследование, полиморфизм, принципы модульности, паттерны проектирования и программирования и т. д. Используя эти возможности мы превращаем высокоуровневые концепции дизайна в код.

Также на одном из проектов мы посмотрели насколько удобно заранее спроектировать ясный дизайн, еще до написания большого количества кода. Последующий код по такому дизайну получается более структурирован, лаконичен и прост. И гораздо проще и быстрее продумать дизайн сначала, чем вникать и переписывать дизайн для существующего кода.

Также на практике становится более ясен такой момент, что думание на уровне выполнения кода и/или на уровне исходного кода в отрыве от ясного понимания дизайна чревато запутыванием, усложнением, неоднозначной и вообще некоректной реализацией функционала.