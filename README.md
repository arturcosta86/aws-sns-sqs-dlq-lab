# Laboratório: Tópico SNS e Filas SQS com Dead-Letter Queue (DLQ)

Este repositório documenta a implementação de um fluxo de mensagens resiliente na AWS utilizando SNS, SQS e uma Dead-Letter Queue (DLQ). O projeto foi realizado como parte dos estudos para a certificação **AWS Certified Developer - Associate** na Escola da Nuvem.

O objetivo é demonstrar uma arquitetura de microsserviços desacoplada, onde eventos são processados de forma assíncrona com um mecanismo robusto para tratamento de falhas.

**Instrutor:** Tomas Alric ([@TomasAlric](https://github.com/TomasAlric/TomasAlric))
**Aluno:** Artur Costa ([@arturcosta86](https://github.com/arturcosta86))

---

## 🎯 Objetivos do Laboratório

* Criar e configurar uma fila SQS para atuar como Dead-Letter Queue (DLQ).
* Criar uma fila SQS Padrão (principal) e configurá-la para usar a DLQ através de uma Política de Redirecionamento (Redrive Policy).
* Criar um tópico SNS Padrão para publicar eventos.
* Inscrever a fila SQS principal no tópico SNS para receber as mensagens.
* Testar o fluxo de mensagens e simular uma falha de processamento para observar a mensagem sendo enviada para a DLQ.

---

## 🛠️ Arquitetura e Fluxo de Mensagens

A solução implementa um padrão de arquitetura comum para sistemas orientados a eventos, garantindo desacoplamento e tolerância a falhas.

O diagrama abaixo ilustra o fluxo:

```
+-------------------+      +-----------------+      +-----------------------+
|  Publicador de    |----->|   Tópico SNS    |----->|  Fila SQS Principal   |
|    Mensagens      |      | (meu-topico-lab)|      | (minha-fila-principal)|
+-------------------+      +-----------------+      +-----------+-----------+
                                                                |
                                                                | 1. Consumidor tenta
                                                                |    processar
                                                                V
                                                        +-------+-------+
                                                        |   Falha no      |
                                                        | Processamento?  |
                                                        +-------+-------+
                                                                | (Sim, após 3 tentativas)
                                                                | Redrive Policy
                                                                V
                                                        +-----------------------+
                                                        | Fila Dead-Letter (DLQ)|
                                                        |   (minha-dlq-lab)     |
                                                        +-----------------------+
```

**Fluxo Detalhado:**

1.  Um evento é publicado em um **Tópico SNS**.
2.  O SNS envia uma cópia da mensagem para a **Fila SQS Principal** que está inscrita no tópico.
3.  Um consumidor (não simulado no lab) tenta processar a mensagem da Fila Principal.
4.  Se o consumidor falhar em processar e excluir a mensagem repetidamente (3 vezes, conforme configurado na Redrive Policy), a política é acionada.
5.  A mensagem é movida para a **Fila de Mensagens Mortas (DLQ)** para análise posterior, evitando o bloqueio do processamento de outras mensagens.

---

## 📄 Política de Acesso da Fila SQS (Código)

Para que o Tópico SNS possa enviar mensagens para a Fila SQS, uma política de acesso baseada em recursos é necessária na fila principal. Abaixo está o modelo da política em JSON utilizada.

```json
{
  "Version": "2012-10-17",
  "Id": "__default_policy_ID",
  "Statement": [
    {
      "Sid": "__owner_statement",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<COLE AQUI SEU ID CONTA DA AWS (12 DÍGITOS)>:root"
      },
      "Action": "SQS:*",
      "Resource": "<COLE AQUI O ARN DA SUA FILA PRINCIPAL>"
    },
    {
      "Sid": "Allow-SNS-SendMessage",
      "Effect": "Allow",
      "Principal": {
        "Service": "sns.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "<COLE AQUI O ARN DA SUA FILA PRINCIPAL>",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "< COLE AQUI O ARN DO SEU TÓPICO SNS >"
        }
      }
    }
  ]
}
```

---

## ✅ Evidências da Implementação

As capturas de tela a seguir comprovam a configuração e o funcionamento da arquitetura.

### 1. Criação da Fila de Mensagens Mortas (DLQ)
* **Descrição:** Evidência da criação da fila `minha-dlq-lab-arturcosta`, que servirá como repositório para as mensagens que falharam no processamento.

![Criação da DLQ](https://github.com/arturcosta86/aws-sns-sqs-dlq-lab/blob/main/Print%20das%20Filas%20SQS%20(DLQ%20-%20criada%20-%20%20Artur%20Costa.jpeg)

### 2. Configuração da Fila Principal com a Política de Redirecionamento
* **Descrição:** A fila `minha-fila-principal-lab-arturcosta` configurada com a Política de Redirecionamento (Redrive Policy). Note que ela aponta para a DLQ e o "Número máximo de recebimentos" está definido como **3**.

![Configuração da Fila Principal]([Print%20das%20Filas%20SQS%20(principal)%20-%20Artur%20Costa.jpeg](https://github.com/arturcosta86/aws-sns-sqs-dlq-lab/blob/main/Print%20das%20Filas%20SQS%20(principal)%20-%20Artur%20Costa.jpeg))

### 3. Inscrição da Fila SQS no Tópico SNS
* **Descrição:** Detalhes da assinatura (subscription) no tópico SNS, mostrando que a fila SQS principal está inscrita como um endpoint, pronta para receber mensagens.

![Inscrição SQS no SNS]([Print%20do%20Tópico%20SNS%20-%20Artur%20Costa.jpeg](https://github.com/arturcosta86/aws-sns-sqs-dlq-lab/blob/main/Print%20do%20T%C3%B3pico%20SNS%20-%20Artur%20Costa.jpeg))

### 4. Resultado Final: Mensagem na DLQ
* **Descrição:** Prova final do funcionamento da arquitetura. Após simular falhas de processamento na fila principal, a mensagem foi movida com sucesso para a DLQ, onde pode ser inspecionada. O "Número de recebimentos" (Receive Count) maior que 3 confirma que a política de redirecionamento foi acionada.

![Mensagem na DLQ](https://github.com/arturcosta86/aws-sns-sqs-dlq-lab/blob/main/Print%20das%20Filas%20SQS%20(DLQ)%20-%20com%20a%20mensagem%20n%C3%A3o%20processada%20-%20%20Artur%20Costa.jpeg)
