# Цикломатическая сложность

Задача: на нескольких примерах кода рассмотреть возможность и методы уменьшения цикломатической сложности (ЦС).

Цикломатическая сложность определяется как измерение "объема логики принятия решений в функции исходного кода". Чем больше решений, которые должны приниматься в коде, тем он сложнее.

Рассмотрим несколько реальных примеров кода и попробуем использовать различные методы снижения ЦС.

## Пример 1

Данный метод возвращает список элементов `BlogPostYearModel`.  
ЦС составляет 8.

```csharp
public virtual async Task<List<BlogPostYearModel>> PrepareBlogPostYearModelAsync()
{
    var store = await _storeContext.GetCurrentStoreAsync();
    var currentLanguage = await _workContext.GetWorkingLanguageAsync();
    var cacheKey = _staticCacheManager.PrepareKeyForDefaultCache(NopModelCacheDefaults.BlogMonthsModelKey, currentLanguage, store);
    var cachedModel = await _staticCacheManager.GetAsync(cacheKey, async () =>
    {
        var model = new List<BlogPostYearModel>();

        var blogPosts = await _blogService.GetAllBlogPostsAsync(store.Id,
            currentLanguage.Id);
        if (blogPosts.Any())  // ЦС == 1
        {
            var months = new SortedDictionary<DateTime, int>();

            var blogPost = blogPosts[blogPosts.Count - 1];
            var first = blogPost.StartDateUtc ?? blogPost.CreatedOnUtc;  // ЦС == 2
            while (DateTime.SpecifyKind(first, DateTimeKind.Utc) <= DateTime.UtcNow.AddMonths(1))  // ЦС == 3
            {
                var list = await _blogService.GetPostsByDateAsync(blogPosts, new DateTime(first.Year, first.Month, 1),
                    new DateTime(first.Year, first.Month, 1).AddMonths(1).AddSeconds(-1));
                if (list.Any())  // ЦС == 4
                {
                    var date = new DateTime(first.Year, first.Month, 1);
                    months.Add(date, list.Count);
                }

                first = first.AddMonths(1);
            }

            var current = 0;
            foreach (var kvp in months)  // ЦС == 5
            {
                var date = kvp.Key;
                var blogPostCount = kvp.Value;
                if (current == 0)  // ЦС == 6
                    current = date.Year;

                if (date.Year > current || !model.Any())  // ЦС == 8
                {
                    var yearModel = new BlogPostYearModel
                    {
                        Year = date.Year
                    };
                    model.Insert(0, yearModel);
                }

                model.First().Months.Insert(0, new BlogPostMonthModel
                {
                    Month = date.Month,
                    BlogPostCount = blogPostCount
                });

                current = date.Year;
            }
        }

        return model;
    });

    return cachedModel;
}
```

Для уменьшения ЦС можно вынести часть логики в отдельную функцию. Возьмем часть кода, которая отвечает за формирование отсортированного словаря дат постов блога за определенный промежуток и извлечем ее в отдельный метод:

```csharp
private async Task<SortedDictionary<DateTime, int>> GetMonthsBlogPostsAsync(IList<BlogPost> blogPosts, DateTime fromDate)
{
    var dates = new SortedDictionary<DateTime, int>();

    while (DateTime.SpecifyKind(fromDate, DateTimeKind.Utc) <= DateTime.UtcNow.AddMonths(1))
    {
        var list = await _blogService.GetPostsByDateAsync(blogPosts, new DateTime(fromDate.Year, fromDate.Month, 1),
            new DateTime(fromDate.Year, fromDate.Month, 1).AddMonths(1).AddSeconds(-1));
        if (list.Any())
        {
            var date = new DateTime(fromDate.Year, fromDate.Month, 1);
            dates.Add(date, list.Count);
        }

        fromDate = fromDate.AddMonths(1);
    }

    return dates;
}
```

Тогда основной метод станет таким:
```csharp
public virtual async Task<List<BlogPostYearModel>> PrepareBlogPostYearModelAsync()
{
    var store = await _storeContext.GetCurrentStoreAsync();
    var currentLanguage = await _workContext.GetWorkingLanguageAsync();
    var cacheKey = _staticCacheManager.PrepareKeyForDefaultCache(NopModelCacheDefaults.BlogMonthsModelKey, currentLanguage, store);
    var cachedModel = await _staticCacheManager.GetAsync(cacheKey, async () =>
    {
        var model = new List<BlogPostYearModel>();
        var blogPosts = await _blogService.GetAllBlogPostsAsync(store.Id, currentLanguage.Id);

        if (blogPosts.Any())  // ЦС == 1
        {
            var blogPost = blogPosts[blogPosts.Count - 1];
            var fromDate = blogPost.StartDateUtc ?? blogPost.CreatedOnUtc;  // ЦС == 2
            // Используем новый созданный метод
            var months = await GetMonthsBlogPostsAsync(blogPosts, fromDate);

            var current = 0;
            foreach (var kvp in months)  // ЦС == 3
            {
                var date = kvp.Key;
                var blogPostCount = kvp.Value;
                if (current == 0)  // ЦС == 4
                    current = date.Year;

                if (date.Year > current || !model.Any())  // ЦС == 6
                {
                    var yearModel = new BlogPostYearModel
                    {
                        Year = date.Year
                    };
                    model.Insert(0, yearModel);
                }

                model.First().Months.Insert(0, new BlogPostMonthModel
                {
                    Month = date.Month,
                    BlogPostCount = blogPostCount
                });

                current = date.Year;
            }
        }

        return model;
    });

    return cachedModel;
}
```

Выглядит немного более выразительно. ЦС основного метода уменьшилась на 2 до 6.  
Еще можно поработать над циклами и условиями:

```csharp
public virtual async Task<List<BlogPostYearModel>> PrepareBlogPostYearModelAsync()
{
    var store = await _storeContext.GetCurrentStoreAsync();
    var currentLanguage = await _workContext.GetWorkingLanguageAsync();
    var cacheKey = _staticCacheManager.PrepareKeyForDefaultCache(NopModelCacheDefaults.BlogMonthsModelKey, currentLanguage, store);
    var cachedModel = await _staticCacheManager.GetAsync(cacheKey, async () =>
    {
        var blogPosts = await _blogService.GetAllBlogPostsAsync(store.Id, currentLanguage.Id);

        // Так можно избавиться от вложенности цикла и условий.
        if (!blogPosts.Any())  // ЦС == 1
        {
            return model;
        }

        var firstBlogPost = blogPosts[blogPosts.Count - 1];
        var fromDate = firstBlogPost.StartDateUtc ?? firstBlogPost.CreatedOnUtc;  // ЦС == 2
        var months = await GetMonthsBlogPostsAsync(blogPosts, fromDate);

        // В список можно сразу добавить исходные данные. Тем самым убираем лишнее условие внутри цикла.
        var model = new List<BlogPostYearModel>();
        model.Insert(
            0,
            new BlogPostYearModel
            {
                Year = fromDate.Year
            }
        );
        model.First().Months.Insert(
            0,
            new BlogPostMonthModel
            {
                Month = fromDate.Month,
                BlogPostCount = blogPosts.Count
            }
        );
        // Переменную можно сразу инициализировать исходные данными. Тем самым убираем лишнее условие внутри цикла.
        var current = fromDate.Year;
        foreach (var kvp in months)  // ЦС == 3
        {
            var date = kvp.Key;

            // Здесь 'if' допустимо использовать для пропуска итерации.
            if (date.Year <= current)  // ЦС == 4
            {
                continue;
            }

            model.Insert(
                0,
                new BlogPostYearModel
                {
                    Year = date.Year
                }
            );

            var blogPostCount = kvp.Value;
            model.First().Months.Insert(0, new BlogPostMonthModel
            {
                Month = date.Month,
                BlogPostCount = blogPostCount
            });

            current = date.Year;
        }

        return model;
    });

    return cachedModel;
}
```
ЦС основного метода уменьшилась на 2. Однако выглядит еще не слишком читаемо и выразительно. Есть возможность извлечь повторяемую часть кода в отдельный метод.

Извлечем часть кода, которая отвечает за наполнение списка элементов типа `BlogPostYearModel`:

```csharp
private async Task<List<BlogPostYearModel>> FillBlogPostYearModelListAsync(List<BlogPostYearModel> model, DateTime date, int blogPostCount)
{
    model.Insert(
        0,
        new BlogPostYearModel
        {
            Year = date.Year
        }
    );
    model.First().Months.Insert(
        0,
        new BlogPostMonthModel
        {
            Month = date.Month,
            BlogPostCount = blogPostCount
        }
    );

    return model;
}
```
Тогда основной метод станет таким:
```csharp
public virtual async Task<List<BlogPostYearModel>> PrepareBlogPostYearModelAsync()
{
    var store = await _storeContext.GetCurrentStoreAsync();
    var currentLanguage = await _workContext.GetWorkingLanguageAsync();
    var cacheKey = _staticCacheManager.PrepareKeyForDefaultCache(NopModelCacheDefaults.BlogMonthsModelKey, currentLanguage, store);
    var cachedModel = await _staticCacheManager.GetAsync(cacheKey, async () =>
    {
        var blogPosts = await _blogService.GetAllBlogPostsAsync(store.Id, currentLanguage.Id);

        if (!blogPosts.Any())  // ЦС == 1
        {
            return blogPostYearModels;
        }

        var firstBlogPost = blogPosts[blogPosts.Count - 1];
        var fromDate = firstBlogPost.StartDateUtc ?? firstBlogPost.CreatedOnUtc;  // ЦС == 2
        var months = await GetMonthsBlogPostsAsync(blogPosts, fromDate);
        // Используем новый созданный метод
        var blogPostYearModels = await FillBlogPostYearModelListAsync(new List<BlogPostYearModel>(), fromDate, blogPosts.Count);

        var current = fromDate.Year;
        foreach (var kvp in months)  // ЦС == 3
        {
            var date = kvp.Key;

            if (date.Year <= current)  // ЦС == 4
            {
                continue;
            }

            blogPostYearModels = await FillBlogPostYearModelListAsync(blogPostYearModels, date, kvp.Value);
            current = date.Year;
        }

        return blogPostYearModels;
    });

    return cachedModel;
}
```
### Итого
Для рассматриваемого метода цикломатическая сложность понижена с 8 до 4.

Использованы следующие методы уменьшения ЦС:
- извлечение части логики в отдельные методы;
- избавление от вложенных циклов и условий.

Соблюдены принципы обеспечения низкой ЦС:
- между аргументами функции и телами условий соответствие один-к-одному;
- в цикле только одно условие и оно используется для пропуска итерации;
- отсутствуют `else` и цепочки `else if`;
- отсутствуют `if`, вложенные в `if`, и циклы, вложенные в `if`.

Остается неприятным момент с проверкой на `null` -- `blogPost.StartDateUtc ?? blogPost.CreatedOnUtc`.
Здесь необходимо решить вопрос на уровне типа данных `blogPost` так, чтобы этот тип имел однозначное определение полей `StartDateUtc` и `CreatedOnUtc`.

## Пример 2
В методе используется много условных конструкций, в которых генерируется html-строка пагинатора. Тело каждого условия отражает алгоритм построения отдельного элемента пагинатора с определенным набором данных.
ЦС составляет 26.

```csharp
public static async Task<IHtmlContent> PagerAsync<TModel>(this IHtmlHelper<TModel> html, PagerModel model)
{
    if (model.TotalRecords == 0)  // ЦС == 1
        return new HtmlString(string.Empty);

    var localizationService = EngineContext.Current.Resolve<ILocalizationService>();

    var links = new StringBuilder();
    if (model.ShowTotalSummary && (model.TotalPages > 0))  // ЦС == 3
    {
        links.Append("<li class=\"total-summary\">");
        links.Append(string.Format(await model.GetCurrentPageTextAsync(), model.PageIndex + 1, model.TotalPages, model.TotalRecords));
        links.Append("</li>");
    }

    if (model.ShowPagerItems && (model.TotalPages > 1))  // ЦС == 5
    {
        if (model.ShowFirst)  // ЦС == 6
        {
            //first page
            if ((model.PageIndex >= 3) && (model.TotalPages > model.IndividualPagesDisplayedCount))  // ЦС == 9
            {
                model.RouteValues.PageNumber = 1;

                links.Append("<li class=\"first-page\">");
                if (model.UseRouteLinks)  // ЦС == 10
                {
                    var link = html.RouteLink(await model.GetFirstButtonTextAsync(),
                        model.RouteActionName,
                        model.RouteValues,
                        new { title = await localizationService.GetResourceAsync("Pager.FirstPageTitle") });
                    links.Append(await link.RenderHtmlContentAsync());
                }
                else
                {
                    var link = html.ActionLink(await model.GetFirstButtonTextAsync(),
                        model.RouteActionName,
                        model.RouteValues,
                        new { title = await localizationService.GetResourceAsync("Pager.FirstPageTitle") });
                    links.Append(await link.RenderHtmlContentAsync());
                }
                links.Append("</li>");
            }
        }

        if (model.ShowPrevious)  // ЦС == 11
        {
            //previous page
            if (model.PageIndex > 0)  // ЦС == 12
            {
                model.RouteValues.PageNumber = model.PageIndex;

                links.Append("<li class=\"previous-page\">");
                if (model.UseRouteLinks)  // ЦС == 13
                {
                    var link = html.RouteLink(await model.GetPreviousButtonTextAsync(),
                        model.RouteActionName,
                        model.RouteValues,
                        new { title = await localizationService.GetResourceAsync("Pager.PreviousPageTitle") });
                    links.Append(await link.RenderHtmlContentAsync());
                }
                else
                {
                    var link = html.ActionLink(await model.GetPreviousButtonTextAsync(),
                        model.RouteActionName,
                        model.RouteValues,
                        new { title = await localizationService.GetResourceAsync("Pager.PreviousPageTitle") });
                    links.Append(await link.RenderHtmlContentAsync());
                }
                links.Append("</li>");
            }
        }

        if (model.ShowIndividualPages)  // ЦС == 14
        {
            //individual pages
            var firstIndividualPageIndex = model.GetFirstIndividualPageIndex();
            var lastIndividualPageIndex = model.GetLastIndividualPageIndex();
            for (var i = firstIndividualPageIndex; i <= lastIndividualPageIndex; i++)  // ЦС == 15
            {
                if (model.PageIndex == i)  // ЦС == 16
                    links.AppendFormat("<li class=\"current-page\"><span>{0}</span></li>", i + 1);
                else
                {
                    model.RouteValues.PageNumber = i + 1;

                    links.Append("<li class=\"individual-page\">");
                    if (model.UseRouteLinks)  // ЦС == 17
                    {
                        var link = html.RouteLink((i + 1).ToString(),
                            model.RouteActionName,
                            model.RouteValues,
                            new { title = string.Format(await localizationService.GetResourceAsync("Pager.PageLinkTitle"), i + 1) });
                        links.Append(await link.RenderHtmlContentAsync());
                    }
                    else
                    {
                        var link = html.ActionLink((i + 1).ToString(),
                            model.RouteActionName,
                            model.RouteValues,
                            new { title = string.Format(await localizationService.GetResourceAsync("Pager.PageLinkTitle"), i + 1) });
                        links.Append(await link.RenderHtmlContentAsync());
                    }
                    links.Append("</li>");
                }
            }
        }

        if (model.ShowNext)  // ЦС == 18
        {
            //next page
            if ((model.PageIndex + 1) < model.TotalPages)  // ЦС == 19
            {
                model.RouteValues.PageNumber = (model.PageIndex + 2);

                links.Append("<li class=\"next-page\">");
                if (model.UseRouteLinks)  // ЦС == 20
                {
                    var link = html.RouteLink(await model.GetNextButtonTextAsync(),
                        model.RouteActionName,
                        model.RouteValues,
                        new { title = await localizationService.GetResourceAsync("Pager.NextPageTitle") });
                    links.Append(await link.RenderHtmlContentAsync());
                }
                else
                {
                    var link = html.ActionLink(await model.GetNextButtonTextAsync(),
                        model.RouteActionName,
                        model.RouteValues,
                        new { title = await localizationService.GetResourceAsync("Pager.NextPageTitle") });
                    links.Append(await link.RenderHtmlContentAsync());
                }
                links.Append("</li>");
            }
        }

        if (model.ShowLast)  // ЦС == 21
        {
            //last page
            if (((model.PageIndex + 3) < model.TotalPages) && (model.TotalPages > model.IndividualPagesDisplayedCount))  // ЦС == 24
            {
                model.RouteValues.PageNumber = model.TotalPages;

                links.Append("<li class=\"last-page\">");
                if (model.UseRouteLinks)  // ЦС == 25
                {
                    var link = html.RouteLink(await model.GetLastButtonTextAsync(),
                        model.RouteActionName,
                        model.RouteValues,
                        new { title = await localizationService.GetResourceAsync("Pager.LastPageTitle") });
                    links.Append(await link.RenderHtmlContentAsync());
                }
                else
                {
                    var link = html.ActionLink(await model.GetLastButtonTextAsync(),
                        model.RouteActionName,
                        model.RouteValues,
                        new { title = await localizationService.GetResourceAsync("Pager.LastPageTitle") });
                    links.Append(await link.RenderHtmlContentAsync());
                }
                links.Append("</li>");
            }
        }
    }

    var result = links.ToString();
    if (!string.IsNullOrEmpty(result))  // ЦС == 26
        result = "<ul>" + result + "</ul>";

    return new HtmlString(result);
}
```

Можно реализовать ad-hoc полиморфизм для устранения необходимости использования условных конструкций и для понижения ЦС.

Создадим типы данных для каждого элемента пагинатора и реализуем функцию генерации html-строки элемента по этому типу данных. Для каждого типа данных будет своя функция, но называться она будет одинакого для всех.

Выделим следующие типы:
- PagerTotalSummaryItem
- PagerFirstItem
- PagerPreviousItem
- PagerIndividualItems
- PagerNextItem
- PagerLastItem

```csharp
public struct PagerTotalSummaryItem
{
    public int PageIndex { get; }
    public bool Show { get; }
    public Task<string> LinkText { get; }
    public int TotalPages { get; }
    public int TotalRecords { get; }

    public PagerTotalSummaryItem(
        int pageIndex,
        bool show,
        Task<string> linkText,
        int totalPages,
        int totalRecords
    ){
        PageIndex = pageIndex;
        Show = show;
        LinkText = linkText;
        TotalPages = totalPages;
        TotalRecords = totalRecords;
    }
}

public struct PagerFirstItem
{
    public int PageIndex { get; }
    public bool Show { get; }
    public IRouteValues RouteValues { get; }
    public bool UseRouteLinks { get; }
    public IHtmlHelper Html { get; }
    public Task<string> LinkText { get; }
    public Task<string> LinkTitle { get; }
    public string RouteActionName { get; }
    public int TotalPages { get; }
    public int IndividualPagesDisplayedCount { get; }

    public PagerFirstItem(
        int pageIndex,
        bool show,
        IRouteValues routeValues,
        bool useRouteLinks,
        IHtmlHelper html,
        Task<string> linkText,
        Task<string> linkTitle,
        string routeActionName,
        int totalPages,
        int individualPagesDisplayedCount
    ){
        PageIndex = pageIndex;
        Show = show;
        RouteValues = routeValues;
        UseRouteLinks = useRouteLinks;
        Html = html;
        LinkText = linkText;
        LinkTitle = linkTitle;
        RouteActionName = routeActionName;
        TotalPages = totalPages;
        IndividualPagesDisplayedCount = individualPagesDisplayedCount;
    }

public struct PagerIndividualItems
{
    public int PageIndex { get; }
    public bool Show { get; }
    public IRouteValues RouteValues { get; }
    public bool UseRouteLinks { get; }
    public IHtmlHelper Html { get; }
    public Task<string> LinkTitle { get; }
    public string RouteActionName { get; }
    public int FirstIndividualPageIndex { get; }
    public int LastIndividualPageIndex { get; }

    public PagerIndividualItems(
        int pageIndex,
        bool show,
        IRouteValues routeValues,
        bool useRouteLinks,
        IHtmlHelper html,
        Task<string> linkTitle,
        string routeActionName,
        int firstIndividualPageIndex,
        int lastIndividualPageIndex
    ){
        PageIndex = pageIndex;
        Show = show;
        RouteValues = routeValues;
        UseRouteLinks = useRouteLinks;
        Html = html;
        LinkTitle = linkTitle;
        RouteActionName = routeActionName;
        FirstIndividualPageIndex = firstIndividualPageIndex;
        LastIndividualPageIndex = lastIndividualPageIndex;
    }

    // И так далее...
}
```
В типе `PagerModel` добавим дополнительные поля для этих типов, так чтобы можно было обращаться `pagerModel.PagerTotalSummaryItem`. Создание объектов этих типов инкапсулируем в `PagerModel` либо в каком-либо сервисе, фабрике или другом подходящем функционале (оставим это за рамками рассматриваемой темы).

Реализуем функции построения html-элементов для этих типов:
```csharp
private static Task<StringBuilder> BuildPagerItemAsync(StringBuilder pagerString, PagerTotalSummaryItem pagerItem)
{
    // Попутно избавляемся от двойного условия `if (model.ShowTotalSummary && (model.TotalPages > 0)`
    if (!pagerItem.Show)
    {
        return pagerString;
    }

    if (pagerItem.TotalPages <= 0)
    {
        return pagerString;
    }

    pagerString.Append("<li class=\"total-summary\">");
    pagerString.Append(
        string.Format(
            pagerString.LinkText,
            pagerString.PageIndex + 1,
            pagerString.TotalPages,
            pagerString.TotalRecords
        ));
    pagerString.Append("</li>");

    return pagerString;
}

private static Task<StringBuilder> BuildPagerItemAsync(StringBuilder pagerString, PagerFirstItem pagerItem)
{
    // Попутно оптимизируем условия
    if (!pagerItem.Show)
    {
        return pagerString;
    }

    if (pagerItem.PageIndex < 3)
    {
        return pagerString;
    }

    if (pagerItem.TotalPages <= pagerItem.IndividualPagesDisplayedCount)
    {
        return pagerString;
    }

    pagerItem.RouteValues.PageNumber = 1;
    pagerString.Append("<li class=\"first-page\">");

    // Еще оптимизируем условия. Воспользуемся тернарным оператором и делегатом.
    Func<Task<string>, string, IRouteValues, object, IHtmlContent> buildLink = pagerItem.UseRouteLinks
        ? html.RouteLink
        : html.ActionLink;

    var link = buildLink(
        pagerItem.LinkText,
        pagerItem.RouteActionName,
        pagerItem.RouteValues,
        new { title = pagerItem.LinkTitle }
    );

    pagerString.Append(await link.RenderHtmlContentAsync());
    pagerString.Append("</li>");

    return pagerString;
}

private static Task<StringBuilder> BuildPagerItemAsync(StringBuilder pagerString, PagerIndividualItems pagerItem)
{
    if (!pagerItem.Show)
    {
        return pagerString;
    }

    Func<Task<string>, string, IRouteValues, object, IHtmlContent> buildLink = pagerItem.UseRouteLinks
        ? html.RouteLink
        : html.ActionLink;

    pagerItem.RouteValues.PageNumber = pagerItem.firstIndividualPageIndex + 1;

    // Понизим ЦС за счет применения рекурсивной функции вместо цикла.
    return BuildPagerIndividualItemsAsync(pagerString, pagerItem, buildLink);
}

private static Task<StringBuilder> BuildPagerIndividualItemsAsync(
    StringBuilder pagerString,
    PagerIndividualItems pagerItem,
    Func<Task<string>, string, IRouteValues, object, IHtmlContent> buildLink
){
    if (pagerItem.RouteValues.PageNumber > pagerItem.lastIndividualPageIndex + 1)
    {
        return pagerString;
    }

    if (pagerItem.RouteValues.PageNumber == pagerItem.PageIndex + 1)
    {
        pagerString.AppendFormat("<li class=\"current-page\"><span>{0}</span></li>", pagerItem.RouteValues.PageNumber);
        return BuildPagerIndividualItemsAsync(pagerString, pagerItem, buildLink);
    }

    pagerString.Append("<li class=\"individual-page\">");

    var link = buildLink(
        (pagerItem.RouteValues.PageNumber).ToString(),
        pagerItem.RouteActionName,
        pagerItem.RouteValues,
        new { title = pagerItem.LinkTitle }
    );

    pagerString.Append(await link.RenderHtmlContentAsync());
    pagerString.Append("</li>");

    pagerItem.RouteValues.PageNumber += 1;

    return BuildPagerIndividualItemsAsync(pagerString, pagerItem, buildLink);
}

// И так далее...
```

Также можно вынести в отдельный метод участок кода:
```csharp
private static HtmlString WrapHtmlListItems(StringBuilder pagerString)
{
    var result = pagerString.ToString();
    // Заменим стандартный условный оператор на тернарный.
    result = string.IsNullOrEmpty(result) ? result : "<ul>" + result + "</ul>";

    return new HtmlString(result);
}
```

И получаем оптимизированный основной метод:
```csharp
public static async Task<IHtmlContent> PagerAsync<TModel>(this IHtmlHelper<TModel> html, PagerModel model)
{
    if (model.TotalRecords == 0)  // ЦС == 1
        return new HtmlString(string.Empty);

    links = await BuildPagerItemAsync(new StringBuilder(), model.PagerTotalSummaryItem);

    if (!model.ShowPagerItems)  // ЦС == 2
    {
        return WrapHtmlListItems(links);
    }

    if (model.TotalPages <= 1)  // ЦС == 3
    {
        return WrapHtmlListItems(links);
    }

    links = await BuildPagerItemAsync(links, model.PagerFirstItem);
    links = await BuildPagerItemAsync(links, model.PagerPreviousItem);
    links = await BuildPagerItemAsync(links, model.PagerIndividualItems);
    links = await BuildPagerItemAsync(links, model.PagerNextItem);
    links = await BuildPagerItemAsync(links, model.PagerLastItem);

    return WrapHtmlListItems(links);
}
```

### Итого
Рассматриваемый код достаточно объемный и в нем получилось применить множество методов снижения ЦС, такие как:
- ad hoc полиморфизм;
- использование специально созданных типов данных для описания отдельных блоков состояний;
- использование делегатов;
- использование рекурсии;
- извлечение логики в отдельные методы;
- использование синтаксических сокращений условных операторов;
- оптимизация условных конструкций.

Соблюдены принципы обеспечения низкой ЦС:
- между аргументами функции и телами условий соответствие один-к-одному;
- отсутствуют `else` и цепочки `else if`;
- отсутствуют `if`, вложенные в `if`, и циклы, вложенные в `if`;
- от цикла вообще получилось избавиться за счет рекурсии.

ЦС основного метода был понижен с 26 до 3.

## Заключение
В данной работе мы рассмотрели два примера кода на C# и применили различные методы для снижения цикломатической сложности (ЦС). ЦС является важной метрикой, которая помогает оценить сложность программы или функции. Чем выше ЦС, тем сложнее структура кода, и тем сложнее его поддержка и тестирование.

В первом примере мы использовали методы извлечения логики в отдельные методы и избавления от вложенных циклов и условий. Это позволило снизить ЦС с 8 до 4.

Во втором примере мы применили более сложные методы, такие как ad-hoc полиморфизм, использование специально созданных типов данных, использование делегатов. Это позволило снизить ЦС с 26 до 3.

В обоих примерах мы соблюдали принципы обеспечения низкой ЦС: между аргументами функции и телами условий соответствие один-к-одному, отсутствие else и цепочек else if, отсутствие if, вложенных в if, и циклов, вложенных в if.

Особенно ярко получилось увидеть эффект увеличения выразительности кода во втором случае.

На основе проведенной работы можно сделать следующие выводы:
1. Извлечение логики в отдельные методы: Это один из самых простых и эффективных способов снижения ЦС. Он также улучшает читаемость и поддерживаемость кода.
2. Избегание вложенных циклов и условий: Вложенные циклы и условия увеличивают ЦС и делают код сложнее для понимания. Если возможно, стоит избегать их использования.
3. Использование ad-hoc полиморфизма и специально созданных типов данных: Эти методы могут быть сложнее для реализации, но они могут значительно улучшить структуру кода и снизить ЦС.
4. Использование делегатов и рекурсии: Эти методы могут быть полезны для снижения ЦС, но они также могут сделать код сложнее для понимания. Их следует использовать с осторожностью.
5. Использование синтаксических сокращений условных операторов и оптимизация условных конструкций: Эти методы могут помочь снизить ЦС и улучшить читаемость кода.