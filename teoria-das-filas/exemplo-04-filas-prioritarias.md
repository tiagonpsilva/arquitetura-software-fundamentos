# Exemplo 4: Filas Prioritárias para QoS

Filas prioritárias (Priority Queues) são estruturas fundamentais para implementar Quality of Service (QoS) em sistemas distribuídos. Aplicando conceitos da teoria das filas, elas permitem que requisições mais críticas sejam processadas antes de requisições menos importantes, mesmo que estas tenham chegado anteriormente.

## Modelo de Filas Prioritárias

O modelo de filas prioritárias estende os modelos tradicionais associando classes de prioridade às requisições:

```mermaid
graph LR
    A1[Requisições<br>Alta Prioridade] -->|λ1| H[Fila<br>Alta Prioridade]
    A2[Requisições<br>Média Prioridade] -->|λ2| M[Fila<br>Média Prioridade]
    A3[Requisições<br>Baixa Prioridade] -->|λ3| L[Fila<br>Baixa Prioridade]
    
    H --> S[Servidores<br>μ]
    M --> S
    L --> S
    
    S --> D[Saída]
    
    style H fill:#f55,stroke:#333,stroke-width:2px
    style M fill:#fc5,stroke:#333,stroke-width:2px
    style L fill:#5a5,stroke:#333,stroke-width:2px
```

## Disciplinas de Atendimento

Existem diferentes estratégias para implementar a priorização:

### 1. Prioridade Estrita (Preemptiva)

```mermaid
flowchart LR
    subgraph "Prioridade Preemptiva"
    A[Nova requisição<br>de alta prioridade] -->|Interrompe| B[Processamento<br>atual]
    B -->|Retorna para fila| C[Requisição de<br>menor prioridade]
    end
```

### 2. Prioridade Não-Preemptiva

```mermaid
flowchart LR
    subgraph "Prioridade Não-Preemptiva"
    A[Nova requisição<br>de alta prioridade] -->|Aguarda| B[Conclusão do<br>processamento atual]
    B -->|Em seguida| C[Processa requisição<br>de alta prioridade]
    end
```

### 3. Weighted Fair Queuing

```mermaid
flowchart TD
    subgraph "Weighted Fair Queuing"
    A[Atribuição de pesos<br>Alta: 70%<br>Média: 20%<br>Baixa: 10%] -->|Distribui capacidade| B[Processamento<br>Compartilhado]
    end
```

## Análise Matemática

Para um sistema com N classes de prioridade (não preemptivo):

- **Taxa de chegada total**: λ = λ₁ + λ₂ + ... + λₙ
- **Utilização**: ρ = λ/μ
- **Utilização por classe**: ρᵢ = λᵢ/μ

O tempo médio de espera para cada classe de prioridade é:

```
W₁ = (ρ/μ)/(1-ρ₁)
W₂ = (ρ/μ)/((1-ρ₁)(1-ρ₁-ρ₂))
...
Wᵢ = (ρ/μ)/((1-ρ₁-...-ρᵢ₋₁)(1-ρ₁-...-ρᵢ))
```

Observe que o tempo de espera aumenta consideravelmente para prioridades mais baixas conforme a utilização do sistema cresce.

## Implementação em Arquiteturas de Microserviços

```mermaid
graph TD
    subgraph "API Gateway com Filas Prioritárias"
    A[Requisições] --> B[Classificador<br>de Tráfego]
    
    B -->|Premium| C[Fila P1<br>SLA: 100ms]
    B -->|Regular| D[Fila P2<br>SLA: 500ms]
    B -->|Background| E[Fila P3<br>SLA: 2000ms]
    
    C --> F[Pool de<br>Processamento]
    D --> F
    E --> F
    
    F --> G[Microsserviços<br>Backend]
    end
```

## Caso de Estudo: Sistema E-commerce

Considere um sistema de e-commerce durante uma promoção especial:

```mermaid
flowchart LR
    subgraph "E-commerce com QoS"
    A[Tráfego de entrada<br>10.000 req/s] --> B{Classificador}
    
    B -->|20%| C[Checkout<br>Prioridade 1]
    B -->|30%| D[Carrinho<br>Prioridade 2]
    B -->|50%| E[Navegação<br>Prioridade 3]
    
    C --> F[Processadores<br>Compartilhados]
    D --> F
    E --> F
    end
```

### Análise de Desempenho

Com uma capacidade de processamento de 12.000 req/s:

```mermaid
graph TD
    subgraph "Tempos de Resposta por Classe"
    A[Checkout<br>μ = 2.000 req/s<br>W = 45ms] 
    B[Carrinho<br>μ = 3.000 req/s<br>W = 120ms]
    C[Navegação<br>μ = 5.000 req/s<br>W = 350ms]
    
    D[Diferença de latência<br>entre classes:]
    D --> E[Checkout vs Navegação:<br>8x mais rápido]
    D --> F[Carrinho vs Navegação:<br>3x mais rápido]
    end
```

## Código: Implementação de Filas Prioritárias

### Exemplo em Java com ThreadPoolExecutor

```java
import java.util.concurrent.*;

public class PriorityQueueService {
    // Define prioridades
    public enum RequestPriority {
        HIGH(0),
        MEDIUM(5),
        LOW(10);
        
        private final int value;
        
        RequestPriority(int value) {
            this.value = value;
        }
        
        public int getValue() {
            return value;
        }
    }
    
    // Requisição com prioridade
    public static class PriorityRequest implements Runnable, Comparable<PriorityRequest> {
        private final Runnable task;
        private final RequestPriority priority;
        private final long timestamp;
        
        public PriorityRequest(Runnable task, RequestPriority priority) {
            this.task = task;
            this.priority = priority;
            this.timestamp = System.currentTimeMillis();
        }
        
        @Override
        public void run() {
            task.run();
        }
        
        @Override
        public int compareTo(PriorityRequest other) {
            // Primeiro compara por prioridade
            int priorityCompare = Integer.compare(priority.getValue(), other.priority.getValue());
            if (priorityCompare != 0) {
                return priorityCompare;
            }
            // Para mesma prioridade, usa FIFO
            return Long.compare(timestamp, other.timestamp);
        }
    }
    
    // Executor com fila de prioridades
    private final ThreadPoolExecutor executor;
    
    public PriorityQueueService(int corePoolSize, int maxPoolSize) {
        // Usa PriorityBlockingQueue para ordenar por prioridade
        BlockingQueue<Runnable> queue = new PriorityBlockingQueue<>();
        executor = new ThreadPoolExecutor(
            corePoolSize, 
            maxPoolSize,
            60L, TimeUnit.SECONDS,
            queue
        );
    }
    
    public void submit(Runnable task, RequestPriority priority) {
        executor.execute(new PriorityRequest(task, priority));
    }
    
    public void shutdown() {
        executor.shutdown();
    }
}
```

## Monitoramento e Métricas

Para garantir que as filas prioritárias estejam funcionando adequadamente:

```mermaid
graph TB
    subgraph "Monitoramento de QoS"
    A[Métricas por<br>Classe de Prioridade] --> B[Tempo de Resposta]
    A --> C[Throughput]
    A --> D[Taxa de Abandono]
    A --> E[Profundidade da Fila]
    
    F[Alertas] --> G[SLA Violation<br>P1 > 100ms]
    F --> H[Queue Depth<br>P3 > 1000]
    F --> I[Starvation<br>P3 wait > 30s]
    end
```

## Estratégias Avançadas

### Aging (Envelhecimento)

Para evitar starvation de requisições de baixa prioridade:

```mermaid
graph TD
    subgraph "Mecanismo de Aging"
    A[Requisição de<br>baixa prioridade] --> B{Tempo de espera<br>>threshold?}
    B -->|Não| C[Mantém prioridade]
    B -->|Sim| D[Aumenta prioridade]
    end
```

### Rate Limiting por Classe

```mermaid
flowchart LR
    A[Cliente Premium] -->|Limite: 1000 req/min| B[API]
    C[Cliente Regular] -->|Limite: 100 req/min| B
    D[Cliente Free] -->|Limite: 10 req/min| B
```

## Considerações de Design e Pitfalls

1. **Head-of-Line Blocking**: Em sistemas de alta carga, requisições de baixa prioridade podem nunca ser atendidas
2. **Complexidade**: O sistema de priorização adiciona overhead
3. **Propagação de Prioridade**: Em sistemas distribuídos, a prioridade deve ser propagada através de toda a cadeia de serviços
4. **Deadlocks**: Requisições de alta prioridade podem depender de recursos bloqueados por requisições de baixa prioridade

## Conclusão

Filas prioritárias são essenciais para sistemas que precisam fornecer diferentes níveis de serviço. A teoria das filas fornece o embasamento matemático para dimensionar e configurar esses sistemas corretamente, garantindo:

1. **Previsibilidade**: SLAs consistentes para diferentes categorias de usuários/requisições
2. **Resiliência**: Proteção de funcionalidades críticas durante picos de carga
3. **Otimização de Recursos**: Uso mais eficiente da capacidade disponível
4. **Diferenciação de Serviço**: Capacidade de oferecer diferentes níveis de qualidade

Em arquiteturas de software modernas, filas prioritárias são implementadas em múltiplas camadas, desde load balancers e API gateways até sistemas de mensageria e pools de threads internos.
