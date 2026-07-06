# Telegram Weather Chatbot

Este projeto é um chatbot para Telegram construído no **n8n**. Ele permite que os usuários consultem a temperatura atual de qualquer cidade de forma rápida e intuitiva. 

O fluxo integra a **OpenWeather API** para buscar os dados meteorológicos (temperatura, sensação térmica, umidade e condição do tempo) e utiliza o **Google Gemini** para formatar a resposta com linguagem natural e amigável. Caso a IA falhe ou não possua credenciais configuradas, o sistema possui uma rota de **fallback determinístico**, garantindo que o usuário sempre receba a sua resposta.

## Tecnologias e Pré-requisitos

* **n8n** (Instância local via Docker)
* **Telegram Bot Token** (Criado via BotFather no Telegram)
* **OpenWeather API Key** (Conta gratuita na OpenWeatherMap)
* **Google Gemini API Key** (Opcional, para melhoria da mensagem de saída)

## Preparação do ambiente local (Docker)

Se você deseja rodar o n8n localmente, este repositório inclui um arquivo `docker-compose.yml`.

1. Clone o repositório.
2. Renomeie o arquivo `.env.example` para `.env`.
3. Edite o arquivo `.env` e insira a sua chave real da OpenWeather na variável esperada:
   
   OPENWEATHER_API_KEY=sua_chave_real_aqui
   N8N_ENV_VARS_ALLOW_LIST=OPENWEATHER_API_KEY

4. No terminal, execute o comando para iniciar o n8n:

   ```bash
   docker-compose up -d
   ```

5. Acesse o n8n no seu navegador: `http://localhost:5678`

## Passos para importar o workflow no n8n

1. Abra a sua interface do n8n.
2. No menu lateral esquerdo, clique em **Workflows** e depois no botão **Add workflow**.
3. No canto superior direito, clique no menu de três pontos (opções) e selecione **Import from File**.
4. Selecione o arquivo `workflow-chatbot-telegram.json` incluído neste repositório.
5. O fluxo será carregado na sua tela.

## Configuração das Credenciais no n8n

Para que o workflow funcione, você precisa configurar as credenciais nos nós específicos.

### 1. Telegram (TELEGRAM_BOT_TOKEN)
* No workflow, clique no nó inicial **Telegram Trigger**.
* Em *Credential to connect with*, clique em *Create New Credential*.
* Insira o seu `TELEGRAM_BOT_TOKEN` gerado pelo BotFather e salve.
* **Importante:** Repita a seleção dessa credencial nos nós finais de envio de mensagem (**Telegram: Erro**, **Telegram: Erro1**, **Telegram: Erro Rede** e **Clima atual**).

### 2. OpenWeather (OPENWEATHER_API_KEY)
* Este workflow foi projetado com boas práticas de segurança. A chave da OpenWeather **não** é inserida diretamente na interface do nó.
* Ela é lida a partir das variáveis de ambiente do sistema operacional ou do Docker (arquivo `.env`).
* No nó **OpenWeather API**, a variável interna `queue` (formatada pelo workflow) é enviada ao parâmetro `q` exigido pela API da OpenWeather.
* O nó possui timeout de 10 segundos para evitar esperas indefinidas em caso de lentidão ou indisponibilidade da API.

### 3. Google Gemini (Opcional)
* O workflow possui um nó do **Google Gemini** (LangChain) localizado logo após o nó "Validação de Resposta", na rota de sucesso.
* Para ativá-lo, clique no nó **Message a model**.
* Crie ou selecione uma credencial do tipo *Google Gemini(PaLM) Api account* e insira a sua API Key do Google AI Studio.

### Configuração de Webhook

O Telegram exige que o bot receba atualizações através de uma URL pública (HTTPS). Se você estiver rodando o n8n localmente (localhost), precisará expor a porta `5678` para a internet usando uma ferramenta como o [ngrok](https://ngrok.com/) ou Cloudflare Tunnels.

**Como configurar com o ngrok:**
1. Em um novo terminal, inicie o ngrok apontando para a porta do n8n:

   ```bash
   ngrok http 5678
   ```
2. Copie a URL `Forwarding` gerada pelo ngrok (ex: `https://abcd-12-34.ngrok-free.dev`).
3. Abra o arquivo `docker-compose.yml` e substitua o valor genérico da variável `WEBHOOK_URL` pela sua URL do ngrok gerada no passo anterior:

   ```yaml
   WEBHOOK_URL=[https://abcd-12-34.ngrok-free.dev](https://abcd-12-34.ngrok-free.dev)
   ```
4. Após atualizar a URL, suba o container novamente com `docker-compose up -d`.

## Como executar e testar o Chatbot

Com o workflow ativo (botão superior direito `Active` habilitado) e as credenciais configuradas, abra o Telegram e inicie uma conversa com o seu bot.

**Cenário de Sucesso:**

1. Envie uma mensagem no formato `Cidade,UF,BR`. Exemplo:
   > `São Paulo,SP,BR`
   
2. O bot processará o pedido e retornará (exemplo com fallback determinístico):
   > `🌤️ A temperatura em São Paulo (SP) é de 25°C. Céu limpo. Sensação térmica: 26°C. Umidade: 65%.`

**Cenários de Erro (Tratamento de Exceções):**
1. **Formato inválido:** Envie apenas `São Paulo` (sem a vírgula e o estado).
   * O bot responderá: `⚠️ Formato inválido. Por favor, envie a mensagem no padrão: Cidade,UF,BR (Exemplo: São Paulo,SP,BR).`
   
2. **Cidade inexistente:** Envie `CidadeInventadaXYZ,SP,BR`.
   * O bot responderá: `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).`

3. **Falha de rede ou timeout:** Se a OpenWeather estiver indisponível ou a consulta exceder 10 segundos.
   * O bot responderá: `⚠️ Não foi possível consultar o clima no momento. Verifique sua conexão ou tente novamente.` (ou mensagem específica de timeout).

## Troubleshooting (Problemas Comuns)

### O bot não responde no Telegram

* Confirme que o workflow está **ativo** (toggle `Active` no canto superior direito).
* Verifique se a credencial `TELEGRAM_BOT_TOKEN` está configurada em **todos** os nós Telegram (Trigger, Clima atual, Telegram: Erro, Telegram: Erro1 e Telegram: Erro Rede).
* Se estiver rodando localmente, confirme que o `WEBHOOK_URL` no `docker-compose.yml` aponta para uma URL pública HTTPS válida (ngrok ou similar) e reinicie o container após alterar.
* No ngrok gratuito, a URL muda a cada reinício — atualize o `WEBHOOK_URL` e execute `docker-compose up -d` novamente.

### Erro ao consultar a OpenWeather

* Verifique se `OPENWEATHER_API_KEY` está definida no arquivo `.env` e se `N8N_ENV_VARS_ALLOW_LIST=OPENWEATHER_API_KEY` está presente.
* Confirme que `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` está no `docker-compose.yml` (necessário para o n8n ler variáveis de ambiente nos nós).
* Chaves recém-criadas na OpenWeather podem levar alguns minutos para ativar — aguarde e tente novamente.
* Teste o formato da cidade: `Belo Horizonte,MG,BR` (cidade, UF e país separados por vírgula).

### Mensagem de timeout ou falha de conexão

* O nó **OpenWeather API** possui timeout de 10 segundos. Se a API estiver lenta, o fluxo encaminha para **Telegram: Erro Rede** com mensagem amigável.
* Verifique sua conexão com a internet e o status da [OpenWeather API](https://openweathermap.org/).

### O Gemini não responde / usa sempre o fallback

* Isso é esperado se as credenciais do Google Gemini não estiverem configuradas — o nó **Fallback** garante resposta determinística.
* Para ativar a IA, configure a credencial no nó **Message a model** conforme a seção acima.
* Se a IA falhar em runtime, o fluxo automaticamente usa o **Fallback** com os mesmos dados meteorológicos.

### Variável de ambiente não reconhecida no nó HTTP

* Reinicie o container após editar o `.env`: `docker-compose down && docker-compose up -d`.
* No nó **OpenWeather API**, o parâmetro `appid` deve usar a expressão `={{ $env.OPENWEATHER_API_KEY }}` (sem valor hardcoded).
