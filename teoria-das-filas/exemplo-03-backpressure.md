# Exemplo 3: Backpressure em Sistemas Distribuídos

Backpressure (contrapressão) é um mecanismo fundamental de controle de fluxo em sistemas distribuídos, derivado diretamente dos princípios da teoria das filas. Este mecanismo permite que componentes mais lentos sinalizem para componentes mais rápidos que diminuam a taxa de envio de mensagens ou requisições.

## O Problema da Diferença de Capacidade

Em sistemas distribuídos, é comum que diferentes componentes operem em velocidades diferentes:

```mermaid
graph LR
    A[Produtor Rápido<br>1000 msg/s] -->|Sobrecarga| B[Consumidor Lento<br>200 msg/s]
    
    style A fill:#f9d,stroke:#333,stroke-width:2px
    style B fill:#bbf,stroke:#333,stroke-width:2px
```

Sem backpressure, o consumidor lento será sobrecarregado, levando a:
- Crescimento ilimitado da fila
- Aumento do uso de memória
- Timeout em requisições
- Eventual falha do sistema

## Modelagem com Teoria das Filas

O backpressure pode ser modelado como um sistema de filas com capacidade limitada (M/M/1/K):

```mermaid
flowchart LR
    A[Produtor<br>λ] -->|"Rejeita se<br>fila cheia"| Q[(Fila<br>Tamanho K)]
    Q --> S[Consumidor<br>μ]
    
    style A fill:#f9d,stroke:#333,stroke-width:2px
    style Q fill:#bbf,stroke:#333,stroke-width:2px
    style S fill:#bfb,stroke:#333,stroke-width:2px
```

Quando a fila atinge sua capacidade máxima (K), o sistema aplica contrapressão:
1. **Rejeição**: Recusa novas requisições
2. **Throttling**: Limita a taxa de envio do produtor
3. **Sinalização**: Informa o produtor para reduzir a taxa

## Técnicas de Implementação

### 1. Controle baseado em créditos

```mermaid
sequenceDiagram
    participant P as Produtor
    participant C as Consumidor
    
    C->>P: Concede N créditos iniciais
    P->>C: Envia mensagem (gasta 1 crédito)
    P->>C: Envia mensagem (gasta 1 crédito)
    C->>P: Processa e devolve 2 créditos
    P->>C: Envia mensagem (gasta 1 crédito)
    Note over P,C: Produtor para de enviar<br>quando sem créditos
```

### 2. Filas com Capacidade Limitada

```mermaid
graph TD
    subgraph "Sistema com Backpressure"
    A[Producer] -->|"envia()"| B[Bounded Queue<br>max_size=100]
    B -->|"consome()"| C[Consumer]
    
    D[Quando fila cheia] -->|"1. Bloqueia produtor<br>2. Lança exceção<br>3. Descarta mensagens"| E[Estratégias]
    end
```

### 3. Taxa Adaptativa

```mermaid
flowchart LR
    A[Produtor] --> B[Monitor de<br>Backpressure]
    B -->|"ajusta taxa"| A
    B -->|"monitora"| C[Fila]
    C --> D[Consumidor]
    
    subgraph "Ajuste de Taxa"
    E["
    taxa = min(max_taxa, 
      taxa_base * (1 - ocupação_fila/max_tamanho))
    "]
    end
```

## Aplicação em Arquiteturas de Microsserviços

O backpressure é essencial em microsserviços para preservar a estabilidade do sistema:

```mermaid
graph TD
    subgraph "Arquitetura com Backpressure"
    A[API Gateway] -->|"Rate Limiting"| B[Serviço A]
    B -->|"Circuit Breaker"| C[Serviço B]
    C -->|"Bounded Queue"| D[Banco de Dados]
    
    E[Monitoramento]
    E -->|"observa"| A
    E -->|"observa"| B
    E -->|"observa"| C
    E -->|"observa"| D
    E -->|"ajusta limites"| A
    
    F[Telemetria de pressão<br>- Latência<br>- Tamanho de fila<br>- Taxa de erro<br>- Utilização de CPU]
    end
```

## Exemplo Prático: Reactive Streams

A especificação Reactive Streams implementa backpressure de forma padronizada:

```mermaid
sequenceDiagram
    participant S as Subscriber
    participant P as Publisher
    
    S->>P: subscribe(Subscriber)
    P->>S: onSubscribe(Subscription)
    S->>P: request(n) [demanda inicial]
    P->>S: onNext(item1)
    P->>S: onNext(item2)
    Note over S,P: Subscriber processa itens
    S->>P: request(m) [mais itens]
    P->>S: onNext(item3)
    P->>S: onComplete()
```

## Exemplo de Código: Implementação em Reactive Streams

```java
public class BackpressureExample {
    public static void main(String[] args) {
        // Criando um fluxo de 1000 inteiros com backpressure
        Flowable<Integer> source = Flowable.range(1, 1000)
            .doOnNext(i -> System.out.println("Emitido: " + i));
            
        // Consumidor lento que processa cada item em 100ms
        source.observeOn(Schedulers.io())
              .subscribe(new Subscriber<Integer>() {
                  private Subscription subscription;
                  
                  @Override
                  public void onSubscribe(Subscription s) {
                      this.subscription = s;
                      // Solicita inicialmente apenas 5 itens
                      subscription.request(5);
                  }
                  
                  @Override
                  public void onNext(Integer item) {
                      System.out.println("Recebido: " + item);
                      try {
                          // Simula processamento lento
                          Thread.sleep(100);
                      } catch (InterruptedException e) {
                          e.printStackTrace();
                      }
                      
                      // Solicita mais um item após o processamento
                      subscription.request(1);
                  }
                  
                  @Override
                  public void onError(Throwable t) {
                      t.printStackTrace();
                  }
                  
                  @Override
                  public void onComplete() {
                      System.out.println("Concluído!");
                  }
              });
              
        // Mantém o programa rodando
        try {
            Thread.sleep(60000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

## Análise Matemática

Para um sistema com backpressure modelado como M/M/1/K:

- **Taxa efetiva de chegada (λ')**:  
  λ' = λ × (1 - PK) onde PK é a probabilidade da fila estar cheia

- **Probabilidade de rejeição (PK)**:  
  PK = (1-ρ)ρᴷ/(1-ρᴷ⁺¹) onde ρ = λ/μ

- **Throughput máximo**:  
  T = μ × (1 - P₀) onde P₀ é a probabilidade do sistema estar vazio

## Benefícios do Backpressure

```mermaid
graph TD
    A[Benefícios do<br>Backpressure] --> B[Estabilidade]
    A --> C[Resiliência]
    A --> D[Previsibilidade]
    A --> E[Utilização<br>Ótima]
    A --> F[Degradação<br>Graciosa]
    
    B --> B1[Evita falhas<br>por sobrecarga]
    C --> C1[Mantém o sistema<br>funcional sob pressão]
    D --> D1[Latência mais<br>consistente]
    E --> E1[Recursos usados<br>eficientemente]
    F --> F1[Prioriza tráfego<br>mais importante]
```

## Conclusão

O backpressure baseado em teoria das filas é um padrão essencial para sistemas distribuídos, garantindo:

1. **Robustez**: Proteção contra sobrecarga
2. **Estabilidade**: Evita cascata de falhas 
3. **Performance**: Otimização de recursos
4. **Escalabilidade**: Adaptação automática à carga

Arquiteturas modernas como microsserviços, stream processing e sistemas reativos dependem fundamentalmente do backpressure para manter a integridade operacional do sistema sob condições variáveis de carga.
