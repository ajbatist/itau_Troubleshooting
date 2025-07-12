# itau_Troubleshooting
Desafio Troubleshooting

## 1. Diagnóstico: Passos Imediatos

### 1.1 Verificar saúde do cluster e pods

Checar se há throttling, falhas ou restart loops:

bash
~~~
kubectl -n payment get pods -o wide
kubectl -n payment describe pod <pod-name>
kubectl top pod -n payment
~~~

### 1.2 Observar métricas e alertas no Grafana / Datadog

Dashboards de:

Latência da API

Erro por código HTTP

Conexões com banco de dados

Saturação de CPU/Memória

 ### 1.3 Analisar logs

 Ferramentas: Grafana Loki, Datadog Logs, kubectl logs

 Buscar por:

   - timeout

  - connection refused

  - slow query

  - pool exhausted


bash
~~~
kubectl logs -n payment -l app=payment-api --since=10m

~~~

###  1.4 Validar acesso ao banco (RDS)

Verificar:

  - Conectividade

  - Latência de conexão

  - Erros de timeout na aplicação

## 2. Ferramentas e Comandos
~~~
| Finalidade      | Ferramenta                    | Comando/uso                     |
| --------------- | ----------------------------- | ------------------------------- |
| Métricas        | Grafana / Datadog             | Dashboards + alertas            |
| Logs            | `kubectl logs`, Loki, Datadog | `--since=10m`                   |
| Pods e recursos | `kubectl`                     | `top`, `describe`, `get events` |
| APM             | OpenTelemetry, Datadog APM    | Traces por endpoint             |
| Banco de dados  | RDS Insights                  | Slow queries, conexões          |
| ArgoCD          | Argo UI / CLI                 | Confirmar estado da release     |

~~~

## 3. Hipóteses Prováveis

1. Problemas de conexão com RDS:

  - Exaustão de conexões

  - Latência alta da VPC ou subnet

  - Lock em queries pesadas

2. Recursos saturados no pod ou cluster:

  - CPU/memória insuficiente (throttling)

  -  Pool de conexões não dimensionado

3. Picos de carga:
   
  - Tráfego aumentou sem escala proporcional
    
4. Problemas de rede (NAT, DNS, etc.):
   
  - NAT Gateway saturado (muito comum com RDS)

##  4. Ações Imediatas para Mitigar Impacto
~~~
| Ação                                         | Justificativa                                |
| -------------------------------------------- | -------------------------------------------- |
| Escalar réplicas de `payment-api`            | Reduz impacto e ajuda a atender tráfego      |
| Aumentar pool de conexões (se configurável)  | Evitar timeout no acesso ao RDS              |
| Reiniciar pods com falha no acesso DB        | Liberar conexões presas                      |
| Investigar o RDS: slow queries e locks       | Aliviar gargalo real                         |
| Habilitar autoscaling (caso aplicável)       | Lidar com carga flutuante                    |
| Rota fallback (ex: fila ou resposta default) | Se disponível, garantir continuidade parcial |

~~~

## 5. Comunicação Durante o Incidente
~~~
| Canal                  | Mensagem (exemplo)                                                                                                                       |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Slack #ops-alerts      | “🚨 Incidente em produção: `payment-api` com alta latência (>2s) e erros de banco detectados. Investigando. Atualizações a cada 15 min.” |
| Atualizações regulares | Status do diagnóstico, ações tomadas e próximas etapas                                                                                   |
| Finalização            | Causa raiz, impacto, mitigação e ações preventivas                                                                                       |

~~~

##  6. Documentação Pós-Incidente (PIR - Post Incident Review)

### Sumário

  - Serviço afetado: payment-api

  - Ambiente: Produção

  - Data/Hora de início e fim do incidente

  - Impacto: Processamento de pagamentos afetado

### Linha do tempo
~~~
| Horário | Evento                                            |
| ------- | ------------------------------------------------- |
| 10:05   | Alerta de latência acionado                       |
| 10:10   | Logs indicam timeouts com RDS                     |
| 10:15   | Confirmado gargalo no pool de conexões            |
| 10:30   | Escaladas réplicas e aumentada capacidade do pool |
| 10:45   | Latência estabilizada                             |

~~~

### Causa Raiz

Aumento de carga → saturação do pool de conexões com RDS → timeouts

### Melhorias preventivas
~~~
| Ação                                      | Status     |
| ----------------------------------------- | ---------- |
| Implementar autoescalonamento de pods     | 🔧 A fazer |
| Monitorar métricas de pool de conexão     | 🔧 A fazer |
| Aumentar limite e timeout do DB           | ✅ Feito    |
| Dashboards com alertas de saturação de DB | ✅ Feito    |
| Configurar fallback para erro de DB       | 🔧 A fazer |

~~~

## Modelo de Incidente

### Resumo do Incidente
~~~
- **Serviço impactado:** `payment-api`
- **Ambiente:** Produção (`EKS-prod`)
- **Namespace:** `payment`
- **Data/Hora de Início:** YYYY-MM-DD HH:MM
- **Data/Hora de Fim:** YYYY-MM-DD HH:MM
- **Duração:** XX minutos
- **Impacto:** Alta latência e falhas no processamento de pagamentos. Serviço apresentou >2s de resposta por 10 minutos.

~~~

### Linha do Tempo
~~~
| Horário | Evento |
|--------|--------|
| 10:05  | Alerta de latência da API acionado (> 2s por 10min) |
| 10:10  | Logs mostram timeout na comunicação com RDS |
| 10:15  | Confirmado gargalo no pool de conexões com RDS |
| 10:30  | Escalado número de réplicas + ajuste do pool |
| 10:45  | Métricas estabilizadas, latência voltou ao normal |
~~~

### Causa Raiz

 Aumento de carga inesperado levou à saturação do pool de conexões com o banco RDS. Como consequência, a aplicação apresentou timeouts e aumento expressivo na latência média das requisições HTTP.

 ### Detecção
~~~
- Alerta automático de **latência média > 2s**
- Logs com erros `timeout` ao acessar banco
- Traces via OpenTelemetry mostraram atraso no acesso a serviços externos
~~~
 
### Ações Tomadas
~~~
| Ação | Status |
|------|--------|
| Verificação de pods e métricas de recursos | ✅ |
| Escalonamento das réplicas de `payment-api` | ✅ |
| Ajuste no pool de conexões da aplicação | ✅ |
| Verificação de locks/slow queries no RDS | ✅ |
| Análise de logs e traces distribuídos | ✅ |
~~~

### Hipóteses Verificadas
~~~
- [x] Saturação do pool de conexões com RDS
- [ ] Problemas na rede (NAT Gateway)
- [ ] Mudança recente no código (não houve)
- [ ] Sobrecarga de CPU/memória
~~~

### Lições Aprendidas
~~~
- A saturação de banco é silenciosa, mas crítica
- Métricas de conexão com banco não estavam sendo monitoradas
- Escalabilidade horizontal ajudou, mas o gargalo era o pool fixo
~~~

### Melhorias Preventivas
~~~
| Ação Preventiva | Status |
|-----------------|--------|
| Adicionar monitoramento do uso do pool de conexões | 🔧 A fazer |
| Habilitar HPA baseado em latência | 🔧 A fazer |
| Melhorar limites de recurso dos pods | ✅ Feito |
| Implementar fallback para falha no banco | 🔧 A fazer |
| Adicionar alertas para `DBConnectionTimeout` | ✅ Feito |
~~~

### Comunicação
~~~
- Canal: `#ops-alerts`, reuniões de status a cada 15 minutos
- Atualizações contínuas foram enviadas até a normalização
- Incidente compartilhado com time de engenharia e SRE
~~~

### Artefatos Relacionados
~~~
- Logs e métricas no Grafana: [link]
- Traces de requisições: [link]
- Pull request de ajuste do pool: [link]
- Configuração ArgoCD: [link]
~~~

**Revisado por:** Alexandre José Batista  
**Data:** 14/07/2025

