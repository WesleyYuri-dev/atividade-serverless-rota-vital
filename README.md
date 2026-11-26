# Atividade Individual 1 · Serverless (Rota Vital)

Implementação e teste local de uma função serverless utilizando o AWS SAM CLI e emulação local com Docker Desktop no Windows, simulando o fluxo de emissão de guias de transporte de hemocomponentes do sistema Rota Vital.

## Caminho Escolhido

* Ambiente Local: Emulação completa via AWS SAM CLI (`sam local invoke` e `sam local start-api`) com contêineres Docker (`public.ecr.aws/lambda/python:3.12-rapid-x86_64`), sem necessidade de provisionamento em conta AWS na nuvem.

## Como Rodar o Projeto

1. Compilar: `sam build`
2. Invocar com evento: `sam local invoke -e events/pedido.json`
3. Subir API local (sem reuso): `sam local start-api`
4. Subir API local (aquecida): `sam local start-api --warm-containers LAZY`
5. Testar via curl: `curl -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" -d '{"requisicaoId": "R-002", "hospital":"H-Caruaru"}'`

---
## Respostas das Perguntas

### 1. Por que a primeira invocação demorou mais que a segunda? O que acontece na fase de Init?

Nos testes via `start-api`, a primeira requisição levou 60,20 ms, enquanto as seguintes no contêiner aquecido caíram para 1,09 ms – 1,31 ms (e no `local invoke`, os tempos variaram entre 51,49 ms e 170,90 ms conforme a inicialização do Docker). Isso ocorre pelo Cold Start: na fase de Init, o ambiente provisiona o contêiner, carrega o runtime Python 3.12 e executa o código fora do `lambda_handler` (como `contador = 0`). Nas chamadas seguintes (Warm Start), essa etapa é pulada.

### 2. O que o campo `invocacaoNesteAmbiente` mostrou com e sem `--warm-containers`? O que isso diz sobre guardar estado numa variável global da função?

Sem `--warm-containers`, o campo retornou 1 em todas as chamadas, pois o contêiner era destruído a cada resposta. Com `--warm-containers LAZY`, subiu sequencialmente (1, 2, 3), preservando a variável global em memória. Isso mostra que funções serverless são stateless por natureza: variáveis globais não devem guardar estado crítico de negócio, pois o provedor encerra contêineres ociosos ou cria várias instâncias paralelas isoladas.

### 3. O que aconteceu com `demorar = true`? Qual é o tempo máximo de uma execução no Lambda, e o que fazer com um trabalho mais longo?

Com `"demorar": true`, o `time.sleep(5)` ultrapassou o `Timeout: 3` do `template.yaml`, encerrando a função aos 3000,00 ms com erro HTTP 502. Ao alterar o `Timeout` para 10, concluiu em 5049,90 ms (HTTP 201). O tempo máximo de execução no AWS Lambda é de 15 minutos (900 segundos). Trabalhos mais longos devem ser quebrados com filas (Amazon SQS), orquestrados via AWS Step Functions ou movidos para contêineres (AWS Fargate/ECS ou AWS Batch).

### 4. Com os preços de `us-east-1` da aula (US$ 0,20 por milhão de requisições e US$ 0,0000166667 por GB-s), quanto custa um milhão de execuções desta função com 128 MB e 100 ms? Mostrem a conta.

* Custo por requisições: 1 × US$ 0,20 = US$ 0,20
* Memória: 128 MB ÷ 1024 = 0,125 GB
* Tempo total: 1.000.000 × 0,1 s = 100.000 s
* Computação em GB-s: 100.000 s × 0,125 GB = 12.500 GB-s
* Custo de computação: 12.500 × US$ 0,0000166667 = US$ 0,20833375
* Custo Total (sem Free Tier): US$ 0,20 + US$ 0,20833375 = US$ 0,4083 (aproximadamente US$ 0,41).

### 5. Em que parte do Rota Vital uma função assim faria sentido, e em qual não faria?

Faz sentido em eventos rápidos e pontuais, como emissão/validação da guia de transporte (`POST /guias`), leitura de QR Code de bolsas ou envio de notificações aos hospitais. Não faz sentido no rastreamento GPS contínuo em tempo real das ambulâncias (que exige conexões persistentes via WebSocket/MQTT) nem em otimizações pesadas de rotas em lote que ultrapassem 15 minutos.

## Evidências

Os prints de execução encontram-se na pasta `evidencias/`.

## Uso de IA (PO-CESAR-021)

* Ferramenta: Gemini (Google).
* Finalidade: Apoio na configuração do AWS SAM CLI no Windows/Git Bash, revisão do `template.yaml` e estruturação do README.
* Validação: Todos os comandos e resultados foram executados localmente na máquina do autor e comprovados nos logs da pasta `evidencias/`.