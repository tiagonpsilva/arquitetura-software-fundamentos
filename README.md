# Fundamentos de Arquitetura de Software

Este repositório apresenta uma introdução detalhada sobre três conceitos fundamentais para arquitetura de software: Teoria das Filas, Teoria das Restrições e Máquinas de Estado.

## Importância na Arquitetura de Software

Arquiteturas de software modernas lidam com sistemas distribuídos complexos que precisam gerenciar recursos limitados, processar grandes volumes de requisições, e manter estados consistentes em diversas condições. Os três conceitos abordados neste repositório fornecem fundamentos teóricos e práticos essenciais para enfrentar esses desafios:

### Teoria das Filas

A Teoria das Filas oferece modelos matemáticos para analisar e otimizar o fluxo de requisições em sistemas, permitindo:

- Dimensionar adequadamente recursos como servidores, threads e conexões
- Prever comportamento do sistema sob diferentes cargas
- Implementar estratégias de throttling e backpressure
- Otimizar balanceamento de carga
- Implementar modelos de escalabilidade eficientes

### Teoria das Restrições

A Teoria das Restrições fornece um framework para identificar e gerenciar gargalos em sistemas complexos:

- Identificar recursos limitantes na arquitetura
- Maximizar throughput em sistemas distribuídos
- Implementar estratégias de circuit breaker e fallback
- Otimizar fluxos de processamento de dados
- Priorizar melhorias de performance com maior impacto

### Máquinas de Estado

Máquinas de Estado fornecem um modelo formal para comportamentos orientados a eventos e transição entre estados:

- Modelar workflows complexos de forma precisa e verificável
- Implementar lógica de coordenação em sistemas distribuídos
- Gerenciar ciclos de vida de objetos, processos e transações
- Garantir consistência em processamentos assíncronos
- Implementar lógica de recuperação e resiliência

## Estrutura do Repositório

Este repositório está organizado em três seções principais, cada uma dedicada a um dos conceitos fundamentais:

- **[Teoria das Filas](./teoria-das-filas/)**: Modelos, implementações e casos de uso
- **[Teoria das Restrições](./teoria-das-restricoes/)**: Princípios, aplicações e práticas
- **[Máquinas de Estado](./maquinas-de-estado/)**: Modelagem, implementação e padrões

Cada seção contém 5 exemplos práticos ilustrando a aplicação dos conceitos em cenários reais de arquitetura de software, com diagramas explicativos.

## Como Usar este Repositório

Este material pode ser utilizado como:

- Guia de referência para arquitetos de software
- Material de estudo para engenheiros e desenvolvedores
- Base para implementações práticas dos conceitos apresentados
- Fonte de inspiração para soluções de problemas arquiteturais

Navegue pelas pastas para acessar os exemplos específicos de cada tema.
