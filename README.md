# Microsserviços com gRPC

Este guia oferece uma visão detalhada sobre a implementação e compreensão de um sistema de microsserviços de alto desempenho e segurança usando gRPC. Vamos explorar como o gRPC facilita a comunicação entre microsserviços e como aplicar autenticação e autorização de forma granular.

## O que é gRPC?

[gRPC](https://grpc.io/) é um framework de comunicação de alto desempenho desenvolvido pelo Google. Ele utiliza HTTP/2 e Protocol Buffers (protobuf) para oferecer uma comunicação eficiente entre microsserviços. Suporta múltiplas linguagens e é ideal para cenários de microsserviços devido à sua eficiência e suporte a chamadas síncronas e assíncronas.

## Fluxo de Dados

### 1. Requisição do Cliente

- **Cliente (Web ou Mobile)**: O cliente envia uma requisição HTTP REST para o serviço de pedidos. O token JWT é incluído no cabeçalho da requisição para autenticação.

   ```http
   POST /wsapi/orders HTTP/1.1
   Host: example.com
   Authorization: Bearer <token>
   Content-Type: application/json

   {
     "product_id": "123",
     "quantity": 2
   }
   ```

### 2. Processamento no Serviço de Pedidos

- **Controller do Serviço de Pedidos**: Recebe a requisição HTTP REST e valida o JWT.

   ```java
   @RestController
   public class OrderController {
      @PostMapping("/wsapi/orders") // wsapi -> web security api (segurança e validação do token abstraído pelo Spring Security)
      public ResponseEntity<?> createOrder(@RequestHeader("Authorization") String token, @RequestBody OrderRequest orderRequest) {
          // Validar parâmetros, e outros pontos desejados daí então chamar a camada de Service do microsserviço de pedido
          // para processar o pedido e chamar o microsserviço de pagamento
      }
   }
   ```

- **Camada de Serviço**: Após a validação, a camada de serviço pode chamar outros serviços, como o serviço de pagamento, usando gRPC. 

  **Nota:** A comunicação gRPC é direcionada diretamente para o método na camada de serviço do microsserviço, permitindo chamadas específicas conforme definido na interface do serviço gRPC.

### 3. Comunicação com o Serviço de Pagamento via gRPC

- **Chamada gRPC**: O serviço de pedidos utiliza gRPC para interagir com o serviço de pagamento. O JWT é passado nos metadados da chamada gRPC.

#### Exemplo com Java e gRPC

**Dependências Maven**

Adicione as seguintes dependências no `pom.xml`:

```xml
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty-shaded</artifactId>
    <version>1.48.1</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-protobuf</artifactId>
    <version>1.48.1</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-stub</artifactId>
    <version>1.48.1</version>
</dependency>
```

**Client gRPC**

```java
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.stub.MetadataUtils;
import io.grpc.Metadata;

public class PaymentServiceClient {
    private final PaymentServiceGrpc.PaymentServiceBlockingStub blockingStub;

    public PaymentServiceClient(ManagedChannel channel) {
        blockingStub = PaymentServiceGrpc.newBlockingStub(channel);
    }

    public PaymentResponse processPayment(String token, PaymentRequest request) {
        Metadata headers = new Metadata();
        Metadata.Key<String> authorizationKey = Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
        headers.put(authorizationKey, token);

        return blockingStub.withInterceptors(MetadataUtils.newAttachHeadersInterceptor(headers))
                           .processPayment(request);
    }
}
```

- **Interceptador no Servidor gRPC**: No lado do servidor, um interceptador pode validar o JWT e aplicar regras específicas de autenticação para cada método chamado.

**Nota:** É importante que o validateToken seja similar ou igual em todos os pontos de validação de token JWT para segurança.

```java
import io.grpc.*;

public class AuthInterceptor implements ServerInterceptor {
    private final String SECRET_KEY = "your_secret_key";

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
        return methodName.equals("/PaymentService/ProcessPayment");
    }

    private boolean validateToken(String token) {
        try {
            Jwts.parser()
                .setSigningKey(SECRET_KEY.getBytes())
                .parseClaimsJws(token);
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

### 4. Resposta ao Cliente

- **Processamento e Resposta**: Após o processamento no serviço de pagamento, o resultado é retornado ao serviço de pedidos, que por sua vez envia a resposta final de volta ao cliente.

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



