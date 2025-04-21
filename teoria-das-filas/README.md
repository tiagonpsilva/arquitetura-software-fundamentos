# Teoria das Filas na Arquitetura de Software

A Teoria das Filas é um ramo da matemática que estuda o comportamento de sistemas onde entidades (clientes, requisições, pacotes de dados) aguardam em filas para serem processadas. Na arquitetura de software, esta teoria fornece modelos para análise e otimização de sistemas que lidam com múltiplas requisições concorrentes.

## Conceitos Fundamentais

- **Chegadas (Arrival)**: Processo que determina como as requisições chegam ao sistema
- **Filas (Queues)**: Estruturas que armazenam as requisições pendentes
- **Servidores (Servers)**: Recursos que processam as requisições
- **Disciplina de Atendimento**: Estratégia para selecionar a próxima requisição a ser processada
- **Tempo de Serviço**: Duração do processamento de cada requisição

## Notação de Kendall

A notação de Kendall (A/S/c/K/N/D) é usada para classificar sistemas de filas:

- **A**: Distribuição de chegada (M = Markoviana/Poisson, D = Determinística, G = Geral)
- **S**: Distribuição de tempo de serviço
- **c**: Número de servidores
- **K**: Capacidade do sistema (máximo de requisições)
- **N**: Tamanho da população
- **D**: Disciplina de atendimento (FIFO, LIFO, etc.)

## Aplicações na Arquitetura de Software

- Dimensionamento de pools de threads e conexões
- Configuração de timeouts e retries
- Estratégias de throttling e rate limiting
- Modelos de caching e buffers
- Padrões de balanceamento de carga
- Sistemas de mensageria e filas de trabalhos assíncronos

## Métricas Essenciais

- **Throughput**: Taxa de processamento de requisições
- **Latência**: Tempo de resposta para atender uma requisição
- **Utilização**: Percentual de uso dos recursos disponíveis
- **Comprimento da Fila**: Número médio de requisições aguardando
- **Probabilidade de Bloqueio**: Chance de uma nova requisição ser rejeitada

## Exemplos Neste Diretório

1. [Modelo de Fila Simples (M/M/1)](./exemplo-01-fila-simples-mm1.md)
2. [Load Balancing com Múltiplos Servidores](./exemplo-02-load-balancing.md)
3. [Backpressure em Sistemas Distribuídos](./exemplo-03-backpressure.md)
4. [Filas Prioritárias para QoS](./exemplo-04-filas-prioritarias.md)
5. [Dimensionamento de Pools de Conexão](./exemplo-05-pool-conexoes.md)

Cada exemplo demonstra cenários práticos de aplicação da Teoria das Filas em desafios reais de arquitetura de software, com diagramas explicativos e considerações de implementação.
