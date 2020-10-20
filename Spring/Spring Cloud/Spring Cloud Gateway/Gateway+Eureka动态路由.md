# Gateway+Eureka动态路由

参考文章：

```
https://zhuanlan.zhihu.com/p/96658614
```

## 1.前言

虽然标题是Gateway+Eureka，不过这篇文章并不是严格意义上在gateway上将Eureka作为注册中心的，情况大概是这样，由于网关项目从前使用的是Zuul，服务信息会从Eureka拉取，但是架构升级之后，网关和注册中心分别改为了Gateway和Nacos，虽然主要的项目都做了迁移，但是仍然有一些老的项目注册在eureka上，导致这些旧项目的路由无法通过nacos来注册到gateway上，因此以yml的形式配置在了项目中，显而易见这对程序的扩展性和可用性造成了很大的影响，因此对这个项目进行优化。

**优化的目标**在于使gateway项目在已有的nacos作为注册中心的情况下，从eureka动态获取配置并动态更新路由。

**优化的难点**在于：

1.eureka和nacos无法同时成为注册中心，而当前项目需要以nacos作为注册中心；

2.gateway如何动态更新路由表信息；

3.gateway的yml配置对集群的负载均衡要求通过**lb://${service_name}**这一uri来做映射，而在自定义情况下配置的路由信息可能无法满足这一条件，考虑手动实现

4.旧服务路径携带版本信息，在解析uri时需要正则表达式通过正则表达式进行转换。



## 2.方案规划

针对上述的优化难点，规划以下解决方案：

1.通过eureka的http  API以获取配置，由于服务信息可能产生变化，因此需要创建定时任务动态获取服务配置信息；

2.参考网上教程和spring文档，可以通过SpringEvent进行事件推送，更新路由表；

3.分析请求流程后发现，网关工作原理为根据请求url查找路由表，获取具体路由信息后得到对应uri，并在相关过滤器中对uri进行系列操作，因此考虑创建过滤器对uri进行覆写，并同时完成客户端负载均衡；

4.正则表达式转换参考了Zuul的PatternServiceRouteMapper实现方案，难度较小，但需要注意细节

**简单的总结下，主要需要完成以下任务：**

定时任务+eureka api调用+动态路由更新+负载均衡算法+正则判断，下面放一下实现方法



## 3.方案实现

获取eureka服务配置信息，通过/eureka/apps接口获取所有服务，需要注意的是，该接口默认返回的是xml格式，对解析的难度困难稍大，因此需要在请求头添加Accept：application/json，使返回格式为json。先来看一下返回格式（忽略了一些没有用到的参数，但不代表不重要，具体的可以自己看下）：

```json
GET localhost:8011/eureka/apps

{
    "applications": {
        "application": [
            {
                //关键，服务名
                "name": "REGISTRY-V1",
                "instance": [
                    {
                        "instanceId": "localhost:8013",
                        "hostName": "localhost",
                        "app": "REGISTRY-V1",
                        //关键，ip地址
                        "ipAddr": "192.168.78.124",
                        "status": "UP",
                        "overriddenStatus": "UNKNOWN",
                        "port": {
                            //关键，端口
                            "$": 8013,
                            "@enabled": "true"
                        },
                        "countryId": 1,
                    },
                    {
                        "instanceId": "localhost:8011",
                        "hostName": "localhost",
                        "app": "REGISTRY-V1",
                        "ipAddr": "192.168.78.124",
                        "status": "UP",
                        "overriddenStatus": "UNKNOWN",
                        "port": {
                            "$": 8011,
                            "@enabled": "true"
                        }
                    }
                ]
            }
        ]
    }
}
```

简单了解下配置信息内容，这里我通过封装实体类将通过RestTemplate获取到的JSONObject转换成了java对象，贴一下实体类。

#### EurekaApps

```java
@Data
public class EurekaApps {
    private String versions__delta;

    private String apps__hashcode;

    private List<EurekaApp> application;
}
```

#### EurekaApp

```java
@Data
public class EurekaApp {
    private String name;

    private List<EurekaInstance> instance;
}
```

#### EurekaInstance

```java
@Data
public class EurekaInstance {
    private String instanceId;

    private String ipAddr;

    private String hostName;

    private JSONObject port;

    private String status;

    private Integer countryId;
}
```

余下的内容使用代码做统一展示与说明，主要分为定时任务、路由更新和过滤器的实现

#### 定时任务

```java
@Configuration
@EnableScheduling
public class TimingTask {
    private static final Logger logger = LoggerFactory.getLogger(TimingTask.class);
    private static final Pattern SERVICE_PATTERN = Pattern.compile("(?<name>.*)-(?<version>v.*$)");
    private static final String ROUTE_PATTERN = "${version}/${name}";
    @Value("${eurekaAddr}")
    private String eurekaAddr;
    @Autowired
    private DynamicRouteServiceImpl dynamicRouteServiceImpl;
    @Resource
    private RestTemplate restTemplate;

	//定时任务
    @Scheduled(initialDelay = 5000, fixedDelay = 30000)
    public void configureTasks() {
        //获取eureka服务列表
        if (StringUtils.isEmpty(eurekaAddr)){
            return;
        }
        String[] servers = eurekaAddr.split(",");
        String serverAddr = null;
        if (servers.length == 1){
            serverAddr = servers[0];
        }else {
            //随机指定server
            Random random = new Random();
            int count = random.nextInt(servers.length);
            serverAddr = servers[count];
        }
        if (StringUtils.isEmpty(serverAddr)){
            return;
        }
        HttpHeaders requestHeaders = new HttpHeaders();
        //接收配置类型为json
        requestHeaders.add("Accept", "application/json");
        HttpEntity<MultiValueMap> requestEntity = new HttpEntity<MultiValueMap>(requestHeaders);
        try {
            //发送请求获取配置信息
            ResponseEntity<JSONObject> result = restTemplate.exchange(serverAddr + "/eureka/apps", HttpMethod.GET, requestEntity, JSONObject.class);
            //解析配置到实体类
            JSONObject resultJSON = result.getBody().getJSONObject("applications");
            EurekaApps eurekaApps = resultJSON.toJavaObject(EurekaApps.class);
            List<RouteDefinition> definitionList = new ArrayList();
            //将所有eureka配置信息转换成routeDefinition
            eurekaApps.getApplication().stream().forEach(apps -> {
                List<EurekaInstance> eurekaAppList = apps.getInstance();
                //用来设置为路由id
                String instanceId = apps.getName();
                //解析路由id，若存在版本信息，则进行处理
                Matcher matcher = SERVICE_PATTERN.matcher(instanceId.toLowerCase());
                String route = matcher.replaceFirst(ROUTE_PATTERN);
                route = this.cleanRoute(route);
                instanceId = StringUtils.hasText(route) ? route : instanceId;

                //路由断言
                Map<String, String> args = new HashMap<>();
                args.put("_genkey_0", "/" + instanceId.toLowerCase() + "/**");
                PredicateDefinition predicateDefinition = new PredicateDefinition();
                predicateDefinition.setName("Path");
                predicateDefinition.setArgs(args);
                RouteDefinition routeDefinition = new RouteDefinition();
                List<String> uris = new ArrayList<>();
                //设置路由ip地址
                eurekaAppList.stream().forEach(app -> {
                    if (app.getStatus().equals("UP")) {
                        uris.add("http://" + app.getIpAddr() + ":" + app.getPort().get("$"));
                    }
                });
                //通过路由id判断过滤前缀长度
                FilterDefinition filterDefinition = new FilterDefinition();
                int extraLength = instanceId.length()-instanceId.replace("/","").length();
                filterDefinition.addArg("_genkey_0", instanceId.contains("/") ? 1+extraLength+"" : "1");
                filterDefinition.setName("StripPrefix");
                //路由信息赋值
                routeDefinition.setFilters(Collections.singletonList(filterDefinition));
                URI finalUri = URI.create(uris.stream().collect(Collectors.joining(",")));
                routeDefinition.setUri(uris.size() > 1 ? URI.create("lb://" + finalUri.toString()) : finalUri);
                routeDefinition.setPredicates(Collections.singletonList(predicateDefinition));
                routeDefinition.setId(instanceId);
                definitionList.add(routeDefinition);
                dynamicRouteServiceImpl.updateBatch(definitionList);

            });
        } catch (Exception e) {
            logger.error(e.getMessage());
        }
    }

    private String cleanRoute(final String route) {
        String routeToClean = route.replaceAll("/{2,}", "/");
        if (routeToClean.startsWith("/")) {
            routeToClean = routeToClean.substring(1);
        }
        if (routeToClean.endsWith("/")) {
            routeToClean = routeToClean.substring(0, routeToClean.length() - 1);
        }

        return routeToClean;
    }
}
```

#### 路由更新

```java
@Service
public class DynamicRouteServiceImpl implements ApplicationEventPublisherAware {
    private static final Logger logger = LoggerFactory.getLogger(DynamicRouteServiceImpl.class);

    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;
    private ApplicationEventPublisher publisher;

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
        this.publisher = applicationEventPublisher;
    }

    //更新路由
    public String update(RouteDefinition definition) {
        try {
            delete(definition.getId());
        } catch (Exception e) {
            logger.warn("路由id"+definition.getId()+"不存在");
        }
        try {
            routeDefinitionWriter.save(Mono.just(definition)).subscribe();
            this.publisher.publishEvent(new RefreshRoutesEvent(this));
            logger.info("更新路由"+definition.getId()+"成功");
            return "success";
        } catch (Exception e) {
            return "update route  fail";
        }
    }

    public String updateBatch(List<RouteDefinition> definitions) {
        for (int i = 0; i < definitions.size(); i++) {
            RouteDefinition definition = definitions.get(i);
            try {
                delete(definition.getId());
            } catch (Exception e) {
                logger.warn("路由id"+definition.getId()+"不存在");
            }
            try {
                routeDefinitionWriter.save(Mono.just(definition)).subscribe();
                logger.info("更新路由"+definition.getId()+"成功");
            } catch (Exception e) {
                logger.error(e.getMessage());
                return "update route fail";
            }
        }
        this.publisher.publishEvent(new RefreshRoutesEvent(this));
        return "success";
    }

    //删除路由
    public Mono<ResponseEntity<Object>> delete(String id) {
        return this.routeDefinitionWriter.delete(Mono.just(id))
                .then(Mono.defer(() -> Mono.just(ResponseEntity.ok().build())))
                .onErrorResume((t) -> t instanceof NotFoundException, (t) -> Mono.just(ResponseEntity.notFound().build()));
    }
}
```



#### 过滤器

```java
@Configuration
public class RouteFilter extends RouteToRequestUrlFilter {
    private static final String SCHEME_REGEX = "[a-zA-Z]([a-zA-Z]|\\d|\\+|\\.|-)*:.*";
    static final Pattern schemePattern = Pattern.compile(SCHEME_REGEX);

    private static Integer pos = 0;

    /* for testing */
    static boolean hasAnotherScheme(URI uri) {
        return schemePattern.matcher(uri.getSchemeSpecificPart()).matches()
                && uri.getHost() == null && uri.getRawPath() == null;
    }

    private static final Logger log = LoggerFactory.getLogger(RouteFilter.class);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        Route route = exchange.getAttribute(GATEWAY_ROUTE_ATTR);
        if (route == null) {
            return chain.filter(exchange);
        }
        log.trace("RouteToRequestUrlFilter start");
        URI routeUri = route.getUri();
        //处理自定义的服务器集群uri
        //lb://hello       lb://http://,htpp://
        if ("lb".equalsIgnoreCase(routeUri.getScheme())) {
            //正常的如lb://service-name直接跳过
            if (!routeUri.toString().contains(",")) {
                return chain.filter(exchange);
            }
            //解析uri
            String[] arr = routeUri.toString().split("lb://");
            if (arr.length<=1){
                throw new ArrayIndexOutOfBoundsException("URI解析失败");
            }
            //获取集群ip地址
            String[] ipAddrs = arr[1].split(",");
            Map<String, Integer> serverMap = new HashMap<String, Integer>();
            for (int i = 0; i < Arrays.asList(ipAddrs).size(); i++) {
                serverMap.put(Arrays.asList(ipAddrs).get(i), i);
            }
            Set<String> keySet = serverMap.keySet();
            ArrayList<String> keyList = new ArrayList<String>();
            keyList.addAll(keySet);
            //定义：访问的服务器地址
            String server = null;
            //设置：访问的服务器地址，采用轮询式负载均衡
            synchronized (pos) {
                if (pos > keySet.size() - 1) {
                    pos = 0;
                }
                server = keyList.get(pos);
                pos++;
            }

            try {
                //无法使用setter或继承重构Route对象，故采用反射创建新对象
                Class<Route> routeClass = Route.class;
                //设置构造器accessible并初始化
                Constructor<Route> constructor = routeClass.getDeclaredConstructor(
                        String.class,
                        URI.class,
                        int.class,
                        AsyncPredicate.class,
                        List.class);
                constructor.setAccessible(true);
                Route newRoute = constructor.newInstance(
                        route.getId(),
                        URI.create(server),
                        route.getOrder(),
                        route.getPredicate(),
                        route.getFilters());
                //覆盖route
                exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, newRoute);
            } catch (Exception ex) {
                log.error(ex.getMessage());
            }
        }
        return chain.filter(exchange);
    }
}
```

