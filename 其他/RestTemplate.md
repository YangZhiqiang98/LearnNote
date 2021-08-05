| 请求   | 方法            |
| ------ | --------------- |
| GET    | getForEntity    |
|        | getForObject    |
| POST   | psoForEntity    |
|        | postForEntity   |
|        | postForLocation |
| PUT    | put             |
| DELETE | delete          |

### GET请求
#### 第一种：getForEntity
该方法返回的是 `ReponseEntity` 对象，该对象是 Spring 对 Http 请求响应的封装，其中主要存储了 Http 的几个重要对象 。如请求状态的枚举对象 HttpStatus,在它的父类 HttpEntity 中还存储着 Http 请求的头信息对象 HttpHeaders 以及泛型类型的请求体对象。

该函数共有三个重载实现
- `ResponseEntity<T> getForEntity(String url, Class<T> responseType, Object... uriVariables)`：url 为请求地址，reponseType 为请求响应体 body 的包装类型，urlVariables 为 url 中的参数绑定。GET 请求的参数绑定通常使用 url 中拼接的方式，但更好的方法是在 url 中使用占位符并配合urlVariables 参数实现GET请求的参数绑定。
```java
ResponseEntity<String> yzq1 = restTemplate.getForEntity("http://provider/hello2?name={1}", String.class, "yzq1");
```
- `ResponseEntity<T> getForEntity(String url, Class<T> responseType, Map<String, ?> uriVariables)`：参数与上一方法一致。除了urlVariables 的参数类型发生了变化。
```java
Map<String,Object> map = new HashMap<>();
map.put("name", "张三");
restTemplate.getForEntity("http://provider/hello2?name={name}", String.class, map);
```
- `ResponseEntity<T> getForEntity(URI url, Class<T> responseType)`：该方法实用URI对象来代替之前的url和urlVariables参数来指定访问地址和参数绑定。

```java
String url = "http://provider/hello2?name=" + URLEncoder.encode("张三", "UTF-8");
URI uri = URI.create(url);
restTemplate.getForEntity(uri, String.class);
```
#### 第二种：getForObject
可以理解为对getForEntity的进一步封装，它通过HttpMessageConverterExtractor对HTTP的请求响应体body内容进行对象转换，实现请求直接返回包装好的对象内容。
```java
User user = restTemplate.getForObject(uri, User.class);
```
该函数也有三种重载实现。
- T getForObject(String url, Class<T> responseType, Map<String, ?> uriVariables)：url参数指定访问的地址，reponseType参数定义该方法的返回类型，urlVariables参数为url中占位符对应的参数。

- T getForObject(URI url, Class<T> responseType)：使用map类型的urlVariables替代上面数组形式的urlVariables。

- ResponseEntity<T> getForObject(String url, Class<T> responseType, Object... uriVariables)：使用URI对象来替代之前的url和urlVariables参数使用。 
### POST请求
#### 第一种：postForEntity
- ResponseEntity<T> postForEntity(String url, @Nullable Object request,  Class<T> responseType, Object... uriVariables)
- ResponseEntity<T> postForEntity(String url, @Nullable Object request,  Class<T> responseType, Map<String, ?> uriVariables) 
- ResponseEntity<T> postForEntity(URI url, @Nullable Object request, Class<T> responseType)
    参数用法大致和getForEntity一致，需要注意的是新增加的request参数，该参数可以是一个普通对象，也可以是一个HttpEntity对象，如果是一个普通对象，RestTemplate会将请求对象转化为一个HttpEntity对象来处理；而如果request是一个HttpEntity对象，那么就会被当做一个完整的Http请求对象来处理，这个request中不仅包含了body的内容，也包含了header的内容。

#### 第二种：postForObject
通过直接将请求相应的body内容包装成对象来返回使用，简化postForEntity的后续处理。
- T postForObject(String url, @Nullable Object request, Class<T> responseType,  Object... uriVariables) 
- T postForObject(String url, @Nullable Object request, Class<T> responseType,  Map<String, ?> uriVariables)
- T postForObject(URI url, @Nullable Object request, Class<T> responseType)

#### 第三种： postForLocation

该方式实现了以POST请求提交资源，并返回资源的URI。 
```java
MultiValueMap<String, Object> map = new LinkedMultiValueMap<>();
map.add("usernmae", "yzq");
map.add("password", "123");
map.add("id", 15);
URI uri = restTemplate.postForLocation("http://provider/register", map);
String s = restTemplate.getForObject(uri, String.class);
System.out.println(s);
```
- URI postForLocation(String url, @Nullable Object request, Object... uriVariables)
- URI postForLocation(String url, @Nullable Object request, Map<String, ?> uriVariables)
- URI postForLocation(URI url, @Nullable Object request)

由于postForLocation函数会返回新资源的URI，该URI就相当于指定了返回类型，所以此方法实现的POST请求不需要reponseType参数。
### PUT请求
-  void put(String url, @Nullable Object request, Object... uriVariables)
- void put(String url, @Nullable Object request, Map<String, ?> uriVariables)
- void put(URI url, @Nullable Object request)‘’

put函数为void类型，没有返回内容，也就没有reponseType参数。
### DELETE请求
- void delete(String url, Object... uriVariables)
- void delete(String url, Map<String, ?> uriVariables)
- void delete(URI url)

由于我们在进行REST 请求时，通常是将DELETE请求的唯一标识拼接在url中，所以DELETE请求不需要request的body信息。url指定DELETE请求的位置，urlVariables绑定url中的参数即可。 