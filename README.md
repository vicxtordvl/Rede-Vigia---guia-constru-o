# Rede Vigia

MVP de monitoramento de nível de água com **Arduino UNO, sensor ultrassônico e SIM800L**, API em **Node.js/Express**, persistência em **MongoDB** e painel em **Next.js/React**.

Este guia descreve como preparar uma máquina a partir do clone do Git, montar e configurar o protótipo, iniciar os serviços, verificar medições reais e encerrar a demonstração. Os comandos principais usam **Windows com PowerShell**, ambiente utilizado no desenvolvimento. Há notas para Linux e macOS ao final.

As faixas Normal, Atenção e Risco são experimentais do MVP. A calibração deve corresponder ao recipiente utilizado; o protótipo não constitui um sistema certificado de alerta de enchentes.

## Índice

- [1. Funcionamento e estrutura](#visao-geral)
- [2. Requisitos de software, contas e rede](#requisitos)
- [3. Componentes e montagem física](#montagem)
- [4. Clonar e instalar as dependências](#instalacao)
- [5. Preparar o MongoDB](#mongodb)
- [6. Configurar backend e frontend](#ambiente)
- [7. Instalar e configurar o ngrok do zero](#ngrok)
- [8. Configurar, calibrar e gravar o Arduino](#arduino)
- [9. Iniciar o MVP](#iniciar)
- [10. Validar a demonstração e usar o painel](#validar)
- [11. Contrato HTTP e endpoints](#api)
- [12. Diagnóstico de problemas](#diagnostico)
- [13. Encerrar o MVP](#encerrar)
- [14. Reiniciar em outro dia](#reiniciar)
- [15. Build e verificações opcionais](#verificacoes)
- [16. Teste sem protótipo e seed](#simulacao)
- [17. Linux, macOS e documentação complementar](#referencias)

<a id="visao-geral"></a>

## 1. Funcionamento e estrutura

```mermaid
flowchart LR
    Sensor["Sensor ultrassônico"] -->|TRIG / ECHO| UNO["Arduino UNO"]
    UNO -->|Comandos AT pela UART| Modem["SIM800L + chip 2G"]
    Modem -->|POST HTTP pela rede móvel| Publico["Endpoint público ngrok"]
    Publico --> Agente["Agente ngrok no computador"]
    Agente --> API["Backend Express :3000"]
    API --> Banco[("MongoDB local ou Atlas")]
    Navegador["Navegador"] --> Front["Frontend Next.js :5173"]
    Front -->|Consultas GET via proxy| API
```

O Arduino mede a distância até a superfície da água, calcula o percentual e envia uma medição. O backend valida os campos, registra a data de recebimento e grava no MongoDB. O painel consulta a API e apresenta a última leitura, gráfico, estatísticas e histórico.

O firmware atual utiliza **um único arquivo `.ino`**, com configuração no próprio arquivo. Não utiliza `config.h` nem ponte USB para enviar as medições. O USB é usado para carregar o sketch, alimentar o UNO quando apropriado e acompanhar o Monitor Serial.

Mesmo com o Arduino fora do USB do computador, **o computador que hospeda o backend e o ngrok precisa continuar ligado, conectado à internet e sem suspensão**. O acesso à internet do Arduino vem do chip do SIM800L. O frontend é necessário para visualizar o painel; o recebimento de medições depende do backend, do banco e do túnel.

```text
redevigia/
├── README.md                         # Este guia geral
├── back/
│   ├── src/                          # API, validação, banco e seed
│   ├── .env.example                  # Modelo de configuração
│   ├── package.json
│   ├── package-lock.json
│   ├── swagger.yaml                  # Contrato OpenAPI
│   ├── redevigia.json                # Coleção Postman
│   ├── TUNNEL.md                     # Orientações específicas do ngrok
│   └── readme.md
├── frontend/
│   ├── src/                          # Painel web
│   ├── tests/                        # Testes de contrato e interface
│   ├── .env.example
│   ├── next.config.ts                # Proxy das consultas para a API
│   ├── package.json
│   ├── package-lock.json
│   └── README.md
└── firmware/
    ├── README.md                      # Montagem e firmware
    └── rede_vigia/
        └── rede_vigia.ino             # Abrir este arquivo na IDE Arduino
```

As pastas reais são **`back`**, **`frontend`** e **`firmware`**. Não existe `package.json` na raiz: cada aplicação web instala e executa suas próprias dependências.

Os arquivos `back/.env`, `frontend/.env.local`, as dependências, os builds e a pasta `back/.tools` são locais e ignorados pelo Git. Portanto, **um clone novo não inclui o executável ngrok nem a configuração autenticada da máquina de desenvolvimento**.

<a id="requisitos"></a>

## 2. Requisitos de software, contas e rede

| Requisito | Uso e orientação |
| --- | --- |
| Git | Clonar o repositório. Instalação: [Git](https://git-scm.com/downloads). |
| Node.js e npm | Utilize **Node.js 24 LTS**. O ambiente utilizado tem Node 24 e npm 11. O frontend declara Node >=20.9, mas o Mongoose instalado requer >=20.19; não se baseie apenas no mínimo do frontend. [Versões Node.js](https://nodejs.org/en/about/previous-releases). |
| MongoDB | Escolha MongoDB Atlas ou MongoDB Community local. Não é necessário usar os dois. |
| Conta ngrok e agente v3 | Criar os endpoints públicos HTTP e HTTPS para o backend. Cada pessoa deve usar o token e domínio de sua própria conta. |
| Arduino IDE 2.x | Compilar e carregar o sketch. Instale o pacote **Arduino AVR Boards** pelo Gerenciador de Placas. [Download Arduino IDE](https://www.arduino.cc/en/software/). |
| Navegador | Acessar o painel. Google Chrome instalado também é necessário para os testes E2E na configuração atual. |
| Cabo USB de dados | Conectar e programar o UNO. Cabo apenas de carga não permite upload. |
| Chip de celular com dados | Precisa estar ativo, com PIN desabilitado e acesso GSM/GPRS **2G** na região do teste. |
| Acesso à internet | Necessário para instalar dependências, conectar o ngrok e, se utilizado, acessar o MongoDB Atlas. |

O SIM800L é um modem 2G. O fato de um chip funcionar em um celular 4G/5G não comprova que haverá serviço 2G disponível para o protótipo. Confirme cobertura, APN e habilitação de dados com a operadora antes da montagem.

### Portas utilizadas

| Porta/endereço | Função |
| --- | --- |
| `http://localhost:5173` | Painel Next.js. |
| `http://localhost:3000` | Backend Express. |
| `http://127.0.0.1:4040` | Inspeção/API local do agente ngrok, na configuração padrão. |
| `localhost:27017` | MongoDB, somente na opção de instalação local padrão. |
| `http://SEU_DOMINIO_NGROK` | Endpoint público para o SIM800L, porta HTTP 80. |
| `https://SEU_DOMINIO_NGROK` | Endpoint público para testes no navegador, porta HTTPS 443. |

O túnel encaminha a **porta 3000**, não a porta do MongoDB nem a porta do frontend. O Arduino não pode usar `localhost` para alcançar o computador.

<a id="montagem"></a>

## 3. Componentes e montagem física

### 3.1 Lista de componentes

| Quantidade | Componente | Observação |
| --- | --- | --- |
| 1 | Arduino UNO R3 ou compatível com ATmega328P | Alvo do firmware atual. Outra família de placa exige revisar compatibilidade elétrica e bibliotecas. |
| 1 | AJ-SR04M ou JSN-SR04T | Variante configurada para **TRIG/ECHO**, com placa controladora e transdutor. Confirme pinout, alimentação, modo e zona cega da variante. |
| 1 | SIM800L | Confira o modelo da placa adaptadora e se já possui regulador/conversão de nível. |
| 1 | Antena compatível com o SIM800L | Conectar antes de ligar o modem. |
| 1 | Chip SIM compatível com o suporte do módulo | Plano com dados e cobertura 2G. |
| 1 conjunto | Fonte DC e alimentação regulada do modem | Saída próxima de 4,0 V e capacidade para picos de 2 A no SIM800L. |
| Conforme montagem | Regulador step-down, como LM2596 | Necessário quando a fonte não fornece diretamente a tensão do modem. Conferir capacidade real, queda nos cabos e dissipação. |
| Conforme montagem | Adaptação de nível lógico para UART | Compatibilizar os sinais do UNO de 5 V com os do SIM800L. |
| 1 | Cabo USB de dados para o UNO | Para upload e diagnóstico. |
| Conforme montagem | Fios, conectores e suporte firme | Usar conexões curtas e adequadas à corrente na alimentação do modem. |
| 1 | Multímetro | Ajustar e verificar as tensões antes de conectar o módulo. |
| 1 | Recipiente de teste e régua/trena | Calibração e demonstração do nível. |

### 3.2 Alimentação

Monte e altere os fios com as fontes desligadas. Utilize fontes/adaptadores DC prontos; o protótipo não se conecta diretamente à rede elétrica da tomada.

**UNO:** pode ser alimentado pelo USB com 5 V ou pelo conector de alimentação/VIN conforme a placa; no UNO clássico a faixa recomendada de entrada externa é 7–12 V. Não aplique essa faixa no pino 5 V. Para a primeira validação, use o USB do computador. [Orientação de alimentação Arduino](https://support.arduino.cc/hc/en-us/articles/360018922259-What-power-supply-can-I-use-with-my-Arduino-board).

**SIM800L:** no módulo sem regulador de entrada, a alimentação nominal é aproximadamente **4,0 V**, dentro de 3,4–4,4 V, e deve suportar picos de **2 A**. Não ligue seu VCC/VBAT diretamente aos 5 V do UNO ou a uma saída digital. Uma placa adaptadora pode ter requisitos diferentes: confira sua documentação. [Manual elétrico SIM800L](https://simcom.ee/documents/SIM800L/SIM800L%28MT6261%29_Hardware%20Design_V1.01.pdf).

Se usar LM2596, ajuste a saída com o SIM800L desconectado e confirme a polaridade com o multímetro. A fonte de entrada e o regulador devem sustentar a carga. Medir 4 V sem carga não comprova que a tensão se mantém nos picos de transmissão.

O sensor deve receber a tensão indicada para sua variante; a montagem de referência utiliza o sensor em 5 V. A placa controladora, UNO e modem precisam compartilhar a referência de GND. Mantenha placas, conexões e fontes secas; somente o elemento apropriado do sensor deve ficar exposto ao ambiente previsto pelo fabricante.

### 3.3 Ligações do firmware atual

| Origem | Destino | Função |
| --- | --- | --- |
| Sensor TRIG | UNO D4 | Pulso de disparo. |
| Sensor ECHO | UNO D5 | Medição do tempo do eco. |
| Sensor VCC | Alimentação conforme a variante, 5 V na referência | Alimentação do sensor. |
| Sensor GND | GND comum | Referência do sensor. |
| SIM800L TXD | UNO D7, com compatibilidade de nível verificada | D7 é **RX do Arduino**. |
| UNO D8 | SIM800L RXD, por adaptação de nível | D8 é **TX do Arduino**. |
| SIM800L VCC/VBAT | Saída regulada adequada ao módulo | Alimentação dedicada. |
| UNO GND, sensor GND, SIM800L GND e saída negativa da fonte/regulador | GND comum | Referência compartilhada mesmo sem USB no computador. |

Não confunda os nomes RX/TX: o **TX de um dispositivo chega ao RX do outro**. Sketches antigos de diagnóstico podem usar D2/D3; o arquivo atual utiliza **D7/D8**.

### 3.4 D7, D8 e adaptação de nível

O D8 fica alto em repouso porque é uma saída de comunicação serial. Isso é normal. O firmware não reduz a tensão desse pino: a adaptação para o RX do SIM800L é feita no circuito.

O arranjo de bancada discutido para **D8 → RXD** usa dois resistores de 10 kΩ:

```text
UNO D8 (TX) ---- 10 kΩ ----+---- RXD do SIM800L
                         |
                        10 kΩ
                         |
                       GND comum
```

Com 5 V na entrada, esse divisor resulta nominalmente em 2,5 V. Ele é um divisor **de sinal**, não de alimentação. Confira a tensão no nó RXD e se a placa do modem já contém outro circuito de adaptação. Uma alternativa é usar conversor de nível adequado a UART nos níveis envolvidos.

No **D7**, o `setup()` já desliga o pull-up que a `SoftwareSerial` habilita. O pino continua sendo entrada; solto, sua tensão pode flutuar. Não adicione divisor para reduzir ainda mais o sinal do TXD do modem.

Há uma diferença entre funcionar em bancada e garantir níveis lógicos: o TXD do SIM800L é tipicamente 2,8 V, enquanto o ATmega328P a 5 V especifica nível alto de entrada a partir de 0,6 × VCC, ou 3,0 V. Para uma montagem com margem elétrica garantida, use adaptação apropriada **também no sentido TXD → D7**, elevando o sinal para o nível aceito pelo UNO. Desligar o pull-up não faz essa conversão. [Características do ATmega328P](https://ww1.microchip.com/downloads/aemDocuments/documents/MCU08/ProductDocuments/DataSheets/Atmel-7810-Automotive-Microcontrollers-ATmega328P_Datasheet.pdf).

### 3.5 Posição do sensor e preparação do chip

Fixe o sensor acima da água, apontando para a superfície e evitando bordas do recipiente no caminho do feixe. Use um alvo estável durante a primeira leitura. O código aceita amostras entre 20 e 500 cm, mas a distância mínima física depende do sensor utilizado.

Com a alimentação desligada, instale o SIM e a antena. No celular, confirme a ativação do chip e desabilite a solicitação de PIN do SIM. Confira APN, usuário e senha da operadora. Algumas placas SIM800L precisam de acionamento de PWRKEY; o firmware não controla esse pino, então o modem precisa estar efetivamente ligado para responder a `AT`.

<a id="instalacao"></a>

## 4. Clonar e instalar as dependências

No PowerShell, confira as ferramentas:

```powershell
git --version
node --version
npm.cmd --version
```

Clone na pasta em que deseja trabalhar:

```powershell
git clone https://github.com/douglvsz/redevigia.git
cd redevigia
```

**A partir daqui, os comandos PowerShell são executados na raiz `redevigia`, salvo indicação explícita.** Em cada novo terminal, navegue até essa pasta. Não copie os caminhos pessoais da máquina de outra pessoa.

Instale as duas aplicações usando os lockfiles versionados:

```powershell
npm.cmd --prefix .\back ci
npm.cmd --prefix .\frontend ci
```

`npm ci` instala as versões registradas no `package-lock.json`. Se falhar, verifique a versão do Node, acesso ao registro npm e a mensagem de erro; não apague os lockfiles como primeira tentativa. A instalação inicial exige acesso à internet.

Crie os arquivos locais sem sobrescrever configurações já existentes:

```powershell
if (-not (Test-Path .\back\.env)) {
    Copy-Item .\back\.env.example .\back\.env
}
if (-not (Test-Path .\frontend\.env.local)) {
    Copy-Item .\frontend\.env.example .\frontend\.env.local
}
```

No Windows, `npm.cmd` evita depender da execução de `npm.ps1` quando a política do PowerShell bloqueia scripts. Não é necessário alterar essa política para executar os comandos deste guia.

<a id="mongodb"></a>

## 5. Preparar o MongoDB

Escolha **uma** das alternativas abaixo. A API só começa a escutar na porta 3000 depois de conectar ao banco. Não é necessário executar seed, criar tabelas ou importar a coleção Postman para inicializá-la; a persistência é feita pelos modelos Mongoose.

### 5.1 MongoDB Atlas

1. Crie sua conta, projeto e um cluster adequado ao teste no [MongoDB Atlas](https://www.mongodb.com/atlas).
2. Crie um **usuário de banco**, com acesso de leitura e escrita ao banco `redevigia`. Esse usuário é diferente do login do painel Atlas.
3. Cadastre na lista de acesso de rede o IP público de saída do computador que executará o backend.
4. Na opção de conexão para aplicações/drivers, copie a URI fornecida pelo cluster.
5. Preencha usuário, senha e o nome do banco `redevigia` na URI e coloque-a em `back/.env`.

Exemplo ilustrativo, a substituir pela URI do seu cluster:

```dotenv
MONGODB_URI=mongodb+srv://USUARIO:SENHA_CODIFICADA@SEU_CLUSTER.mongodb.net/redevigia?retryWrites=true&w=majority
```

Caracteres especiais nas credenciais precisam estar codificados corretamente para uma URI. Preserve os parâmetros exigidos pela conexão fornecida pelo Atlas. Se mudar de rede, confira novamente o IP autorizado.

O acesso ao MongoDB parte do **backend**, portanto não é o IP do Arduino nem o domínio ngrok que precisa ser cadastrado nessa lista. Consulte [usuários de banco](https://www.mongodb.com/docs/atlas/security-add-mongodb-users/) e [lista de IPs](https://www.mongodb.com/docs/atlas/security/add-ip-address-to-list/).

### 5.2 MongoDB Community local no Windows

Instale o MongoDB Community seguindo o [guia oficial para Windows](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-windows/). A opção de serviço do Windows facilita a inicialização. O MongoDB Compass é uma interface opcional; instalar apenas o Compass não substitui o servidor MongoDB.

Se o serviço foi instalado com o nome padrão `MongoDB`, consulte seu estado:

```powershell
Get-Service -Name MongoDB
```

Se estiver parado, em um PowerShell com permissão para gerenciar serviços:

```powershell
Start-Service -Name MongoDB
```

Configure:

```dotenv
MONGODB_URI=mongodb://127.0.0.1:27017/redevigia
```

Se instalou o banco sem serviço, inicie `mongod` com a configuração e o diretório de dados da sua instalação e mantenha esse processo em execução. Se habilitou autenticação local, use uma URI com as credenciais correspondentes.

<a id="ambiente"></a>

## 6. Configurar backend e frontend

### 6.1 Backend: `back/.env`

Abra o arquivo no editor. Exemplo para banco local e a chave de teste já presente no sketch:

```dotenv
PORT=3000
NODE_ENV=development
MONGODB_URI=mongodb://127.0.0.1:27017/redevigia
API_KEY=rv-dev-key-2026
CORS_ORIGIN=http://localhost:5173
```

| Variável | Configuração |
| --- | --- |
| `PORT` | Porta da API; referência: `3000`. |
| `NODE_ENV` | `development` para executar o MVP em desenvolvimento. |
| `MONGODB_URI` | URI do seu banco local ou Atlas. |
| `API_KEY` | Valor que o Arduino enviará no header `x-api-key`. Precisa coincidir com o sketch. |
| `CORS_ORIGIN` | Origem do painel; referência: `http://localhost:5173`. |

O modelo `.env.example` contém uma chave de exemplo diferente da chave já escrita no firmware. **Copiar o `.env.example` sem alinhar `API_KEY` com o `.ino` resulta em HTTP 401.**

A chave `rv-dev-key-2026` é uma credencial pública de teste presente no projeto. Para usar uma chave própria, altere `API_KEY` no `.env` e no `.ino`, reinicie a API e grave o sketch novamente. O backend atual desativa a verificação de chave quando `API_KEY` fica vazia, independentemente do valor de `NODE_ENV`.

### 6.2 Frontend: `frontend/.env.local`

```dotenv
BACKEND_URL=http://localhost:3000
NEXT_PUBLIC_DEFAULT_DEVICE_ID=RV-001
```

`BACKEND_URL` deve conter o endereço da API **sem `/api` no final**. O Next.js encaminha as consultas do painel para esse destino. Não é necessário apontar o frontend para o ngrok quando os dois serviços estão na mesma máquina.

`NEXT_PUBLIC_DEFAULT_DEVICE_ID` define o dispositivo selecionado inicialmente. Use o mesmo ID do firmware ou selecione o dispositivo correto no painel. O frontend consulta dados e não precisa receber a chave de escrita do Arduino.

Depois de alterar a configuração do frontend, reinicie seu servidor. Na execução com build, reconstrua o frontend após mudar as configurações relevantes.

### 6.3 Três credenciais diferentes

| Credencial | Onde fica | Finalidade |
| --- | --- | --- |
| `API_KEY` da Rede Vigia | `back/.env` e `.ino` | Autorizar POST de medições. |
| Authtoken ngrok | Configuração local do agente | Autenticar o computador na conta ngrok. Não vai no Arduino. |
| Usuário/senha MongoDB | URI em `back/.env` | Permitir ao backend acessar o banco. Não vão no Arduino nem no frontend. |

O firmware utiliza HTTP sem TLS no trecho entre modem e endpoint público; a chave de teste e os dados trafegam sem criptografia nesse trecho. O endpoint HTTPS para o navegador não muda o protocolo usado pelo modem. O túnel publica também as rotas GET da API, que atualmente não exigem chave.

<a id="ngrok"></a>

## 7. Instalar e configurar o ngrok do zero

### 7.1 Obter o executável e a conta

1. Crie sua conta no [ngrok](https://dashboard.ngrok.com/signup).
2. Obtenha seu authtoken no painel e consulte o domínio de desenvolvimento atribuído à conta em [Domains](https://dashboard.ngrok.com/domains).
3. Baixe o agente Windows correspondente à arquitetura da máquina na [página oficial de download](https://ngrok.com/download/windows).
4. Para seguir os comandos deste guia, crie a pasta abaixo e coloque o executável extraído em `back/.tools/ngrok.exe`.

```powershell
New-Item -ItemType Directory -Path .\back\.tools -Force
.\back\.tools\ngrok.exe version
```

O primeiro comando cria a pasta; o segundo só funcionará **depois que você extrair o executável nela**. Se preferir uma instalação global que disponibilize `ngrok` no PATH, substitua o caminho do executável nos comandos seguintes por `ngrok`.

### 7.2 Autenticar e configurar os dois endpoints

Substitua `SEU_AUTHTOKEN` pelo token da sua conta:

```powershell
.\back\.tools\ngrok.exe config add-authtoken "SEU_AUTHTOKEN" --config .\back\.tools\ngrok.yml
```

Abra `back/.tools/ngrok.yml`. Mantenha o token real gerado e configure os endpoints no mesmo arquivo, com esta estrutura:

```yaml
version: "3"
agent:
  authtoken: SEU_AUTHTOKEN

endpoints:
  - name: arduino-http
    url: http://SEU_DOMINIO_NGROK
    upstream:
      url: http://127.0.0.1:3000

  - name: navegador-https
    url: https://SEU_DOMINIO_NGROK
    upstream:
      url: http://127.0.0.1:3000
```

Substitua `SEU_DOMINIO_NGROK` pelo **hostname completo da sua conta**, sem caminho, por exemplo `nome-atribuido.ngrok-free.dev`. Não coloque `/api/medicoes` no YAML. Use espaços para a indentação, não tabulações. [Configuração ngrok v3](https://ngrok.com/docs/gateway/agent/config/v3).

O hostname `kindling-factor-unfounded.ngrok-free.dev`, presente no sketch e na documentação específica, pertence à configuração de desenvolvimento original. Ele não é criado nem transferido para sua conta ao clonar o Git. **Use seu domínio no YAML e no firmware.**

Valide a sintaxe, sem abrir o túnel ainda:

```powershell
.\back\.tools\ngrok.exe config check --config .\back\.tools\ngrok.yml
```

### 7.3 Por que HTTP e HTTPS?

O sketch envia HTTP. O endpoint `arduino-http` deve responder diretamente nesse protocolo, sem redirecionar para HTTPS. O endpoint `navegador-https` permite abrir o health check no navegador. Criar somente um dos endpoints não ativa automaticamente o outro.

Na máquina de desenvolvimento, os dois encaminham para a mesma API local. Não é necessário habilitar TLS no Express para essa configuração.

### 7.4 Limites e avisos do serviço

Na consulta de referência em setembro de 2026, o plano gratuito informa **20.000 requisições HTTP por mês**. Com um envio a cada 20 segundos, o teto teórico seria 4.320 envios por dia, antes do tempo gasto com conexão e requisição. Portanto, esse intervalo é adequado para demonstrações, mas não para presumir operação contínua durante um mês no plano gratuito. Confira [limites vigentes e consumo da conta](https://ngrok.com/docs/pricing-limits/free-plan-limits).

O navegador pode exibir a página de aviso do ngrok; para sua própria URL de teste, use **Visit Site**. Nos comandos de diagnóstico deste guia, o header `ngrok-skip-browser-warning: 1` permite receber diretamente a resposta da API. Ele não substitui a API key.

<a id="arduino"></a>

## 8. Configurar, calibrar e gravar o Arduino

### 8.1 Abrir o arquivo correto

Abra na IDE Arduino:

```text
firmware/rede_vigia/rede_vigia.ino
```

A pasta e o arquivo têm o mesmo nome, como esperado pela IDE. Todo o código está nesse arquivo. Não use uma aba antiga apontando para `firmware/rede_vigia.ino`, caminho que não corresponde à organização atual.

Em **Ferramentas → Placa**, selecione **Arduino UNO**, no pacote **Arduino AVR Boards**. Instale o pacote pelo Gerenciador de Placas se necessário. A biblioteca `SoftwareSerial` já faz parte desse pacote.

### 8.2 Ajustar a configuração da sua montagem

No início do `.ino`, configure:

```cpp
#define DEVICE_ID "RV-001"
#define API_URL "http://SEU_DOMINIO_NGROK/api/medicoes"
#define API_KEY "MESMO_VALOR_DE_BACK_ENV"

#define GPRS_APN "java.claro.com.br"
#define GPRS_USER "Claro"
#define GPRS_PASSWORD "Claro"
```

Os valores de APN acima são os utilizados no sketch para Claro Brasil e constam na [orientação da Claro](https://configuraraparelhos.claro.com.br/sistema-operacional/samsung-android-16/como-configurar-a-internet-do-seu-aparelho-apn). Confirme os valores do seu chip, especialmente em planos IoT/M2M ou outra operadora. Esses dados são distintos da senha de uma rede Wi-Fi; o firmware não utiliza Wi-Fi.

O `DEVICE_ID` precisa ter de 1 a 20 caracteres; o firmware aceita letras, números, ponto, hífen e sublinhado. Mantenha esse ID no painel para acompanhar a placa correta.

Os pinos configurados são D4/D5 para TRIG/ECHO e D7/D8 para RX/TX. Tanto o Monitor Serial quanto a UART do modem estão configurados em **9600 baud**.

### 8.3 Calibrar o nível de água

O sensor mede a distância entre ele e a água. Quanto mais a água sobe, menor essa distância.

1. Fixe o sensor na posição em que será utilizado.
2. Coloque o recipiente no nível mínimo/vazio de referência e meça a distância: esse valor é `D0_CM`.
3. Coloque a água no nível máximo de referência e meça novamente: esse valor é `D100_CM`.
4. Ajuste as constantes no sketch, respeitando a zona cega do sensor.

```cpp
const float D0_CM = 50.0f;
const float D100_CM = 20.0f;
```

**50/20 cm são exemplos que já estão no arquivo; não são uma calibração universal.** Na configuração atual, a validação exige:

```text
20 cm <= D100_CM < D0_CM <= 500 cm
```

O cálculo é:

```text
nivel_percentual = ((D0_CM - distancia_cm) / (D0_CM - D100_CM)) × 100
```

O resultado é limitado a 0–100% e arredondado para uma casa decimal antes de classificar o estado.

| Estado | Regra do firmware |
| --- | --- |
| `0` — Normal | Nível até 50%, inclusive. |
| `1` — Atenção | Nível acima de 50% até 75%, inclusive. |
| `2` — Risco | Nível acima de 75%. |

Com `D0_CM=50` e `D100_CM=20`, uma distância de 31,7 cm resulta em **61,0%**, estado Atenção. Uma distância de 83 cm resulta em **0%**, pois está além da referência de vazio. Isso não demonstra falha no HTTP; indica que a calibração deve ser comparada ao recipiente real.

### 8.4 Como a leitura é filtrada

O sketch preserva a rotina de disparo LOW de 5 µs, HIGH de 12 µs, timeout de 38.000 µs e conversão `tempo × 0,0343 / 2` para centímetros. Faz até sete tentativas, espaçadas por 80 ms, para obter até cinco amostras entre 20 e 500 cm; exige ao menos três amostras válidas e seleciona a amostra central ordenada.

Se não houver amostras suficientes, imprime `SEM ECO valido` e não envia uma medição. Ausência de eco não é registrada como nível zero. A resolução exibida de uma casa decimal não garante precisão física de 0,1 cm: posicionamento, ambiente, variante e calibração continuam influenciando a medida.

### 8.5 Carregar e observar

1. Conecte o UNO por cabo USB de dados.
2. Selecione a porta correspondente em **Ferramentas → Porta**. `COM6` foi usada na bancada, mas pode ser outra na sua máquina.
3. Clique em **Verificar** e corrija qualquer erro de compilação antes de continuar.
4. Clique em **Carregar** e aguarde a confirmação do upload.
5. Abra o **Monitor Serial em 9600 baud**.

Se a porta não aparecer, teste o cabo e identifique o conversor USB da sua placa. Placas compatíveis podem precisar do driver correspondente; obtenha-o do fabricante. Não é necessário alterar D7/D8 para corresponder ao número da porta COM.

O firmware executa automaticamente após ligar a placa e não espera o Monitor Serial. A abertura do monitor pode reiniciar o UNO. Ele também não oferece ponte manual para digitar comandos AT; selecionar uma terminação de linha no monitor não transforma esse sketch em um terminal AT.

Depois de trocar URL, chave, APN, pinos ou calibração, **grave novamente o sketch**. Salvar o arquivo no editor não atualiza a placa.

<a id="iniciar"></a>

## 9. Iniciar o MVP

### 9.1 Ordem de inicialização

Use esta ordem: **banco → backend → ngrok → frontend → protótipo**. Para acompanhar os processos e encerrá-los depois, mantenha cada serviço em um terminal separado.

Todos os terminais abaixo devem começar na **raiz `redevigia`**. No PowerShell, `Get-Location` mostra a pasta atual.

### 9.2 Banco

Se usar Atlas, confirme que o cluster está disponível e o IP do computador está autorizado. Se usar o serviço MongoDB local, confirme que está em execução:

```powershell
Get-Service -Name MongoDB
```

Esse comando é somente para quem instalou MongoDB como serviço local; não se aplica ao Atlas.

### 9.3 Terminal 1 — backend

```powershell
npm.cmd --prefix .\back run dev
```

Aguarde a confirmação de conexão ao MongoDB e a mensagem `Servidor: http://localhost:3000`. Na inicialização, a conexão ao banco pode levar alguns segundos.

Em outro terminal, valide:

```powershell
Invoke-RestMethod http://localhost:3000/api/health
```

Resultado esperado: `status` igual a `ok` e `database` igual a `connected`. O JSON também contém uptime, timestamp e versão. O campo `status: ok` isoladamente não substitui a conferência do banco.

### 9.4 Terminal 2 — ngrok

```powershell
.\back\.tools\ngrok.exe start --all --config .\back\.tools\ngrok.yml
```

Confirme que aparecem **os dois endpoints**, HTTP e HTTPS, com encaminhamento para a porta 3000. Não inicie outra instância se esse agente já estiver rodando.

Substitua o domínio nos testes:

```powershell
curl.exe -i --max-time 20 -H "ngrok-skip-browser-warning: 1" "http://SEU_DOMINIO_NGROK/api/health"
curl.exe -i --max-time 20 -H "ngrok-skip-browser-warning: 1" "https://SEU_DOMINIO_NGROK/api/health"
```

Os dois devem retornar **HTTP 200** com JSON da Rede Vigia e banco conectado. Os comandos não seguem redirecionamentos: o teste HTTP deve responder diretamente, pois é esse caminho que o SIM800L utilizará.

Para inspecionar o tráfego localmente, abra `http://127.0.0.1:4040`. Não use a função de replay de POST para validar uma medição real: repetir uma requisição aceita pode criar outro registro.

### 9.5 Terminal 3 — frontend

```powershell
npm.cmd --prefix .\frontend run dev
```

Abra **[http://localhost:5173](http://localhost:5173)**. O primeiro acesso pode demorar enquanto o Next.js compila a página.

Valide também a comunicação pelo proxy do frontend:

```powershell
Invoke-RestMethod http://localhost:5173/api/health
```

### 9.6 Ligar o protótipo

Com montagem e calibração verificadas, ligue as fontes do UNO, sensor e modem. A inicialização imprime:

```text
Rede Vigia - sensor + SIM800L + HTTP
Aguardando registro GSM/GPRS...
Abrindo internet GPRS...
Internet GPRS conectada.
```

Algumas mensagens aparecem somente quando a conexão precisa ser aberta. Registro e conexão podem levar dezenas de segundos. O sketch faz tentativas limitadas e volta a tentar em outro ciclo se falhar.

Ele mede **depois** de conseguir conexão, envia o POST e espera pelo menos 20 segundos após o fim do ciclo antes de iniciar outro. A duração total inclui a comunicação com o modem; não espere um POST rigorosamente a cada 20 segundos. Durante os comandos AT bloqueantes, as leituras do sensor ficam pausadas.

<a id="validar"></a>

## 10. Validar a demonstração e usar o painel

### 10.1 Confirmar uma medição real

No Monitor Serial, procure uma sequência como:

```text
Distancia: 31.7 cm | Nivel: 61.0% | Status: 1 | CSQ: 19
Resultado HTTP: 201
Envio confirmado pela API (201).
```

**HTTP 201** é a confirmação do endpoint de criação após salvar a medição. O `OK` de um comando AT, sozinho, não comprova que um POST foi aceito.

Consulte a medição pelo backend:

```powershell
Invoke-RestMethod http://localhost:3000/api/medicoes/ultima/RV-001 | ConvertTo-Json -Depth 6
```

Substitua `RV-001` se alterou o ID. Confira distância, nível, estado e timestamp. Os dispositivos aparecem na listagem a partir das medições gravadas; não há cadastro prévio obrigatório da placa.

### 10.2 Conferir o painel

1. Selecione o mesmo dispositivo configurado no Arduino.
2. Mantenha a atualização automática habilitada ou use **Atualizar dados**.
3. Mude a altura da água/alvo de forma controlada e aguarde uma nova medição e a atualização do painel.
4. Compare a leitura com a distância medida fisicamente e a calibração.
5. Confira histórico e estatísticas no período selecionado.

O painel consulta a cada 15 segundos enquanto a aba está visível. Uma leitura com mais de 120 segundos recebe indicação de desatualização; isso descreve a idade do registro, não uma detecção direta de conexão do modem.

O histórico é paginado. O gráfico usa até 200 leituras recentes do período; o CSV exporta somente a página visível. Sem dados, o painel mostra ausência de valores, não inventa medições nem transforma ausência em nível normal.

### 10.3 Testar fora do USB do computador

Primeiro valide o conjunto pelo USB. Depois, desligue a montagem e prepare a alimentação externa correta do UNO. Confira se sensor e modem mantêm suas alimentações e GND comum sem depender do cabo USB.

Ligue novamente e acompanhe os novos registros pelo painel ou pela API. O Monitor Serial do PC não recebe dados de uma placa desconectada do USB; ao reconectar e abrir o monitor, o UNO pode reiniciar. Não confunda esse reinício com um registro contínuo do funcionamento anterior.

Se após trocar a fonte aparecer `SEM ECO valido`, confira tensão no sensor sob carga, GND, TRIG/ECHO, posição e distância antes de atribuir a falha ao ngrok.

### 10.4 Critério de conclusão do MVP

- [ ] API local responde e informa MongoDB conectado.
- [ ] Endpoints públicos HTTP e HTTPS respondem com o health check.
- [ ] SIM800L registra na rede e abre a conexão GPRS.
- [ ] Sensor produz medições coerentes com o recipiente e a calibração.
- [ ] Arduino recebe HTTP 201 em envios sucessivos.
- [ ] API e painel mostram as novas leituras do dispositivo correto.
- [ ] Funcionamento com alimentação externa foi verificado, se fizer parte da apresentação.
- [ ] Evidências e dados necessários foram salvos antes do encerramento.

<a id="api"></a>

## 11. Contrato HTTP e endpoints

### 11.1 Requisição do Arduino

```text
POST http://SEU_DOMINIO_NGROK/api/medicoes
Content-Type: application/x-www-form-urlencoded
x-api-key: MESMA_CHAVE_DO_BACKEND

device_id=RV-001&distancia_cm=31.7&nivel_percentual=61.0&status=1&signal=19
```

| Campo | Regra |
| --- | --- |
| `device_id` | Identificação do dispositivo, até 20 caracteres; o sketch aplica a restrição de caracteres descrita anteriormente. |
| `distancia_cm` | API aceita 0–500 cm; o filtro do firmware usa 20–500 cm. |
| `nivel_percentual` | Número entre 0 e 100, calculado pelo firmware. |
| `status` | Inteiro 0, 1 ou 2, calculado pelo firmware. |
| `signal` | Opcional, inteiro de 0 a 31. CSQ 99 significa desconhecido e é omitido pelo sketch. Não representa porcentagem nem dBm. |
| `timestamp` | Gerado pelo backend no recebimento; não precisa ser enviado pelo Arduino. |

A API também aceita JSON. Ela valida os limites dos campos, mas não recalcula a calibração nem a coerência entre percentual e estado; essa responsabilidade está no firmware. O backend acrescenta `status_label` a partir do estado recebido.

### 11.2 Rotas disponíveis

| Método | Rota | Uso |
| --- | --- | --- |
| GET | `/api/health` | Estado da API e conexão ao banco. |
| POST | `/api/medicoes` | Gravar uma medição; requer `x-api-key` quando configurada. |
| GET | `/api/medicoes/ultima/:deviceId` | Última medição do dispositivo. |
| GET | `/api/medicoes/historico/:deviceId` | Histórico com paginação e período. |
| GET | `/api/medicoes/estatisticas/:deviceId` | Estatísticas do período. |
| GET | `/api/dispositivos` | Dispositivos com medições e resumo de registros. |

Exemplo de histórico:

```powershell
Invoke-RestMethod "http://localhost:3000/api/medicoes/historico/RV-001?pagina=1&limite=20" | ConvertTo-Json -Depth 6
```

Os filtros de período utilizam `de` e `ate`, com datas em formato ISO. Veja os contratos completos em [back/swagger.yaml](back/swagger.yaml) e importe [back/redevigia.json](back/redevigia.json) no Postman, se desejar. O projeto não inicia automaticamente uma página Swagger UI.

<a id="diagnostico"></a>

## 12. Diagnóstico de problemas

Investigue na ordem **alimentação e UART → SIM/rede → GPRS → sensor → HTTP → banco/painel**. Uma falha em uma etapa pode impedir que a próxima seja executada.

| Sintoma | O que conferir |
| --- | --- |
| `npm` não reconhecido | Instalação do Node e PATH; reabra o terminal após instalar. |
| `npm.ps1` bloqueado | Use `npm.cmd` no PowerShell. |
| Erro de engine/dependências | Use Node 24 LTS e instale com `npm ci` na aplicação correta. |
| API não responde em 3000 | Aguarde a conexão inicial ao banco; consulte o terminal do backend. |
| Falha MongoDB/Atlas | URI, usuário de banco, senha, IP autorizado e acesso de rede. Para banco local, confira o servidor/serviço. |
| `EADDRINUSE` | Outra instância ocupa a porta. Identifique-a antes de iniciar um segundo processo. |
| `ngrok` não reconhecido | Use `.\back\.tools\ngrok.exe` a partir da raiz ou instale o agente no PATH. |
| Ngrok recusa autenticação | Token válido da sua conta e arquivo correto em `--config`. |
| Ngrok não aceita o domínio | Use o hostname atribuído à sua conta, não o domínio de outra pessoa. |
| HTTPS offline, mas HTTP funciona | Confira se iniciou os dois endpoints com `start --all`. |
| HTML de aviso no health check público | No navegador, Visit Site; no cliente de teste, header `ngrok-skip-browser-warning: 1`. |
| Ngrok responde, mas não alcança a API | Backend deve estar ativo e `upstream.url` deve apontar para sua porta. |
| `SIM800L sem resposta` | Modem ligado, fonte, GND comum, TX/RX cruzados, níveis lógicos e baud 9600. |
| Quadrados/caracteres ilegíveis | Baud, ruído, GND, nível lógico e alimentação. O log sozinho não identifica qual deles falhou. |
| `SIM nao pronto` | Chip instalado, PIN desabilitado, tempo de inicialização e alimentação. Uma falha inicial pode se resolver em ciclos posteriores. |
| `Sem registro 2G` | Cobertura GSM/GPRS, antena, chip ativo, plano e habilitação pela operadora. |
| `Falha GPRS` | APN/usuário/senha corretos, dados disponíveis e fonte estável. |
| `SEM ECO valido` | Menos de três amostras válidas; confira VCC, GND, TRIG/ECHO, modo do sensor, alvo, zona cega e faixa de 20–500 cm. |
| Nível preso em 0% ou 100% | Compare a distância com `D0_CM`/`D100_CM`; o cálculo é limitado nesses extremos. |
| Banner inicial reaparece | A placa reiniciou. Pode ser troca de alimentação, abertura do Monitor Serial ou outro reset; repetição espontânea pede revisão da fonte. |
| Funciona no PC e falha na fonte externa | Confira entrada usada no UNO, tensões sob carga, alimentação do sensor e GND comum independente do USB. |
| `ERROR` seguido de HTTP 201 | Pode ser a tentativa de fechar bearer/sessão já fechados. O código tolera erro nesses comandos de limpeza; não significa que todo `ERROR` é inofensivo. |
| HTTP 400 | Dados fora do contrato; examine os campos e a resposta de validação. |
| HTTP 401 | Chave do sketch diferente de `back/.env`; reinicie a API e regrave o sketch após corrigir. |
| HTTP 3xx | Redirecionamento; valide a URL HTTP direta. O sketch não segue redirects. |
| HTTP 404 | Confira domínio, túnel, rota e ID. Um GET de última medição sem registros também retorna 404, com `NOT_FOUND`. |
| HTTP 429 | Confira os limites de uso do serviço que respondeu. |
| Código 600 ou superior do SIM800L | Erro interno do modem/rede; confira GPRS, DNS e servidor. |
| Resultado HTTP -1 | Nenhuma confirmação válida no prazo ou falha em um comando; não prova que o servidor deixou de gravar. |
| Painel abre, mas API aparece indisponível | Confira backend e `BACKEND_URL`; reinicie o frontend após alterar a configuração. |
| Painel sem dados ou desatualizado | ID selecionado, período, atualização automática e horário da última medição. |

O firmware não repete automaticamente o mesmo POST após um timeout, porque o servidor pode já ter gravado a medição. A próxima tentativa utiliza uma nova leitura. Não há fila persistente de medições offline no sketch atual.

<a id="encerrar"></a>

## 13. Encerrar o MVP

### 13.1 Encerramento normal após a demonstração

1. Salve capturas, exportações ou anotações necessárias. Se usar o CSV do painel, lembre que ele contém a página visível.
2. Aguarde a conclusão do último envio e desligue as fontes do protótipo de forma coordenada. Evite deixar sinais UART alimentando um módulo desligado.
3. No terminal do **ngrok**, pressione **Ctrl+C**. Os endpoints públicos deixam de encaminhar requisições.
4. No terminal do **frontend**, pressione **Ctrl+C**.
5. No terminal do **backend**, pressione **Ctrl+C**.
6. Se o terminal perguntar se deseja finalizar o trabalho em lotes, confirme a finalização.
7. Feche o Monitor Serial e os terminais após a devolução do prompt.

Esse encerramento não apaga medições, arquivos `.env`, configuração do ngrok nem o programa gravado na placa. Quando o Arduino receber alimentação novamente, o sketch voltará a executar.

### 13.2 Banco de dados

No Atlas, parar o backend encerra o uso da aplicação, mas não exclui nem encerra o cluster. Não é necessário apagar o banco para concluir uma apresentação.

Se iniciou um MongoDB local dedicado a esse teste e deseja pará-lo, use a forma correspondente à sua instalação. Para o serviço padrão no Windows, com permissão administrativa:

```powershell
Stop-Service -Name MongoDB
```

Execute isso somente se o serviço não estiver sendo utilizado por outras aplicações. Se o banco foi iniciado manualmente em um terminal, finalize-o de forma normal nesse terminal.

### 13.3 Conferir as portas

```powershell
Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
    Where-Object { $_.LocalPort -in 3000, 5173, 4040 } |
    Select-Object LocalAddress, LocalPort, OwningProcess
```

Se não houver saída, não há processos escutando nessas três portas. Se aparecer um processo, identifique seu comando antes de encerrá-lo. O MongoDB local, quando mantido ligado, utiliza outra porta.

### 13.4 Se os serviços foram iniciados em segundo plano

Fechar um terminal ou janela não garante que um processo iniciado em segundo plano terminou. Liste os comandos relevantes:

```powershell
Get-CimInstance Win32_Process |
    Where-Object { $_.Name -in 'node.exe', 'ngrok.exe' } |
    Select-Object ProcessId, ParentProcessId, CommandLine
```

Identifique os processos que pertencem **a esta cópia do Rede Vigia**. O backend em modo watch e o Next.js podem ter processos filhos; encerrar somente o filho pode fazer o supervisor iniciá-lo novamente.

Para encerrar uma árvore já identificada, substitua `12345` pelo PID real do processo principal do serviço:

```powershell
taskkill.exe /PID 12345 /T /F
```

Esse é um recurso de finalização forçada para processos que ficaram em segundo plano; prefira Ctrl+C nos terminais originais. Não use um comando para matar todos os `node.exe` da máquina. Repita a conferência das portas depois.

<a id="reiniciar"></a>

## 14. Reiniciar em outro dia

Não é necessário reinstalar dependências, recriar o token nem carregar o Arduino a cada execução, desde que a configuração e o firmware continuem os mesmos.

1. Confirme o banco, a internet do computador e, no Atlas, o IP autorizado.
2. Abra três terminais na raiz do repositório.
3. Inicie os serviços abaixo, aguardando a API conectar antes de testar o túnel.
4. Ligue o protótipo e confira uma nova medição HTTP 201.

| Terminal | Comando a partir da raiz |
| --- | --- |
| Backend | `npm.cmd --prefix .\back run dev` |
| Ngrok | `.\back\.tools\ngrok.exe start --all --config .\back\.tools\ngrok.yml` |
| Frontend | `npm.cmd --prefix .\frontend run dev` |

Abra `http://localhost:5173`. Para finalizar, siga a seção de encerramento; não há um script único de start/stop na raiz do projeto atual.

<a id="verificacoes"></a>

## 15. Build e verificações opcionais

Esses comandos são úteis para manutenção e validação do software. Não são necessários a cada demonstração e não substituem o teste com sensor, modem e fonte reais.

### 15.1 Backend compilado

Na raiz:

```powershell
npm.cmd --prefix .\back run build
npm.cmd --prefix .\back start
```

O build gera `back/dist`; `start` executa `dist/app.js`. Encerre a instância `dev` antes de iniciar `start`, pois usam a mesma porta. A configuração do banco continua necessária.

### 15.2 Frontend compilado

```powershell
npm.cmd --prefix .\frontend run build
npm.cmd --prefix .\frontend start
```

Encerre o frontend de desenvolvimento antes de iniciar a versão compilada. A porta continua sendo 5173. O backend e o ngrok continuam sendo processos separados; executar build não os inicia.

### 15.3 Testes e análise estática

```powershell
npm.cmd --prefix .\frontend run typecheck
npm.cmd --prefix .\frontend run lint
npm.cmd --prefix .\frontend test
npm.cmd --prefix .\frontend run test:e2e
```

Os testes E2E usam Google Chrome instalado, iniciam ou reaproveitam o frontend e simulam respostas da API no navegador. Não comprovam integração física com o Arduino nem escrevem medições reais no MongoDB. Artefatos ficam em `frontend/test-results`.

O backend atual possui comando de build, mas não possui script `npm test`. O comando `format` do frontend modifica arquivos; não é necessário para apenas executar o MVP.

### 15.4 Compilação opcional com Arduino CLI

Se já utiliza o [Arduino CLI](https://arduino.github.io/arduino-cli/latest/installation/), pode verificar o sketch sem fazer upload:

```powershell
arduino-cli core update-index
arduino-cli core install arduino:avr
arduino-cli compile --fqbn arduino:avr:uno .\firmware\rede_vigia
```

Isso instala/usa o core AVR da sua instalação do CLI. A compilação verifica o software para o UNO; não testa tensões, cobertura celular, ecos ou gravações HTTP pela placa.

<a id="simulacao"></a>

## 16. Teste sem protótipo e seed

É possível verificar o backend e o painel sem Arduino, executando banco, API e frontend. O ngrok só é necessário quando deseja testar o acesso externo.

### 16.1 Inserir uma medição manual identificada como teste

O comando abaixo **grava um registro real no banco configurado**, usando um ID separado para não se confundir com a placa. Ajuste a chave ao seu `.env`:

```powershell
$testHeaders = @{ 'x-api-key' = 'SUA_API_KEY' }
$testBody = 'device_id=RV-TESTE-MANUAL&distancia_cm=31.7&nivel_percentual=61.0&status=1'
Invoke-RestMethod -Method Post -Uri 'http://localhost:3000/api/medicoes' -Headers $testHeaders -ContentType 'application/x-www-form-urlencoded' -Body $testBody
```

Selecione `RV-TESTE-MANUAL` no painel. Esse teste valida o caminho local da API até o banco, mas não valida o sensor, SIM800L nem rede móvel.

### 16.2 Seed existente

O repositório inclui:

```powershell
npm.cmd --prefix .\back run seed
```

**Não execute como parte da inicialização normal.** O script atual remove todas as medições de `RV-001` no banco selecionado e insere 20 registros simulados. Use somente em banco de teste descartável, nunca no banco em que pretende preservar as leituras da apresentação.

Os valores simulados são exemplos independentes da calibração física do seu recipiente. O script também imprime a URI de conexão no terminal; evite compartilhar esse log se a URI contiver credenciais.

<a id="referencias"></a>

## 17. Linux, macOS e documentação complementar

### 17.1 Adaptação dos comandos

Os comandos das aplicações usam o mesmo npm. Em Linux/macOS, troque `npm.cmd` por `npm` e utilize `/` nos caminhos:

```bash
# Na raiz do clone, primeira instalação
npm --prefix ./back ci
npm --prefix ./frontend ci

# Somente se esses arquivos locais ainda não existirem
test -f back/.env || cp back/.env.example back/.env
test -f frontend/.env.local || cp frontend/.env.example frontend/.env.local
```

Em terminais separados, na raiz:

```bash
# Terminal do backend
npm --prefix ./back run dev
```

```bash
# Terminal do frontend
npm --prefix ./frontend run dev
```

Instale o agente ngrok correspondente ao seu sistema e arquitetura. Com `ngrok` disponível no PATH, mantenha o mesmo YAML v3 descrito neste guia em `back/.tools/ngrok.yml`:

```bash
mkdir -p back/.tools
ngrok config add-authtoken "SEU_AUTHTOKEN" --config ./back/.tools/ngrok.yml
# Edite o YAML para incluir os dois endpoints antes de iniciar.
ngrok config check --config ./back/.tools/ngrok.yml
ngrok start --all --config ./back/.tools/ngrok.yml
```

Use `curl` no lugar de `curl.exe`, Ctrl+C para finalizar os processos e o gerenciador de serviços do sistema se utilizar MongoDB local. A porta da placa pode ser `/dev/ttyACM*`, `/dev/ttyUSB*` ou `/dev/cu.*`; selecione a detectada pela IDE e confira permissões do dispositivo no sistema. Os procedimentos elétricos e a calibração são os mesmos.

### 17.2 Documentação do repositório

- [Backend: instalação e API](back/readme.md)
- [Túnel ngrok](back/TUNNEL.md)
- [Frontend: comportamento e testes](frontend/README.md)
- [Firmware: operação e montagem](firmware/README.md)
- [Sketch único para Arduino UNO](firmware/rede_vigia/rede_vigia.ino)
- [Especificação OpenAPI](back/swagger.yaml)
- [Coleção Postman](back/redevigia.json)

Este guia descreve os comandos, pinos, contratos e limitações da implementação atual. Configurações de uma máquina, domínio de uma conta e calibração de um recipiente precisam ser ajustados ao reproduzir o protótipo em outro ambiente.
