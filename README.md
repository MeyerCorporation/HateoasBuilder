# MeyerCorp.HateoasBuilder

A .NET Standard Library allowing convenient creation of HATEOAS for REST API models created by developers at [Meyer Corporation](https://meyerus.com).

## Background

HATEOAS or Hypermedia as the Engine of Application State is a helpful feature of REST and one many consider necessary to implement REST propertly. It can be often overlooked by backend developers as not critical but is very helpful to SPA development as API responses will contain predictable object schemas with URLs that can be used in pages of the SPA and linked pages ad infinitum.

In the following example, data is returned as an array of items. Each item has a description as well as an array of links. In this case, only one link is necessary to inform the consumer where the details for each item can be found. There is an array of links describing how to retrieve the previous page or data, the next page, and this page which is helpful for paginated data.

```JSON
{
    "data":[
        {
            "description":"first item",
            "links":[       
                {
                    "href":"https://foo.bar/api/item/1",
                    "rel":"self",
                    "type":"GET"
                }
            ]
        },
        {
            "description":"second item",
            "links":[       
                {
                    "href":"https://foo.bar/api/item/2",
                    "rel":"self",
                    "type":"GET"
                }
            ]
        },
        {
            "description":"third item",
            "links":[       
                {
                    "href":"https://foo.bar/api/item/3",
                    "rel":"self",
                    "type":"GET"
                }
            ]
        },
    ],
    "links":[
        {
            "href":"https://foo.bar/api/main?page=2",
            "rel":"self",
            "type":"GET"
        },
        {
            "href":"https://foo.bar/api/main?page=1",
            "rel":"previous",
            "type":"GET"
        },
        {
            "href":"https://foo.bar/api/main?page=3",
            "rel":"next",
            "type":"GET"
        }
    ]
}
```

I am not trying to convince anyone that they need to use HATEOAS, but if you do, and you are creating APIs in .NET, this can make it far more convenient.

## Getting Started

Add the [nuget package](https://www.nuget.org/packages/MeyerCorp.HateoasBuilder) to your .NET web application.

When returning data as a model in a method of a Web API controller, start by using one of the extension methods to create a link array.

```C#

// Some controller method
[HttpGet("all")]
public object GetAll()
{
   // The relative URL of the link target
    var relativeUrl1 = "employees?page=1"
    var relativeUrl2 = "employees?page=2"
    // Create a link array that has links for pagination to that relative URL
    var links = HttpContext
        .AddLink("previous", relativeUrl1)
        .AddLink("next", relativeUrl2)
        .Build();

    return new
    {
        Links = links,
        Results = Enumerable.Range(1, 6).Select(index => new WeatherForecast
        {
            Date = DateTime.Now.AddDays(index),
            TemperatureC = Random.Shared.Next(-20, 55),
            Summary = Summaries[Random.Shared.Next(Summaries.Length)],
           
            // Here we use a format string to create our link
            // http://base.url/WeatherForecast/1
            Links = HttpContext.AddFormattedLink("self", "WeatherForecast/{0}", index)
            .Build(),
        })
        .ToArray()
    };
}
```

By using the extension method for the `HttpContext`, the link builder is able to determine the base URL and append the employees route. This allow you to not worry what the base URL is as the `HttpContext` knows this and links can be created dynamically.

### Sample API

Included in this repository is a minimal .NET Web API application that references the library and demonstrates how to use some methods. Feel free to use this Postman collection to make calls to the API: [MmeyerCorporation/Hateoasbuilder](https://www.postman.com/meyerdevelopment/workspace/meyercorporation-hateoasbuilder).

## Methods

Various methods allow convenient creation of links. They are all methods of the LinkBuilder class as well as complimentary extension methods that allow initializing the LinkBuilder object starting with a base URL string or the `HttpContext` property of a Web API controller.

### AddLink

`AddLink` allows you to add a raw relative URL to the base URL which is either extracted from the `HttpContext` or a string.

```C#
"https://foo.bar".AddLink("label", "relativeLinkWithRoutesAndQueries");

// or
this.HttpContext
    .AddLink("previous", "data?page=1") //https://foobar/data?page=1
    .AddLink("self", "data?page=2") //https://foobar/data?page=2
    .AddLink("next", "data?page=3"); //https://foobar/data?page=3
```

### AddRouteLink

`AddRouteLink` allows you to add a relative URL to the base URL which is either extracted from the `HttpContext` or a string and add as many route items as you like appended to that.

#### Example

```C#
"https://foo.bar".AddRouteLink("label", "relativeUrl", "route", 1, "subroute", 2);

// or
this.HttpContext
    .AddRouteLink("employees", "id", 1, "dateOfHire") //https://foobar/employees/id/1/dateOfHire
    .AddRouteLink("locations", "id", 2, "address") //https://foobar/employees/id/2/address
    .AddRouteLink("products", "id", 3, "price"); //https://foobar/employees/id/3/price
```

### AddQueryLink

`AddRouteLink` allows you to add a relative URL to the base URL which is either extracted from the `HttpContext` or a string and add as many query parameters as you like appended to that.

#### AddQueryLink Example

```C#
"https://foo.bar".AddQueryLink("label", "relativeUrl", "route", 1, "subroute", 2);

// or
this.HttpContext
    .AddQueryLink("employees", "id", 1, "dateOfHire", "wednesday") //https://foobar/employees?id=1&dateOfHire=wednesday
    .AddQueryLink("locations", "id", 2, "address", "95687") //https://foobar/locations?id=2&address=95687
    .AddQueryLink("products", "id", 3, "price", "100") //https://foobar/products?id=3&price=100
    .AddQueryLink("products", "id", null, "price", "100"); //https://foobar/products?id=&price=100
```

### AddFormattedLink

`AddFormattedLink` allows you to add a relative URL to the base URL which is either extracted from the `HttpContext` or a string and format your URL as you like as if you were using `String.Format()`.

#### AddFormattedLink Example

```C#
this.HttpContext
    .AddFormattedLink("{0}/{1}/{2}?{3}={4}"employees", "id", 1, "dateOfHire", "wednesday") //https://foobar/employees/id/1?dateOfHire=wednesday
    .AddFormattedLink("locations", "id", 2, "address") //https://foobar/locations/id/2/address
    .AddFormattedLink("products", "id", 3, "price"); //https://foobar/products/id/3/price
```

### Build | BuildEncoded

`Build` or `BuildEncoded` is always the final call in the chain and returns the the links/name pairs as a collection which can be added to your returned data object. The `Link` objects will serialize to JSON automatically. XML is not officially supported at this time. `BuildEncoded` will encode your URLs which is a good practice but also crucial if your parameters contain special characters such as spaces, question marks, equal signs, etc.

#### Build Example

```C#
this.HttpContext
    .AddRouteLink("employees", "id", 1, "dateOfHire") //https://foobar/employees/id/1/dateOfHire
    .Build(); //[ https://foobar/employees/id/3/price ... ]
```

### AddParameters

`AddParameters` allows you to add any number of query parameters to the end of the last link that it is run from. The parameters are added as pairs of parameters in the .NET method with each pair representing a parameter name, then value `("name", "value", "name1", "value1")` which yields `?name=value&name1=value1`. The `AddParameters` method will only work after the `Add**Link` methods adding the parameters to that link it follows. Do not chain the `AddParameters` method.

#### AddParameters Example

```C#
this.HttpContext
    .AddRouteLink("employees", "id", 1, "dateOfHire") //https://foobar/employees/id/1/dateOfHire
    .AddParameters("location", "1", "address",2 ) //?location=1&address=2
    .Build(); //Yields: https://foobar/employees/id/1/dateOfHire?location=1&address=2
```

## Best Practices

* Always use the extension methods to create the link builder.
* Do not try to create a link builder as a member of a class. Only create an instance as a local variable in a method.
* Do not reuse an instance of the link builder.
* Most methods rely on a `params` attributed array parameter so you can just use an array instead of an explicit list of values.

### Which Method to Use

If your URL is simple and not dynamically generated, or you want to format your URL using inline syntax, then use `AddLink`:

```C#
var relativeUrl = "employees";
var route1 = "id";
var routeValue = "1234";
var query = "dateOfHire";
var queryValue = "20231225";
var query1 = "location";
var queryValue1 = "Vallejo";

var href = $"{relativeUrl}/{route1}/{routeValue}?{query}={queryValue}&{query1}={queryValue1}";

HttpContext.AddLink(href);

// https://foo.bar/employees/id/1234?dateOfHire=20231225&location=Vallejo
```

If your link relies on the route (w/o parameters) , then use `AddRouteLink`:

```C#
var relativeUrl = "employees";
var route1 = "id";
var routeValue1 = "1234";
var route2 = "dateOfHire";
var routeValue2 = "20231225";
var route3 = "location";
var routeValue3 = "Vallejo";

HttpContext.AddRouteLink(relativeUrl, route1, reouteValue1, route2, routeValue2, route3, routeValue3);

// https://foo.bar/employees/id/1234/dateOfHire/20231225/location/Vallejo
```

If your link relies on only parameters, then use `AddQueryLink`:

```C#
var relativeUrl = "employees";
var param1 = "id";
var paramValue1 = "1234";
var param2 = "dateOfHire";
var paramValue2 = "20231225";
var param3 = "location";
var paramValue3 = "Vallejo";

HttpContext.AddQueryLink(relativeUrl, param1, paramValue1, param2, paramValue2, param3, paramValue3);

// https://foo.bar/employees/?id=1234&dateOfHire=20231225&location=Vallejo
```

If your link relies on a route along with parameters, then use `AddQueryLink` and `AddParameters`:

```C#
var relativeUrl = "employees";
var route1 = "id";
var routeValue1 = "1234";
var param2 = "dateOfHire";
var paramValue2 = "20231225";
var param3 = "location";
var paramValue3 = "Vallejo";

HttpContext.AddRouteLink(relativeUrl, route1, routeValue1).AddParameters(param2, paramValue2, param3, paramValue3);

// https://foo.bar/employees/id/1234/?dateOfHire=20231225&location=Vallejo
```

If you prefer using a format string or your format is dynamically generated, then use `AddFormattedLink`:

```C#
var format = "{0}/{1}/{2}/?{3}={4}&{5}={6}";
var relativeUrl = "employees";
var route1 = "id";
var routeValue1 = "1234";
var param2 = "dateOfHire";
var paramValue2 = "20231225";
var param3 = "location";
var paramValue3 = "Vallejo";

HttpContext.AddFormattedLink(format, relativeUrl, route1, routeValue1, param2, paramValue2, param3, paramValue3);

// https://foo.bar/employees/id/1234/?dateOfHire=20231225&location=Vallejo
```

## Glossary

* REST: [**RE**presentational **S**tate **T**ransfer](https://restfulapi.net/)
* SPA: A **S**ingle **P**age **A**pplication is a SaaS application where the application goes back to the server for only data from an API. THe webpages are created by the code in the application typically by manipulation of the HTML DOM.

## References

* HATEOAS: [HATEOAS Driven REST APIs](https://restfulapi.net/hateoas/)
