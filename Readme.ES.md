MvcThrottle
===========

[![Build status](https://ci.appveyor.com/api/projects/status/xdyd4xb4bihivdjt?svg=true)](https://ci.appveyor.com/project/stefanprodan/mvcthrottle)
[![NuGet](https://img.shields.io/nuget/v/MvcThrottle.svg)](https://www.nuget.org/packages/MvcThrottle)

Con MvcThrottle puede proteger su sitio de crawlers agresivos, herramientas de raspado o picos de tráfico no deseados originados desde la misma ubicación, limitando la tasa de peticiones que un cliente de la misma IP puede hacer a su sitio o a rutas específicas.

Puede establecer límites múltiples para diferentes escenarios, como permitir que una IP realice un número máximo de llamadas por segundo, por minuto, por hora o por día. Puede definir estos límites para direccionar todas las solicitudes realizadas a su sitio web o puede ampliar los límites a cada Controlador, Acción o URL, con o sin parámetros de cadena de consulta.

### Global throttling basado en IP

La siguiente configuración limitará el número de solicitudes originadas desde la misma IP. 

Si desde la misma IP, en el mismo segundo, harás una llamada a <code> home/index </code> y <code>home/about</code> la última llamada se bloqueará.

``` cs
public class FilterConfig
{
    public static void RegisterGlobalFilters(GlobalFilterCollection filters)
    {
        var throttleFilter = new ThrottlingFilter
        {
            Policy = new ThrottlePolicy(perSecond: 1, perMinute: 10, perHour: 60 * 10, perDay: 600 * 10)
            {
                IpThrottling = true
            },
            Repository = new CacheRepository()
        };

        filters.Add(throttleFilter);
    }
}
```

Para permitir el throttled tendrá que decorar su Controlador o Acción con <code>EnableThrottlingAttribute</code>, si desea excluir una determinada Acción puede aplicar <code>DisableThrottlingAttribute</code>.

``` cs
[EnableThrottling]
public class HomeController : Controller
{
    public ActionResult Index()
    {
        return View();
    }

    [DisableThrottling]
    public ActionResult About()
    {
        return View();
    }
}
```

Puede definir límites personalizados utilizando el atributo EnableThrottling, estos límites sustituirán a los predeterminados.

``` cs
[EnableThrottling(PerSecond = 2, PerMinute = 10, PerHour = 30, PerDay = 300)]
public ActionResult Index()
{
    return View();
}
```

### Endpoint throttling basados en IP

Si, desde la misma IP, en el mismo segundo, realiza dos llamadas a <code>home/index</code>, la última llamada se bloqueará.
Pero si en el mismo segundo usted llama <code>home/about</code> también, la petición pasará porque es una ruta diferente.

``` cs
var throttleFilter = new ThrottlingFilter
{
    Policy = new ThrottlePolicy(perSecond: 1, perMinute: 10, perHour: 60 * 10, perDay: 600 * 10)
    {
        IpThrottling = true,
        EndpointThrottling = true,
        EndpointType = EndpointThrottlingType.ControllerAndAction
    },
    Repository = new CacheRepository()
};
```

Usando la propiedad <code>ThrottlePolicy.EndpointType</code> puede elegir cómo se compone la tecla de aceleración.

``` cs
public enum EndpointThrottlingType
{
    AbsolutePath = 1,
    PathAndQuery,
    ControllerAndAction,
    Controller
}
```

### Personalización de la respuesta de límite de tasa

De forma predeterminada, cuando un cliente tiene una tasa limitada, se devuelve un código de estado HTTP 429 junto con un encabezado <code>Retry-After</code>. Si desea devolver una vista personalizada en lugar de la página de error IIS, deberá implementar su propio ThrottlingFilter y anular el método <code>QuotaExceededResult</code>.

``` cs
public class MvcThrottleCustomFilter : MvcThrottle.ThrottlingFilter
{
    protected override ActionResult QuotaExceededResult(RequestContext context, string message, HttpStatusCode responseCode)
    {
        var rateLimitedView = new ViewResult
        {
            ViewName = "RateLimited"
        };
        rateLimitedView.ViewData["Message"] = message;

        return rateLimitedView;
    }
}
```

He creado una vista llamada RateLimited.cshtml ubicada en la carpeta Views/Shared y usando ViewBag.Message estoy enviando el mensaje de error a esta vista. Eche un vistazo al proyecto MvcThrottle.demo para la implementación completa.

### IP, Endpoint y lista blanca de clientes

Si las solicitudes se inician desde una IP de lista blanca o a una URL de lista blanca, entonces la política de throttle no se aplicará y las solicitudes no se almacenarán. La lista blanca de IP soporta rangos IP v4 y v6 como "192.168.0.0.0/24", "fe80::/10" y "192.168.0.0.0-192.168.0.255" para más información consulte[jsakamoto/ipaddressrange](https://github.com/jsakamoto/ipaddressrange).

``` cs
var throttleFilter = new ThrottlingFilter
{
	Policy = new ThrottlePolicy(perSecond: 2, perMinute: 60)
	{
		IpThrottling = true,
		IpWhitelist = new List<string> { "::1", "192.168.0.0/24" },
		
		EndpointThrottling = true,
		EndpointType = EndpointThrottlingType.ControllerAndAction,
		EndpointWhitelist = new List<string> { "Home/Index" },
		
		ClientThrottling = true,
		//white list authenticated users
		ClientWhitelist = new List<string> { "auth" }
	},
	Repository = new CacheRepository()
});
```

El proyecto Demo viene con una lista blanca de IPs de Google y Bing bot, vea[FilterConfig.cs](https://github.com/stefanprodan/MvcThrottle/blob/master/MvcThrottle.Demo/App_Start/FilterConfig.cs).

### IP y/o Endpoint límites personalizados

Puede definir límites personalizados para IPs y endpoints conocidos, estos límites prevalecerán sobre los predeterminados. 
Tenga en cuenta que un límite personalizado sólo funcionará si ha definido una contraparte global.
Puede definir reglas de punto final proporcionando rutas relativas como <code>Home/Index</code> o simplemente un segmento de URL como <code>/About/</code>. 
El motor de aceleración del punto final buscará la expresión que ha proporcionado en la URI absoluta, 
si la expresión está contenida en la ruta de la petición, se aplicará la regla. 
Si dos o más reglas coinciden con la misma URI, se aplicará el límite inferior.

``` cs
var throttleFilter = new ThrottlingFilter
{
	Policy = new ThrottlePolicy(perSecond: 1, perMinute: 20, perHour: 200, perDay: 1500)
	{
		IpThrottling = true,
		IpRules = new Dictionary<string, RateLimits>
		{ 
			{ "192.168.1.1", new RateLimits { PerSecond = 2 } },
			{ "192.168.2.0/24", new RateLimits { PerMinute = 30, PerHour = 30*60, PerDay = 30*60*24 } }
		},
		
		EndpointThrottling = true,
		EndpointType = EndpointThrottlingType.ControllerAndAction,
		EndpointRules = new Dictionary<string, RateLimits>
		{ 
			{ "Home/Index", new RateLimits { PerMinute = 40, PerHour = 400 } },
			{ "Home/About", new RateLimits { PerDay = 2000 } }
		}
	},
	Repository = new CacheRepository()
});
```

### User-Agent limitación de tasa

Puede definir límites personalizados para los User-Agents conocidos o eventos de lista blanca, estos límites sustituirán a los predeterminados.  

``` cs
var throttleFilter = new ThrottlingFilter
{
	Policy = new ThrottlePolicy(perSecond: 5, perMinute: 20, perHour: 200, perDay: 1500)
	{
		IpThrottling = true,
		EndpointThrottling = true,
		EndpointType = EndpointThrottlingType.AbsolutePath,

		UserAgentThrottling = true,
		UserAgentWhitelist = new List<string>
		{
			"Googlebot",
			"Mediapartners-Google",
			"AdsBot-Google",
			"Bingbot",
			"YandexBot",
			"DuckDuckBot"
		},
		UserAgentRules = new Dictionary<string, RateLimits>
		{
			{"Slurp", new RateLimits { PerMinute = 1 }},
			{"Sogou", new RateLimits { PerHour = 1 } }
		}
	},
	Repository = new CacheRepository()
});
```

La configuración anterior permitirá al bot Sogou rastrear cada URL una vez cada hora, mientras que Google, Bing, Yandex y DuckDuck no tendrán ninguna tarifa limitada. 
Cualquier otro bot que no esté presente en la configuración tendrá una tarifa limitada basada en las reglas globales definidas en la constuctora ThrottlePolicy.

### Apilar las solicitudes rechazadas

Por defecto, las llamadas rechazadas no se añaden al contador del acelerador. Si un cliente hace 3 peticiones por segundo 
y ha establecido un límite de una llamada por segundo, los contadores de minutos, horas y días sólo grabarán la primera llamada, la que no fue bloqueada.
Si desea que las solicitudes rechazadas cuenten para los otros límites, deberá establecer <code>StackBlockedRequests</code> en true.

``` cs
var throttleFilter = new ThrottlingFilter
{
	Policy = new ThrottlePolicy(perSecond: 1, perMinute: 30)
	{
		IpThrottling = true,
		EndpointThrottling = true,
		StackBlockedRequests = true
	},
	Repository = new CacheRepository()
});
```

### Almacenamiento de métricas de throttle

MvcThrottle almacena todos los datos de solicitud en memoria utilizando ASP.NET Cache. Si desea modificar el almacenamiento a 
Velocity, MemCache o Redis, todo lo que tiene que hacer es crear su propio repositorio implementando la interfaz IThrottleRepository. 

``` cs
public interface IThrottleRepository
{
	bool Any(string id);
	
	ThrottleCounter? FirstOrDefault(string id);
	
	void Save(string id, ThrottleCounter throttleCounter, TimeSpan expirationTime);
	
	void Remove(string id);
	
	void Clear();
}
```

### Registro de las solicitudes throttled 

Si desea registrar las peticiones throttled, deberá implementar la interfaz IThrottleLogger y proporcionarla al ThrottlingFilter. 

``` cs
public interface IThrottleLogger
{
	void Log(ThrottleLogEntry entry);
}
```

Logging implementation example
``` cs
public class MvcThrottleLogger : IThrottleLogger
{
    public void Log(ThrottleLogEntry entry)
    {
        Debug.WriteLine("{0} Request {1} from {2} has been blocked, quota {3}/{4} exceeded by {5}",
            entry.LogDate, entry.RequestId, entry.ClientIp, entry.RateLimit, entry.RateLimitPeriod, entry.TotalRequests);
    }
}
```

Ejemplo de uso de registro 
``` cs
var throttleFilter = new ThrottlingFilter
{
	Policy = new ThrottlePolicy(perSecond: 1, perMinute: 30)
	{
		IpThrottling = true,
		EndpointThrottling = true
	},
	Repository = new CacheRepository(),
	Logger = new DebugThrottleLogger()
});
```
