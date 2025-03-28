##  go mod init <your-module-name>
## go get github.com/rabbitmq/amqp091-go


### 만약 Exchange에서 주는 메시지를 각각의 애플리케이션이 독립적으로 받고싶다면?
  
각 애플리케이션이 큐를 생성하고 해당 익스체인지에 대해서 바인딩한다.


### 하나의 큐를 공통으로 쓰고 싶다면
바인딩된 큐를 여러 워크로드가 동시에 컨슘하면 된다.