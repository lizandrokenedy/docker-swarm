Tutorial Docker Swarm 

 

Para ativa o modo swarm do docker: docker swarm init 

Recupera a instrução de como inserir novos managers: docker swarm join-token manager 

Comando para exibir todos os nós que fazem parte deste swarm: docker node ls 

Comando para criar um serviço: docker service create --replicas 3 <nome da imagem> <command> 

Obter lista de tarefas criadas para um serviço: docker service ps <nome ou id do serviço> 

Escalar um serviço aumentando seu número de réplicas: docker service update <nome ou id do serviço> --replicas 3 

 

Comando para escalar um serviço:  

Cria o serviço: docker service create –p 8080:80 –-name web nginx:1.12.2 

Escala o serviço: docker service scale web=6 

Atualiza a imagem do serviço: docker service update –-image nginx:1.14.2 web 

Atualizar uma porta: docker service update --publish-rm 8080 --publish-add 9090:80 web 

Nos casos de atualização, o swarm irá remover as tarefas atuais e irá cria-lás novamente 

 

Rebalanceamento de tarefas: docker service update --force web 

Este comando irá remover todas as tarefas existentes e criará novas tarefas, balanceando a distribuição de tarefas entre os nós disponíveis no swarm 

 

Remover um serviço: docker service rm <nome ou id do serviço> 

Para listar todos os nós: docker node ls 

Promover um nó para manager: docker node update --role manager <hotsname> 

Criar network overlay: docker network create --driver overlay <nome da network> 

 

Arquivos utilizados para stacks necessitam da versão  >= 3.0 no arquivo 

Comando para fazer o deploy de um arquivo .yml: docker stack deploy –c <nome do arquivo.yml> <nome para aplicação> 

Comando para exibir uma lista dos stacks em execução: docker stack ls 

Comando para exibir as tarefas do stack: docker stack ps <nome do stack> 

Comando para exibir os serviços do stack: docker stack services <nome do stack> 

Exemplo de arquivo docker-stack.yml: 

    version: “3” 

    services: 

        redis: 

             image: redis:alpine 

             ports: 

                - “6379” 

             networks: 

                - frontend 

            deploy: 

                replicas: 1 

                update_config: 

                    parallelism: 2 # Define quantos containers poderão ser atualizados em paralelo, por padrão é 1 

                    delay: 10s # Define um intervalo entre a atualização de um grupo de containers e o próximo grupo 

                restart_policy: 

                    condition: on-failure # Reinicia o serviço automaticamente em caso de falha 

                    delay: 10s 

                    max_attempts: 3 # Em caso de falha de um container, este parâmetro irá informar a quantidade de tentativas que o docker fará para restaurar o serviço 

                    window: 120s # Quanto tempo esperar até decidir se uma reinicialização foi bem sucedida. O padrão é imediatamente. 

                placement: 

                    constraints: [node.role == manager]  # Restrição o serviço só poderá rodar em nós do tipo manager 

           

 

Comando para criar um secret no banco de dados do swarm: docker secret create psql-user psql-user.txt 

echo “password” | docker secret create psql-pass - 

Um secret pode ser criado destas duas formas, informando o nome do secret e o arquivo contendo o valor, ou a partir do input usando echo e passando a flag – ao final da instrução. 

Estas duas formas não são seguras para criação de um secret porque a deixa armazenado o secret na máquina em um arquivo, ou se for utilizado através do echo no histórico do terminal 

A maneira mais segura é utilizar a api remota do docker e passar o arquivo desta maneira 

 

Comando utilizado para listar os secrets criados: docker secret ls 

 

Comando utilizado para inspecionar um secret: docker secret inspect <nome do secret> 

 

O acesso ao conteúdo de um secret só será possível se o serviço for explicitamente autorizado 

O comando para autorizar o acesso de um secret na criação de um serviço é através da flag: --secret <nome do secret> 

Exemplo: docker service create --name psql –-secret psql-user –-secret psql-pass –e POSTGRES_PASSWORD_FILE=/run/secrets/psql-pass  -e POSTGRES_USER_FILE=/run/secrets/psql-user postgres 

Para remover um secret utilize o comando: docker service update –-secret-rm <nome do secret> <nome do serviço> 

Para adicionar um secret utilize: docker service update –-sercret-add <nome do secret> <nome do serviço>  

Ao adicionar ou atualizar um secret, o docker irá remover e recriar o container, esse comportamento não é recomendado. 

 

Para utilizar secrets em arquivos .yml, é necessária a versão >= 3.1 no arquivo. 

 

Para utilizar um secret em um arquivo docker-stack.yml é necessária a versão >= 3.1 no arquivo: 

    version: “3.5” 

    services: 

        psql: 

            image: postgres 

            secrets:  

              - psql-user 

              - psql-pass 

            environment: 

                POSTGRES_PASSWORD_FILE: /run/secrets/psql-pass 

                POSTGRES_USER_FILE: /run/secrets/psql-user 

            secrets: 

                psql-user: 

                    external: 

                        name: psql-another-name 

                psql-pass: 

                    file: ./psql-pass.txt  

                psql-db: 

                    external: true 

 

Todos os secrets deverão ser declarados no final do arquivo .yml 

Informar um name de um secret no parâmetro external, necessita da versão >= 3.5 

Também podemos declarar o secret informando o caminho do arquivo. 

Se o parâmetro external for igual a true, indica que o secret foi criado previamente e contém o mesmo nome do secret declarado no arquivo .yml 

Ao remover um stack, você também irá deletar os secrets vinculados a esta stack 

Os secrets utilizados no docker compose só funcionam se baseados em arquivos. 

Docker compose e secrets não funcionam com docker-machine / virtual box. A solução é utilziar volumes bind-mount. 

 

Comando para informar o nome de um arquivo compose: docker-compose –f <nome-do-arquivo.yml> –f <nome-do-arquivo-prod.yml> up –d 

Para validar um conjunto de arquivos e verificar se existem erros use: docker-compose –f <nome-do-arquivo.yml> –f <nome-do-arquivo-prod.yml> config > docker-stack.yml 

Este comando irá validar os arquivos e agrupar o mesmo em um arquivo docker-stack.yml 

 

 

HEALTHCHECK: utilizado para avaliar a saúde do container, é executado dentro do container, os resultados esperados são 0 (OK) ou 1 (ERRO) 

 

Posicionamento do container refere-se ao nó do swarm onde o container será executado 

Por padrão, o swarm distribui as tarefas de um serviço entre os nós 

Existem algumas maneiras de controlar o posicionamento dos containers: 

  1 – Utilizar os labels rótuno em nós + restrições de serviço (<key>=<value>) 

  2 – Modos de serviço (replicated | global) 

  3 – Preferências de posicionamento versão 17.04+ 

  4 – Disponibilidades de nós (active | pause | drain) 

  5 – Requisitos de recursos (CPU | memória) 

 

Restrições de serviços (Service Constraints) 

Posicionamento baseado em labels internos ou personalizados (“node.labels.” ou “engine.labels.”) 

Para posicionar serviços somente em nós do tipo manager, exemplo: 

docker service create --constraint=node.role==manager nginx 

docker service create --constraint=node.role!=work nginx 

Criar um label em um nó e usa este label em uma restrição de um serviço 

docker node update --label-add=ssd=true node3 

docker service create  --constraint=node.labels.ssd==true nginx  

 

Exemplos: 

Criar uma restrição: 

docker service create --name web1 --constraint node.role==worker nginx 

       Remover e adicionar uma nova restrição: 

docker service update --constraint-rm node.role==worker  --constraint-add node.role==manager web1 

       Criando um label em um nó e criando um serviço que irá rodar somente se houver este label 

docker node update --label-add ssd=true node2 

docker service create --name pg1 --constraint node.labels.ssd==true --replicas 2 postgres   

 

Restrições de serviços em arquivos stacks: 

    version: “3.1” 

    services: 

        mypg: 

            image: postgres:10 

            deploy: 

                placement: 

                    constraints: 

                      - node.labels.ssd == true 

 

Modo de Serviço (service mode) 

Pode ser visto no resultado do comando “docker service ls” (coluna “MODE”) 

O modo padrão é “replicated”, mas há uma opção: “global” 

Um serviço no modo “global” sempre cria uma única tarefa em cada nó 

O modo é definido somente na criação do serviço e não poderá ser alterado 

“global” é adequado para serviços de agentes (de segurança, monitoramento, backup, proxy, etc...), ele irá garantir a execução de apenas uma única instância do serviço em todos os nós do swarm 

 

Exemplos: 

Colocar uma tarefa em cada nó do swarm: 

docker service create --mode=global nginx 

Colocar uma tarefa em cada worker do swarm: 

 docker service create --mode=global --constraint=node.role==worker nginx 

 

Modo de serviços em arquivos stacks: 

    version: “3.1” 

    services: 

        web: 

            image: nginx 

            deploy: 

                mode: global 

 


Preferências de posicionamento de serviço 

“soft requirement”: se possível, suas preferências serão atendidas, mas não impedirão os containers de rodar, se o docker não conseguir atender suas preferências, ele irá colocar os serviços para rodar em outros nós 

Utiliza labels e um algoritmo para distribuir tarefas entre nós 

Atualmente, há apenas um algoritmo (estratégia): spread, que distribui uniformemente as tarefas entre os valores de um label 

Bom para distribuição em zonas de disponibilidade, datacenters, racks, subnets, etc... 

 

Exemplos: 

Adicione um label a todos os nós para identificar a zona de disponibilidade 

docker node update --label-add=zd=a node1 

docker node update --label-add=zd=b node2 

docker node update --label-add=zd=c node3  

       Distribua seus serviços por todas as zonas de disponibilidade 

docker service create --placement-pref spread=node.labels.zd --replicas 3 nginx 

       Use a atualização de serviços para adicionar ou remover preferências 

--placement-pref-add spread=node.labels.rack 

--placement-pref-rm spread=node.labels.rack 

       Usando múltiplas preferências 

docker service create --replicas 9 --name redis_2 --placement-pref ‘spread=node.labels.datacenter’ --placement-pref ‘spread=node.labels.rack’ redis:3.0.6 

 

Preferência de posicionamento em arquivos stacks: 

    version “3.1” 

    services:  

        web: 

            image:nginx 

            deploy: 

                placement: 

                    preferences: 

                       - spread: node.labels.zd 

 

 

Disponibilidade de nós do swarm 

Você pode controlar o estado de cada nó, selecionando 1 entre 3 possíveis estados 

Afeta containers novos e existentes, dependendo do estado 

Active: mantém tarefas existentes disponível para novas tarefas 

Pause: mantém tarefas existentes; indisponível para novos tarefas 

Drain: encerra as tarefas existentes, atribuindo-as a outros nós; indisponível para novas tarefas 

 

Exemplos: 

Impedindo um nó de iniciar novos containers/tarefas 

docker node update --availability pause node1 

Parando os containers em um nó e recriando-os em outros nós ativos 

docker node update --availability drain node2 

 

Requisitos de recursos 

As opções de “service create” são diferentes das opções de “docker run” 

Definidos com “service create/update”, mas referem-se a recursos de cada container criado 

Você pode reservar ou limitara apenas CPU e memória 

--limit-cpu .5 

--limit-memory 253M 

Cuidado com erros por falta de memória (OOME), se o limite for atingido o container será removido e o swarm agendará uma nova tarefa para ele. 

Definindo uma reserva de recursos para o container 

--reserve-cpu .5 

--reserve-memory 256M 

 

Exemplos: 

Reserva de memória e CPU 

docker service create --reserve-memory 800M --reserve-cpu 1 mysql 

Limitação de memória e CPU 

docker service create --limit-memory 150M --limit-cpu .25 nginx 

Atualização de serviços funciona da mesma forma 

 

Para remoção, faça uma atualização com o valor 0 

docker service update --limit-memory 0 --limit-cpu 0 myservice 

 

Requisitos de recursos em arquivos stacks 

    version “3.1” 

    services: 

        database: 

            image: mysql 

            deploy: 

                resources: 

                    limits: 

                        cpus: ‘1’ 

                        memory: 1G 

                    reservations: 

                        cpus: ‘0.5’ 

                        memory: 500M 

 

 

Testando atualizações de serviços 

Httping: similar to “ping”, mas para requisições HTTP(S) 

Browncoat: aplicação web que simula comportamentos inesperados 

Procedimento 

Crie um serviço para rodar nossa aplicação web (browncoat) mas, antes, crie uma rede overlay com a opção “--attachable”, esta opção permite que containers criados manualmente sejam adicioandos a esta rede, ela também reduz a segurança da rede overlay, já que permite que qualquer usuário adicione um container a esta rede.  

Normalmente somente managers podem permitir que container se juntem a redes overlay. 

Uma rede overlay só pode ser criada em um swarm 



 

Lidando com falhas de atualização de serviços: rollbacks 

 

Rollback é o processo de retorno do serviço à especificação imediatamente anterior 

Seus conhecimentos de atualização de serviços e healthchecks são necessários para entender rollbacks 

Duas maneiras de executar um rollback 

1 - Processo manual 

docker service rollback [options] <service>  

docker service update --rollback <service> 

2 – Processo automático de rollback durante a atualização de serviço 

Se a atualização falhar, o serviço pode recorrer a esse recurso 

Ative o rollback usando “--update-failure-action rollback” 

Ação padrão em caso de falha de atualização é “pause” 

Há uma única especificação anterior armazenada para cada serviço (“PreviousSpec”) 

Condições para execução automática do rollback 

  1 ) --update-failure-action rollback 

  2 ) --update-max-failure-ratio foi superado 

Comportamento padrão para falha de atualização: pausa 

Ative o rollback automático em ambiente de produção 

Teste: dois cenários de falha de atualização 

Cenário 1: serviços sem rollback (comportamento padrão: “pause”) 

Como sair do estado de inconsistência do serviço? 

Cenário 2: serviço com rollback (depende de configuração: --update-failure-action) 

Para utilizar rollback em arquivos de stacks, utilize a versão 3.7+ 

 

Rollbacks em arquivo stacks: 

    version: “3.7” 

    services: 

        redis: 

            image: redis 

        healthcheck: 

            test: [“CMD”, “redis-healthcheck”] 

            interval: 5s 

            timeout: 3s 

            retries: 3 

            start_period: 60s 

        deploy: 

            replicas: 1 

            update_config: 

                failure_action: rollback 

       configs: 

         - source: redis-healthcheck 

           target: /usr/local/bin/redis-healthcheck 

           mode: 0555 

    network: 

       - frontend  

    configs: 

         redis-healthcheck: 

         file: ./redis-healthcheck 

 

 

Para acompanhar a execução dos containers no linux, utilize o comando: watch docker stack ps <nome do stack> 
