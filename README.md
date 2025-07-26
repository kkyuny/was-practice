## HTTP Request 요청 처리
### 1. 서버 Socket 생성
  ``` Java
    try (ServerSocket serverSocket = new ServerSocket(port)) {
        logger.info("[CustomWebApplicationServer] started {} port.", port);

        ...
        Socket clientSocket;
        
        while ((clientSocket = serverSocket.accept()) != null) {
          ...
          executorService.execute(new ClientRequestHandler(clientSocket));
        }
    }
  ```
  - 소켓을 생성해서 8080 포토에서 연결을 기다릴 준비를 함
  - serverSocket.accept()에서 요청이 들어올 때까지 스레드는 블로킹된 상태로 있음.

### 2. Client의 Http 요청 발생
  ```
    GET /calculate?operand1=11&operator=*&operand2=55 HTTP/1.1
    Host: localhost:8080
    User-Agent: curl/7.64.1
    Accept: */*
  ```
  - 클라이언트가 TCP 연결 요청
    - TCP 3-way handshake 진행
  - 서버가 연결 확인(serverSocket.accept())
    - accpet()가 블로킹 상태에서 Socket을 리턴함.
  - Http 요청을 처리할 로직 수행
    ``` java
        executorService.execute(new ClientRequestHandler(clientSocket));
    ```

### 3. Http 요청 수신 및 파싱(로직 수행)
  - 서버가 클라이언트 소켓의 InputStream 확보
  ``` java
    InputStream in = clientSocket.getInputStream();
  ```
  - Http 요청 읽기
  ``` Java
    BufferedReader br = new BufferedReader(new InputStreamReader(in));
  
    public HttpRequest(BufferedReader br) throws IOException {
        this.requestLine = new RequestLine(br.readLine());
    }
  ```
  - 강의에서 헤더와 바디는 읽지 않았음.

### 4. 비즈니스 로직 처리
  - 요청 Path 확인 및 쿼리스트링 파싱 수행
  ```
  if (httpRequest.isGetRequest() && httpRequest.matchPath("/calculate")) {
    ...
  }
  ```
  
### 5. Http 응답 전송
  - OutputStream으로 응답을 전송
  ``` java
    OutputStream out = clientSocket.getOutputStream();
    DataOutputStream dos = new DataOutputStream(out);
    
    HttpResponse response = new HttpResponse(dos);
    response.response200Header("application/json", body.length);
    response.responseBody(body);
  ```

### 6. 다음 요청 대기
  - while문에서 accpet()가 다시 블로킹되어 다음 요청을 기다림.

### 7. InputStream과 OutputStream
  - Http는 텍스트 기반 프로토콜
  - 하지만 전송은 byte 단위로 이루어짐
  - 따라서 아래와 같이 요청/응답을 변환하여 사용함.
  ```java
    InputStream in = socket.getInputStream(); // 바이너리 스트림
    BufferedReader reader = new BufferedReader(new InputStreamReader(in)); // 텍스트화
    // reader를 이용하여 비즈니스 로직 수행

    OutputStream out = socket.getOutputStream(); // 바이너리 스트림
    DataOutputStream dos = new DataOutputStream(out); // 텍스트/바이너리 전송 가능
    // dos를 이용하여 Http 응답 전송
  ```