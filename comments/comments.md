# Самодокументирующийся код vs комментарии
Известно по лучшим практикам, что необходимо стремиться к самодокументирующемуся коду, с наглядными именами переменных, который хорошо структурирован и хорошо организован. Это всё автоматически делает комментарии как бы излишними. Идея в том, что другие разработчики поймут, что делает код, просто прочитав его.

Однако это несколько идеализировано, потому что скорее всего программист читающий код не обладает тем контекстом, которым обладал программист, написавший код. И очевидные вещи для второго программиста могут быть совершенно не очевидными для первого без контекста, хранящегося в уме автора кода.

Кроме того существуют типы информации, которые невозможно отразить ни в коде, ни в тестах, но при этом такая информация была бы очень полезна:
- что не надо делать (пред- и постусловия);
- советы на будущее (TODO);
- информация об оптимизации;
- информация глобального характера: как этот код вписывается в общую программу.

Поэтому в этой работе попробуем на нескольких примерах воспользоваться комментариями именно для передачи деталей дизайна и прочей информации, не отражающейся в самодокументируемом коде.

**Примеры:**  
В данном примере в комментарии для класса отражена информация глобального характера,отражающая дизайн системы.  
А в комментарии для метода отражен совет о необходимости оптимизации.
```php
/**
 * Class BlockChain.
 * Provides the longest chain of Block objects in the block tree.
 *
 * This class is a central element of the blockchain architecture and should be used for all operations
 * related to blocks and transactions.
 *
 * @package Ingniq\BlockchainTestTask
 */
class BlockChain
{
    // some code

    /**
     * Return a calculated account balance, using the longest existing chain of blocks in the tree.
     *
     * TODO: Consider optimizing the balance calculation method to speed up processing large volumes of transactions.
     *
     * @param string $accountName
     * @return int
     * @throws Exception
     */
    public function getBalance(string $accountName)
    {
        $length = strlen($accountName);
        if ($length < 2 || $length > 10) {
            throw new Exception('The length of the "account" property is not in a valid range.');
        }

        $accountName = strtolower($accountName);
        $account     = Account::getAccount($accountName);

        /** @var BlockTreeNode $node */
        foreach ($this->getBlockChain() as $node) {
            $transactions = $node->getBlock()->getTransactions();

            /** @var Transaction $transaction */
            foreach ($transactions as $transaction) {
                if ($transaction->getTo() === $accountName) {
                    $account['balance'] += $transaction->getAmount();
                }

                if ($transaction->getFrom() === $accountName) {
                    $account['balance'] -= $transaction->getAmount();
                }
            }
        }

        return $account['balance'];
    }

    // some code
}
```
В примере ниже в комментарии к классу указали важную информацию-предусловие, совет и предупреждение.
```csharp
    /// <summary>
    /// Client for interacting with the MapQuest API.
    /// Important: An API key is required for use, which must be specified in the configuration.
    /// TODO: Consider caching the results to reduce the number of API requests.
    /// Warning: Do not use this client for frequent sequential requests without delays to avoid API limitations.
    /// </summary>
    public class MapQuestClient
	{
        private readonly HttpClient _httpClient;
        private readonly string _mapQuestKey;

        public MapQuestClient(HttpClient httpClient, IConfiguration config)
        {
            _mapQuestKey = config.GetValue<string>("AppSettings:MapQuestApiKey");
            _httpClient = httpClient;
            _httpClient.BaseAddress = new Uri("https://www.mapquestapi.com/");

            if (_mapQuestKey == null)
            {
                throw new ArgumentNullException("AppSettings:MapQuestApiKey is null");
            }

        }

        public async Task<HttpResponseMessage> GetReverseGeocodeAsync(Point location)
        {
            return await _httpClient.GetAsync(
                $"geocoding/v1/reverse?key={_mapQuestKey}&location={location.Coordinate.X},{location.Coordinate.Y}" );
        }

        public async Task<HttpResponseMessage> GetRouteAsync(Point from, Point to)
        {
            return await _httpClient.GetAsync(
                $"directions/v2/route?key={_mapQuestKey}&fullShape=true&from={from.Coordinate.X},{from.Coordinate.Y}&to={to.Coordinate.X},{to.Coordinate.Y}" );
        }
    }
```
В примере ниже в комментарии к классу указали информацию о дизайне системе, важную информацию (в том числе что не надо делать) и советы.
```csharp
/// <summary>
/// A service for generating reports on vehicle mileage.
/// This class is key to the reporting module and interacts with the database context
/// to extract the necessary information. Calculations are performed using the GeographicLib library
/// for accurate calculation of distances between points.
///
/// Important:
/// - Do not use this service to calculate mileage at intervals of less than one day, so how
/// data aggregation occurs by day.
/// - When adding new calculation methods, keep in mind that the accuracy of calculations depends on the density and accuracy of geographical data.
///
/// TODO:
/// - Consider the possibility of optimizing the distance calculation to reduce the processing time of large amounts of data.
/// - Explore alternative methods of geographical calculations to improve accuracy.
/// </summary>
public class ReportService {
    public static ICollection<TimeValue> GenMileageReportByDays(VehicleContext _context, MileageReport report)
    {
        List<GeoPoint> geoPoints = (
                                    from gp in _context.GeoPoint
                                    where gp.VehicleId == report.Vehicle.Id
                                        && gp.CreatedAt.CompareTo(report.StartDate.UtcDateTime) >= 0
                                        && gp.CreatedAt.CompareTo(report.EndDate.UtcDateTime) <= 0
                                    select gp
                                ).ToList();

        Console.WriteLine($"geoPoints: {geoPoints.Count}, StartDate {report.StartDate}, EndDate {report.EndDate}");

        var groupedPoints = geoPoints
            .GroupBy(gp => gp.CreatedAt.Date)
            .Select(group => new { CreatedAtDate = group.Key, Points = group.ToList() })
            .ToList();

        Console.WriteLine($"groupedPoints: {groupedPoints.Count}");

        List<TimeValue> distanceByDay = new();

        foreach (var pointsInDay in groupedPoints)
        {
            Console.WriteLine($"Points.Count: {pointsInDay.Points.Count}");
            double distance = CalculateDistance(pointsInDay.Points);
            TimeValue timeValue = new ()
            {
                Time = pointsInDay.CreatedAtDate.ToShortDateString(),
                Value = Math.Round(distance, 2)
            };
            distanceByDay.Add(timeValue);
        }

        return distanceByDay;
    }

    public static double CalculateDistance(List<GeoPoint> pointsInDay)
    {
        double distanceInKm = 0;

        if (pointsInDay.Count == 0)
        {
            return distanceInKm;
        }

        Point startPoint = pointsInDay.First().Location;

        foreach (GeoPoint point in pointsInDay)
        {
            Distance(startPoint, point.Location, out double distanceInMeters);
            distanceInKm += distanceInMeters / 1000;
            startPoint = point.Location;
        }

        return distanceInKm;
    }
    private static void Distance(Point previousPoint, Point currentPoint, out double distanceInMeters)
    {
        Geodesic.WGS84.GenInverse(previousPoint.X, previousPoint.Y, currentPoint.X, currentPoint.Y, GeodesicFlags.Distance, out distanceInMeters, out _, out _, out _, out _, out _, out _);
    }
}
```
В примере ниже в комментарии к классу указали информацию о дизайне системе, важную информацию (в том числе что не надо делать) и советы.
```csharp
/// <summary>
/// The Data Generator Service class is designed to generate test data in a database.
///
/// Important:
/// - Do not use this service in production due to the risk of creating a large amount of data.
/// - Before starting, make sure that the database is configured and available.
///
/// TODO:
/// - Optimize performance by minimizing the number of database accesses.
/// - Consider adding configuration options to control the amount of data generated.
///
/// Note:
/// This class fits into the application architecture as a service for development and testing,
/// helping to quickly create the necessary amount of data to verify the functionality of the application.
/// </summary>
namespace Autopark.console.Services
{
    public class Data GeneratorService : IData GeneratorService
    {
        // some code
    }
}
```

**Резюме**

В ходе данной работы был рассмотрен вопрос использования комментариев в коде в контексте достижения баланса между самодокументируемым кодом и необходимостью комментирования для передачи контекста и деталей, которые не могут быть явно выражены через код. Были приведены примеры, демонстрирующие, как комментарии могут быть использованы для улучшения понимания кода, передачи намерений разработчика, предоставления предупреждений и советов по оптимизации.

Основные моменты:

1. **Комментарии важны:** Несмотря на стремление к написанию самодокументируемого кода, комментарии остаются важным инструментом для передачи контекста, который не может быть выражен через код. Они особенно ценны в сложных системах, где понимание общей картины и намерений разработчика критично для поддержки и развития проекта.

2. **Не злоупотреблять комментариями:** Комментарии должны быть краткими и по делу. Избыточное комментирование может сделать код труднее для восприятия и поддержки. Важно находить баланс между количеством комментариев и их полезностью.

3. **Комментарии для сложных моментов:** Особое внимание следует уделять комментированию сложных участков кода, алгоритмов и решений, где непосредственно код не способен полностью передать все нюансы реализации.

4. **Документирование предупреждений и TODO:** Комментарии могут быть использованы для указания на потенциальные проблемы, предупреждений о неочевидных последствиях использования кода, а также для отметки задач (TODO), которые предстоит выполнить.

В заключение, важно подчеркнуть необходимость поддерживать баланс и актуальность комментариев, а также стремиться к улучшению дизайна и архитектуры программного обеспечения. Эффективное использование комментариев в сочетании с хорошими практиками программирования может значительно улучшить качество кода, облегчить его поддержку и развитие, а также упростить взаимодействие в команде разработчиков.