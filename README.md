# aws-bedrock-ai-assistant-with-nestjs
Exemplo para implementação de assistente de inteligência artificial generativa, usando AWS Bedrock com NestJS

>> Finalidade:
- Para ser implementado em projetos de chat onde é necessário manter um histórico de conversas que seja utilizado como contexto.

>> Funcionalidades:
- É feito um mapeamento de conversas na tabela de registros de conversas do usuário;
- As informações são estruturadas para que possa ser processada pela Inteligência Artificial;
- A última mensagem enviada pelo usuáiro é acerescentada ao "Array" de conversas estruturadas;
- As informações estruturadas são enviadas para o modelo de IA utilizado (No exemplo do código o modelo utilizado é o Claude3 Sonete);
- O modelo recebe e processa as informações, senguindo instruções do prompt e utilizando o contexto do histórico da conversa, obtidos no início do escopo do código;
- O modelo devolve a resposta e a mesma é estruturada para registro em banco;
- A estrutura da resposta é devolvida ao usuário.

Para o exemplo, é utilizada a dependência da biblioteca do Bedrock da aws: ``@aws-sdk/client-bedrock-runtime``
O "AssistantConversationValidator" é um DTO da estrutura dos dados recebidos.
A função é criada no NestJS dentro de uma classe.
O ORM utilizado para o banco de dados é o Prisma.

```
$typescript

async assistant(
    assistantConversationValidator: AssistantConversationValidator,
  ) {
    const { accountId, content, role } = assistantConversationValidator
    const conversations = await this.prisma.conversation
      .findMany({
        where: { accountId: accountId },
        take: 50,
        orderBy: { createdAt: 'desc' },
      })
      .then((response) =>
        response.map((conversation) => {
          const message = {
            role: conversation.role,
            content: [{ type: 'text', text: conversation.content }],
          }
          return message
        }),
      )

    const message = {
      role: role,
      content: [{ type: 'text', text: content }],
    }

    conversations.push(message)

    const command: InvokeModelCommandInput = {
      modelId: 'anthropic.claude-3-sonnet-20240229-v1:0',
      contentType: 'application/json',
      accept: 'application/json',
      body: JSON.stringify({
        system: ASSISTANT_INSTRUCTIONS_PROMPT,
        anthropic_version: 'bedrock-2023-05-31',
        max_tokens: 1000,
        messages: conversations,
      }),
    }

    const invoke = new InvokeModelCommand(command)
    try {
      const data = await this.awsService.bedrockRuntimeClient.send(invoke)
      const dataObject = new TextDecoder().decode(data.body)
      const response = JSON.parse(dataObject)

      const assistant: AssistantConversationValidator = {
        accountId: accountId,
        role: response?.role,
        content: response?.content[0]?.text,
      }

      if (response)
        return await this.prisma.conversation.create({ data: { ...assistant } })

      return
    } catch (error) {
      throw new HttpException(error?.message, error?.code)
    } finally {
    }
  }
```
