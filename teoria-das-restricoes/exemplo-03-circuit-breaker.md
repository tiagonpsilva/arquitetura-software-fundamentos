# Exemplo 3: Circuit Breaker Pattern e TOC

O Circuit Breaker (Disjuntor) é um padrão de design que exemplifica perfeitamente a aplicação da Teoria das Restrições em arquiteturas de software resilientes. Este padrão permite que sistemas distribuídos lidem elegantemente com falhas em componentes, evitando que restrições temporárias ou permanentes em um serviço comprometam todo o ecossistema.

## Conceito do Circuit Breaker

```mermaid
graph TD
    subgraph "Estados do Circuit Breaker"
    A[Fechado<br>Normal] -->|"Falhas > Threshold"| B[Aberto<br>Falha]
    B -->|"Timeout Expirado"| C[Semi-Aberto<br>Teste]
    C -->|"Requisição<br>Bem-Sucedida"| A
    C -->|"Falha"| B
    end
    
    style A fill:#9f9,stroke:#333,stroke-width:2px
    style B fill:#f96,stroke:#333,stroke-width:2px
    style C fill:#ff9,stroke:#333,stroke-width:2px
```

## Conexão com a Teoria das Restrições

O Circuit Breaker implementa diretamente os princípios da TOC:

1. **Identificação da Restrição**: Monitora falhas para identificar serviços comprometidos
2. **Exploração da Restrição**: No estado semi-aberto, permite tráfego limitado para testar recuperação
3. **Subordinação ao Gargalo**: Redireciona tráfego para evitar o componente com restrições
4. **Elevação da Restrição**: Tenta uso gradual do serviço para verificar se a restrição foi superada
5. **Repetição do Processo**: Monitora continuamente o estado do sistema e cicla entre estados

## Implementação Prática 

Considere uma arquitetura de microsserviços com múltiplas dependências:

```mermaid
graph LR
    subgraph "Arquitetura com Circuit Breakers"
    A[API Gateway] --> B[Serviço A]
    A --> C[Serviço B]
    A --> D[Serviço C]
    
    B --> B1[(Database<br>Interna)]
    C --> E[Serviço Externo 1]
    D --> F[Serviço Externo 2]
    
    CB1[Circuit Breaker] -.-> B
    CB2[Circuit Breaker] -.-> C
    CB3[Circuit Breaker] -.-> D
    CB4[Circuit Breaker] -.-> E
    CB5[Circuit Breaker] -.-> F
    end
```

## Análise de Restrições em Sistema Distribuído

```mermaid
graph TD
    subgraph "Identificação de Restrições"
    A[Tipos de Restrições] --> B[Temporárias<br>- Picos de carga<br>- Problemas de rede<br>- Deploys]
    
    A --> C[Persistentes<br>- Bugs<br>- Limitação de recursos<br>- Bottlenecks estruturais]
    
    A --> D[Sazonais<br>- Eventos de alto tráfego<br>- Processos periódicos<br>- Janelas de manutenção]
    end
```

## Estratégias de Fallback

A TOC sugere sempre ter estratégias alternativas para lidar com a restrição:

```mermaid
flowchart TD
    A[Circuito Aberto<br>Serviço Indisponível] --> B{Estratégia<br>de Fallback}
    
    B -->|"Cache"| C[Retornar dados<br>em cache]
    B -->|"Degradação<br>Graciosa"| D[Funcionalidade<br>reduzida]
    B -->|"Filas"| E[Enfileirar para<br>processamento posterior]
    B -->|"Redundância"| F[Redirecionar para<br>serviço alternativo]
    B -->|"Operações<br>Assíncronas"| G[Desencoplar<br>requisição/resposta]
```

## Código: Implementação de Circuit Breaker

Implementação usando a biblioteca Resilience4j:

```java
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import io.vavr.control.Try;

import java.time.Duration;
import java.util.function.Supplier;

public class PaymentServiceWithCircuitBreaker {
    
    private final PaymentGatewayClient paymentClient;
    private final CircuitBreaker circuitBreaker;
    private final PaymentCache paymentCache;
    
    public PaymentServiceWithCircuitBreaker(PaymentGatewayClient paymentClient, PaymentCache paymentCache) {
        this.paymentClient = paymentClient;
        this.paymentCache = paymentCache;
        
        // Configuração baseada em análise de restrições
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)                           // 50% de falhas para abrir
            .waitDurationInOpenState(Duration.ofSeconds(30))    // 30s no estado aberto
            .permittedNumberOfCallsInHalfOpenState(5)           // 5 chamadas de teste
            .slidingWindowSize(20)                              // Avalia as últimas 20 requisições
            .recordExceptions(
                PaymentGatewayException.class,
                TimeoutException.class
            )
            .build();
        
        CircuitBreakerRegistry registry = CircuitBreakerRegistry.of(config);
        this.circuitBreaker = registry.circuitBreaker("paymentService");
        
        // Registra listeners para ações em mudanças de estado
        circuitBreaker.getEventPublisher()
            .onStateTransition(event -> {
                if (event.getStateTransition() == StateTransition.CLOSED_TO_OPEN) {
                    notifyOperations("Serviço de pagamento indisponível!");
                } else if (event.getStateTransition() == StateTransition.OPEN_TO_CLOSED) {
                    notifyOperations("Serviço de pagamento restaurado.");
                }
            });
    }
    
    public PaymentResult processPayment(PaymentRequest request) {
        // Decorando a chamada com circuit breaker
        Supplier<PaymentResult> paymentSupplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> paymentClient.processPayment(request));
        
        // Tentativa com fallback para cache
        return Try.ofSupplier(paymentSupplier)
            .recover(throwable -> {
                if (circuitBreaker.getState() == CircuitBreaker.State.OPEN) {
                    log.warn("Circuit breaker aberto. Usando dados em cache ou processo alternativo.");
                    return getPaymentResultFallback(request);
                }
                throw new RuntimeException("Falha no processamento do pagamento", throwable);
            })
            .get();
    }
    
    private PaymentResult getPaymentResultFallback(PaymentRequest request) {
        // Implementação da TOC: Usando recurso alternativo quando o gargalo está indisponível
        
        // 1. Tentar obter do cache
        Optional<PaymentResult> cachedResult = paymentCache.getRecentPaymentResult(request.getCustomerId());
        if (cachedResult.isPresent()) {
            return cachedResult.get().markAsCached();
        }
        
        // 2. Fallback: Enfileirar para processamento posterior
        queuePaymentForLaterProcessing(request);
        
        // 3. Retornar resultado parcial/pendente
        return PaymentResult.builder()
                .status(PaymentStatus.PENDING)
                .message("Pagamento enfileirado para processamento posterior")
                .timestamp(System.currentTimeMillis())
                .build();
    }
    
    private void queuePaymentForLaterProcessing(PaymentRequest request) {
        // Implementação da subordinação: adaptar-se à restrição atual
        paymentQueue.send(request);
        log.info("Pagamento enfileirado para processamento posterior: {}", request.getId());
    }
}
```

## Monitoramento de Circuit Breakers

A efetividade dos circuit breakers deve ser constantemente monitorada:

```mermaid
graph LR
    subgraph "Métricas de Circuit Breaker"
    A[Monitoramento] --> B[Taxa de Falhas<br>% requisições falhas]
    A --> C[Mudanças de Estado<br>Aberto/Fechado]
    A --> D[Tempo no Estado<br>Aberto/Fechado]
    A --> E[Latência<br>Requisições]
    A --> F[Taxa de<br>Fallbacks]
    A --> G[Histograma de<br>Falhas]
    end
```

## Configuração Baseada em TOC

```mermaid
flowchart TD
    subgraph "Parâmetros Baseados em TOC"
    A[Circuit<br>Breaker] --> B[Threshold de Falhas]
    A --> C[Janela Deslizante<br>de Avaliação]
    A --> D[Tempo no<br>Estado Aberto]
    A --> E[Chamadas permitidas<br>no Estado Semi-Aberto]
    
    B --- B1["Define quando a<br>restrição é significativa"]
    C --- C1["Tamanho da amostra<br>para identificação"]
    D --- D1["Tempo para tentativa<br>de elevação da restrição"]
    E --- E1["Controle da taxa na<br>elevação da restrição"]
    end
```

## Padrões Avançados

### Bulkhead Pattern com Circuit Breaker

Combinando isolamento com circuit breaker:

```mermaid
graph TD
    subgraph "Bulkhead + Circuit Breaker"
    A[API Gateway] --> B[Partição 1<br>Crítica]
    A --> C[Partição 2<br>Importante]
    A --> D[Partição 3<br>Não-crítica]
    
    B --> S1[Serviço 1]
    B --> S2[Serviço 2]
    C --> S3[Serviço 3]
    D --> S4[Serviço 4]
    
    CB1[Circuit<br>Breaker] -.-> S1
    CB2[Circuit<br>Breaker] -.-> S2
    CB3[Circuit<br>Breaker] -.-> S3
    CB4[Circuit<br>Breaker] -.-> S4
    
    subgraph "Lógica TOC"
        L1[Alta Prioridade<br>Bulkhead menor<br>CB mais tolerante]
        L2[Baixa Prioridade<br>Bulkhead maior<br>CB mais rigoroso]
    end
    end
```

### Circuit Breaker Adaptativo

```mermaid
sequenceDiagram
    participant A as Serviço A
    participant CB as Circuit Breaker
    participant B as Serviço B
    participant M as Monitor TOC
    
    Note over M: Monitora restrições<br>em tempo real
    
    A->>CB: Requisição
    CB->>B: Encaminha (estado fechado)
    B-->>CB: Erro (timeout)
    CB-->>A: Propaga erro
    
    Note over CB: Falhas incrementam<br>contador
    
    M->>CB: Detecta horário de pico
    M->>CB: Ajusta threshold para 30%<br>(mais sensível)
    
    A->>CB: Nova requisição
    Note over CB: Abre circuito mais rápido<br>sob alta carga
    CB-->>A: Fallback imediato
    
    Note over M: Detecta fim do<br>horário de pico
    M->>CB: Ajusta threshold para 50%<br>(mais tolerante)
```

## Caso de Estudo: Microsserviços de E-commerce

Um sistema de e-commerce que utiliza circuit breakers seguindo TOC:

```mermaid
graph TD
    subgraph "E-commerce com Circuit Breakers"
    A[Front-end] --> B[API Gateway]
    
    B --> C[Catálogo<br>High Availability]
    B --> D[Carrinho<br>High Availability]
    B --> E[Inventário<br>Medium Availability]
    B --> F[Pagamento<br>External Dependency]
    B --> G[Recomendações<br>Non-critical]
    
    C --> C1[CB: Threshold 60%<br>Fallback: Cache Local]
    D --> D1[CB: Threshold 50%<br>Fallback: Local Storage]
    E --> E1[CB: Threshold 40%<br>Fallback: Reserva Tentativa]
    F --> F1[CB: Threshold 30%<br>Fallback: Enfileiramento]
    G --> G1[CB: Threshold 20%<br>Fallback: Desativar Feature]
    end
```

## Considerações e Best Practices

1. **Configuração Específica por Contexto**: Cada serviço tem suas próprias características e necessidades
2. **Granularidade Adequada**: Circuit breakers muito amplos ou muito específicos podem ser contraproducentes
3. **Testes sob Falha**: Simular cenários de falha regularmente (Chaos Engineering)
4. **Alertas e Monitoramento**: Notificações sobre mudanças de estado permitem intervenção proativa
5. **Reconfiguração Dinâmica**: Ajustar parâmetros do circuit breaker baseado em padrões observados

## Conclusão

O Circuit Breaker Pattern é uma aplicação direta da Teoria das Restrições em sistemas distribuídos que:

1. **Identifica Restrições**: Detecta componentes falhos ou sobrecarregados
2. **Protege Recursos**: Evita sobrecarga de componentes já comprometidos
3. **Falha Rapidamente**: Economiza recursos retornando erros imediatos quando apropriado
4. **Testa Recuperação**: Verifica periodicamente se a restrição foi superada
5. **Adapta-se Dinamicamente**: Ajusta o comportamento do sistema baseado no estado das restrições

Como resultado, sistemas que implementam corretamente este padrão conseguem manter-se operacionais mesmo quando componentes individuais falham, demonstrando resiliência e estabilidade mesmo sob condições adversas - um exemplo perfeito da TOC aplicada à arquitetura de software moderna.
