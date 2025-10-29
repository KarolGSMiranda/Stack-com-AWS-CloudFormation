#  Guia de Estudo: Criando Stacks com AWS CloudFormation

Este documento é parte do meu desafio da DIO e serve como um guia consolidado sobre o processo de criação de Stacks no AWS CloudFormation. O objetivo é documentar não apenas *o que* é o processo, mas as boas práticas, os insights e as lições que aprendi.

## 1. Conceitos Fundamentais (O Alicerce)

Antes de qualquer deploy, eu precisei solidificar dois conceitos que são a base de tudo no CloudFormation:

* **Template (Modelo):** É o documento declarativo (o arquivo `.yaml` ou `.json`). Eu o vejo como a "planta baixa" da arquitetura. É o código-fonte da minha infraestrutura, onde eu *descrevo* o estado desejado dos meus recursos (EC2s, S3, VPCs, etc.).

* **Stack (Pilha):** É a *instância* do meu template. É o conjunto de recursos reais que o CloudFormation provisionou na minha conta AWS. Um único Template pode ser usado para criar múltiplas Stacks (ex: `stack-desenvolvimento` e `stack-producao`).

** Meu primeiro insight:** Tratar meus templates como código-fonte é inegociável. Eu os versiono no Git, o que me permite rastrear *toda* mudança na infraestrutura e saber exatamente o que está em produção.

---

## 2. O Processo de Criação de um Stack (Da Teoria à Prática)

Eu aprendi que "criar um Stack" não é um único clique. É um processo que dividi em 4 etapas cruciais:

### Etapa 1: Autoria do Template

É a fase de escrita do código.

* **Dica Pessoal:** Eu escolhi usar **YAML** em vez de JSON. É infinitamente mais legível para humanos, permite comentários (`#`), o que é essencial para documentar *por que* eu fiz algo, e é menos propenso a erros de sintaxe (como vírgulas faltando).
* **Estrutura Formal:** Meus templates agora sempre seguem uma estrutura clara:
    * `Parameters`: Onde eu defino as "variáveis" (ex: "Qual tipo de instância?"). Isso torna o template reutilizável.
    * `Resources`: O coração do template. A lista de recursos a serem criados.
    * `Outputs`: As "respostas" que o Stack me dará (ex: "Qual o IP da instância criada?").

### Etapa 2: Validar o Template (A Dica de Ouro)

Aqui foi onde eu mais perdi tempo no começo. O anti-padrão é fazer o upload no console e esperar dar erro. É um ciclo de feedback muito lento.

**O que me ajudou muito e talvez possa de ajudar:** Valide seu template *localmente*, antes de qualquer deploy.

1.  **Validação Rápida (Sintaxe):** Usar a AWS CLI para uma checagem rápida.
    ```bash
    aws cloudformation validate-template --template-body file://meu-template.yaml
    ```
2.  **Validação Avançada (Linter):** Esta foi a virada de chave. Eu comecei a usar o **`cfn-lint`**. Instalei-o como extensão no meu VS Code. Ele aponta erros de sintaxe, propriedades inválidas e até más práticas (como um Security Group perigoso) *enquanto eu estou digitando*. Isso me economizou horas de deploy/rollback.

### Etapa 3: Provisionar o Stack (O Deploy)

Com o template validado, partimos para o provisionamento.

1.  **Via Console (Para Aprender):** É ótimo para começar. O processo é visual: Upload do template, preenchimento dos `Parameters` que eu mesmo criei, e confirmação.
2.  **Via CLI (Para Automatizar):** Este é o método profissional, essencial para automação e pipelines de CI/CD.
    ```bash
    aws cloudformation create-stack \
      --stack-name minha-stack-de-teste \
      --template-body file://meu-template.yaml \
      --parameters ParameterKey=TipoInstancia,ParameterValue=t2.micro
    ```

### Etapa 4: Monitorar a Criação

Após o deploy, o Stack entra em `CREATE_IN_PROGRESS`.

* **Minha experiência:** Minha primeira reação foi ficar dando F5 no console. A forma correta é acompanhar a aba **"Events" (Eventos)**. Ela é o log de auditoria em tempo real. Eu consigo ver o CloudFormation criando cada recurso, um por um, na ordem correta.
* **Sucesso:** O status final é `CREATE_COMPLETE`.
* **Falha:** O status muda para `CREATE_FAILED`.

---

## 3. Insights Essenciais (A Experiência Prática)

Aqui estão as lições mais valiosas que aprendi ao lidar com Stacks:

### O que é o `ROLLBACK`? (E por que ele é meu amigo)

* **Conceito Formal:** O CloudFormation trata a criação do Stack como uma transação **atômica**. Ou ele cria *tudo* com sucesso, ou ele (na maioria dos casos) reverte e não cria *nada*.
* **Minha Experiência:** Quando meu Stack falhou pela primeira vez (errei um nome de propriedade), eu entrei em pânico. Mas aí vi a mágica: o Stack entrou em `ROLLBACK_IN_PROGRESS` e o CloudFormation **destruiu automaticamente tudo** o que ele tinha acabado de criar.
* **Insight:** Eu percebi que isso não é um "erro", é uma *feature* de segurança. Garante que minha conta não fique suja de "recursos órfãos" ou em estado inconsistente. A lição foi: quando falhar, não delete o Stack imediatamente. Leia o erro na aba "Events" para saber *o porquê* da falha.

### Atualizando Stacks: O Risco do "Drift" e o poder dos "Change Sets"

* **O Anti-Padrão (Drift):** O maior erro é criar um Stack e depois ir no console do EC2 e alterar um recurso manualmente. Isso cria um **"Drift" (desvio)**: a sua infraestrutura real não é mais igual ao que está no seu template (seu "código-fonte").
* **A Boa Prática (Change Sets):** Eu não clico mais em "Update Stack" direto. Eu aprendi a usar **Change Sets (Conjuntos de Alterações)**.
    1.  Eu subo o meu template modificado e peço para o CloudFormation "Criar um Change Set".
    2.  Ele me dá um **"preview"**, um relatório detalhado do que ele *vai* fazer: `[Modify]` (Vai alterar isto), `[Add]` (Vai adicionar isto), `[Remove]` (Cuidado! Vai deletar isto).
    3.  Eu analiso esse "plano". Se estiver correto, eu o "Executo". Isso é fundamental para evitar desastres em produção.

### Forçando a Ordem: `DependsOn`

* **Dependência Implícita:** O CloudFormation é inteligente. Se minha Instância EC2 (Recurso A) usa `!Ref` para se associar a um Security Group (Recurso B), ele sabe que precisa criar o B *antes* do A.
* **Dependência Explícita:** Contudo, às vezes um recurso depende de outro sem uma referência direta. Nesses casos, eu aprendi a usar a cláusula `DependsOn` para *forçar* a ordem. Isso foi crucial para garantir que uma política de acesso estivesse pronta antes que a aplicação tentasse usá-la.
