# Microsserviços com gRPC

Este guia fornece uma visão detalhada sobre a implementação e compreensão de um sistema de microsserviços de alto desempenho e segurança usando gRPC. Vamos explorar como o gRPC facilita a comunicação entre microsserviços e como você pode aplicar autenticação e autorização de forma granular.

## O que é gRPC?

[gRPC](https://grpc.io/) é um framework de comunicação de alto desempenho desenvolvido pelo Google, baseado no protocolo HTTP/2 e no formato de serialização Protocol Buffers (protobuf). Ele permite a comunicação eficiente entre microsserviços e suporta múltiplas linguagens, incluindo Java, Python, Go, e mais. O gRPC é ideal para cenários de microsserviços devido à sua eficiência e suporte a chamadas síncronas e assíncronas.

Ele se comunica através 

## Fluxo de Dados

Abaixo está uma explicação detalhada do fluxo de dados em um sistema de microsserviços usando gRPC para comunicação entre serviços.

### 1. Requisição do Cliente

1. **Cliente (Web ou Mobile)**: O cliente faz uma requisição HTTP REST para o serviço de pedidos. O token JWT é incluído no cabeçalho da requisição para autenticação.

   ```http
   POST /api/orders HTTP/1.1
   Host: example.com
   Authorization: Bearer <token>
   Content-Type: application/json

   {
     "product_id": "123",
     "quantity": 2
   }
   ```

### 2. Processamento no Serviço de Pedidos

1. **Controller do Serviço de Pedidos**: O serviço de pedidos recebe a requisição HTTP REST e valida o JWT.

   ```java
   @RestController
   public class OrderController {
      @PostMapping("/wsapi/orders") // wsapi -> web security api (segurança e validação do token abstraído pelo Spring Security)
      public ResponseEntity<?> createOrder(@RequestHeader("Authorization") String token, @RequestBody OrderRequest orderRequest) {
          // Processar pedido e chamar o serviço de pagamento
      }
   }
   ```

2. **Camada de Serviço**: Após a validação, o controlador chama a camada de serviço que pode interagir com outros serviços, como o serviço de pagamento.

**Nota:** No gRPC, a comunicação é direcionada diretamente para o método na camada de serviço do microsserviço. Isso significa que, ao fazer uma chamada gRPC, você está invocando um método específico definido na interface do serviço gRPC.

### 3. Comunicação com o Serviço de Pagamento via gRPC

1. **Chamada gRPC**: O serviço de pedidos usa gRPC para se comunicar com o serviço de pagamento, passando o JWT nos metadados da chamada gRPC.

   ```java
   public class PaymentServiceClient {
      private final PaymentServiceGrpc.PaymentServiceBlockingStub blockingStub;

      public PaymentServiceClient(Channel channel) {
         blockingStub = PaymentServiceGrpc.newBlockingStub(channel);
      }

      public PaymentResponse processPayment(String token, PaymentRequest request) {
         Metadata headers = new Metadata();
         Metadata.Key<String> authorizationKey = Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
         headers.put(authorizationKey, token);

         return blockingStub.withCallCredentials(MetadataUtils.newAttachHeadersInterceptor(headers)).processPayment(request);
      }
   }
   ```

2. **Interceptador no Servidor gRPC**: No lado do servidor, você pode usar um interceptador para validar o JWT recebido e decidir se a autenticação é necessária para o método.

   ```java
   public class AuthInterceptor implements ServerInterceptor {
      @Override
      public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
         ServerCall<ReqT, RespT> call, Metadata headers, ServerCallHandler<ReqT, RespT> next) {
         String methodName = call.getMethodDescriptor().getFullMethodName();
         if (requiresAuthentication(methodName)) {
            String token = headers.get(Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER));
            if (token == null || !validateToken(token)) {
               call.close(Status.UNAUTHENTICATED.withDescription("Invalid token"), new Metadata());
               return new ServerCall.Listener<ReqT>() {};
            }
         }
  
         return next.startCall(call, headers);
      }

      private boolean requiresAuthentication(String methodName) {
         // Define quais métodos precisam de autenticação
         return methodName.equals("/PaymentService/ProcessPayment");
      }

      private boolean validateToken(String token) {
         // Implementar validação do JWT aqui
         return token != null && token.startsWith("Bearer ");
      }
   }
   ```

### 4. Resposta ao Cliente

1. **Processamento e Resposta**: Após a validação no serviço de pagamento, o resultado é retornado para o serviço de pedidos, que então envia a resposta final de volta ao cliente.

   ```java
   @Service
   public class OrderService {
      public void handleOrder(OrderRequest request) {
         PaymentRequest paymentRequest = createPaymentRequest(request);
         PaymentResponse paymentResponse = paymentServiceClient.processPayment("Bearer <token>", paymentRequest);
         // Processar resposta do pagamento e retornar ao cliente
      }
   }
   ```

## Resumo

- **Cliente**: Envia uma requisição HTTP REST com JWT.
- **Serviço de Pedidos**: Valida o JWT e faz chamadas gRPC ao serviço de pagamento.
- **Serviço de Pagamento**: Recebe a chamada gRPC, valida o JWT com base no método e processa a requisição.
- **Resposta Final**: O serviço de pagamento retorna a resposta ao serviço de pedidos, que então responde ao cliente.

Para consultar a documentação oficial clique [aqui](https://grpc.io/docs/).



