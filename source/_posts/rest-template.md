---
title: rest-template
date: 2021-01-15 19:29:28
tags: java
---

### 构建restTemplate 
```
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate1() {
        return new RestTemplate();
    }

    //或者
    @Bean
    public RestTemplate restTemplate2(){
        return new RestTemplateBuilder().setConnectTimeout(Duration.ofSeconds(5))
                .setReadTimeout(Duration.ofSeconds(2)).build();
    }
}

```

### POST使用application/x-www-form-urlencoded方式
```
        MultiValueMap<String,Object> requestParam = new LinkedMultiValueMap<>();
        requestParam.add("grant_type","authorization_code");
        requestParam.add("code",code);
        requestParam.add("redirect_uri",redirectUri);
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        String authHeader = new BASE64Encoder().encode(String.format("%s:%s",clientId,clientSecret).getBytes());
        headers.add("Authorization",String.format("Basic %s",authHeader));
        HttpEntity<MultiValueMap<String,Object>> requestEntity = new HttpEntity<>(requestParam,headers);
        AccessToken accessToken=restTemplate.postForObject("http://localhost:8080/uaa/oauth/token",requestEntity,AccessToken.class);
```

### GET方法添加请求头
```
private UserInfo requestUserInfo(String token){
    HttpHeaders headers = new HttpHeaders();
    headers.add("Authorization",String.format("bearer %s",token));
    ResponseEntity<UserInfo> userInfoResponseEntity = restTemplate.exchange(
            "http://localhost:8080/uaa/oauth/user",
            HttpMethod.GET,
            new HttpEntity<>(headers),
            UserInfo.class);
    return userInfoResponseEntity.getBody();
}
```

### POST添加请求头
```
        RestTemplate restTemplate = new RestTemplate();
        HttpHeaders httpHeaders = new HttpHeaders();
        httpHeaders.setContentType(MediaType.APPLICATION_JSON);
        httpHeaders.add("AuthorizationToken","111");
        Map<String, Object> map = new HashMap<>();
        map.put("hello","world");
        HttpEntity requestEntity = new HttpEntity<>(map,httpHeaders);
        String result = restTemplate.postForObject("http://localhost:8081?timestamp=1610703163236",requestEntity,String.class);
```

### 使用https
```
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpclient</artifactId>
            <version>4.5.3</version>
        </dependency>
        

public RestTemplate genHttpRestTemplate() throws KeyStoreException, NoSuchAlgorithmException, KeyManagementException {
    TrustStrategy acceptingTrustStrategy = (X509Certificate[] chain, String authType) -> true;
    SSLContext sslContext = org.apache.http.ssl.SSLContexts.custom()
                .loadTrustMaterial(null, acceptingTrustStrategy)
                .build();
    SSLConnectionSocketFactory csf = new SSLConnectionSocketFactory(sslContext);
    CloseableHttpClient httpClient = HttpClients.custom().setSSLSocketFactory(csf)
                .build();
    HttpComponentsClientHttpRequestFactory requestFactory =
            new HttpComponentsClientHttpRequestFactory();
    requestFactory.setHttpClient(httpClient);
    RestTemplate restTemplate = new RestTemplate(requestFactory);
    return restTemplate;

    }
```

### 上传文件
```
public Attachment uploadAttachment(MultipartFile file){
    SimpleClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
    requestFactory.setBufferRequestBody(false);
    RestTemplate noBufferRestTemplate = new RestTemplate(requestFactory);
    
    MultiValueMap<String, Object> requestParam = new LinkedMultiValueMap<String, Object>();
    try {
            Resource resource = new MultipartStreamResource(file.getInputStream(),file.getSize(),file.getOriginalFilename());
            requestParam.add("file",resource);
    } catch (IOException e) {
        throw new MpaasRuntimeException(e);
    }
    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.MULTIPART_FORM_DATA);
    HttpEntity<MultiValueMap<String, Object>> requestEntity = new HttpEntity<MultiValueMap<String, Object>>(requestParam, headers);
    DataResponse response = noBufferRestTemplate.postForObject(String.format("%s/process/form/uploadAttachment","http://localhost:8081"), requestEntity, DataResponse.class);
}
```
其中的MultipartStreamResource 是继承自InputStreamResource
```
public class MultipartStreamResource extends InputStreamResource {
    private long length;
    private String fileName;

    public MultipartStreamResource(InputStream inputStream) {
        super(inputStream);
    }

    public MultipartStreamResource(InputStream inputStream, int length) {
        super(inputStream);
        this.length = length;
    }

    public MultipartStreamResource(InputStream inputStream, long length,String fileName) {
        super(inputStream);
        this.length = length;
        this.fileName = fileName;
    }


    public long getLength() {
        return length;
    }

    public void setLength(int length) {
        this.length = length;
    }

    @Override
    public String getFilename() {
        return this.fileName;
    }

    @Override
    public long contentLength() throws IOException {
        long estimate = length;
        return estimate == 0?1 :estimate;
    }
}
```
