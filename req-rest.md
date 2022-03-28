## Feign Client

__github contributor history API활용__

    * Interface 선언을 통해 자동으로 Http Client 생성
    * 선언적 Http Client란, Annotation만으로 Http Client를 만들수 있고, 이를 통해서 원격의 Http API호출이 가능

+ Dependency 추가
```java
dependencies {
    ...
    
    /** feign client*/
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    implementation group: 'io.github.openfeign', name: 'feign-gson', version: '11.0'

    /** spring web*/
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'junit:junit:4.13.1'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
    annotationProcessor 'org.projectlombok:lombok'
    
    ...
}
```

+ Controller
```java
package com.example.feigntest.controller;

import ...

@Slf4j
@RestController
@RequiredArgsConstructor
public class GitHubFeignController {

    private final GitHubFeignService GitHubFeignService;

    @GetMapping(value = "/v1/github/{owner}/{repo}")
    public List<Contributor> getGitHubContributors(@PathVariable String owner , @PathVariable String repo){
        return GitHubFeignService.getContributor(owner,repo);
    }
}
```

+ Service
```java
package com.example.feigntest.controller;

import ...

@Slf4j
@RestController
@RequiredArgsConstructor
public class GitHubFeignController {

    private final GitHubFeignService GitHubFeignService;

    @GetMapping(value = "/v1/github/{owner}/{repo}")
    public List<Contributor> getGitHubContributors(@PathVariable String owner , @PathVariable String repo){
        return GitHubFeignService.getContributor(owner,repo);
    }
}
```

+ <span style='color:red'>FeignClient Interface</span>
```java
package com.example.feigntest.client;

import ...

@FeignClient(name="feign", url="https://api.github.com/repos",configuration = Config.class)
public interface GitHubFeignClient {
    @RequestMapping(method = RequestMethod.GET , value = "/{owner}/{repo}/contributors")
    List<Contributor> getContributor(@PathVariable("owner") String owner, @PathVariable("repo") String repo);
}

```


+ @EnableFeignClients Set
```java
package com.example;

import ...

@EnableFeignClients
@SpringBootApplication
public class ApiTestApplication {

    public static void main(String[] args) {
        SpringApplication.run(ApiTestApplication.class, args);
    }

}

```

+ Run 
<img src = '/images/Screen Shot 2022-03-28 at 0.54.37.png'>
