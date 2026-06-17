# DOGFEEDER

Alimentador automático para pets baseado em ESP32, com backend em PHP/MySQL e
interface web estática. O microcontrolador faz *polling* periódico de uma API
HTTP para decidir quando acionar o motor: por horário programado (até três
horários/dia) ou por comando manual disparado pela interface web.

O sistema é dividido em quatro partes que se comunicam exclusivamente por HTTP
e SQL — não há acoplamento direto entre frontend e firmware; ambos conversam
apenas com a API PHP, que persiste o estado no MySQL.

---

## Arquitetura

```
                         HTTP (GET estado/horários/tempo)
   ┌──────────────┐  ◄───────────────────────────────────  ┌──────────────┐        ┌──────────┐
   │    ESP32     │                                         │   API PHP    │  SQL   │  MySQL   │
   │  (firmware)  │  ───────────────────────────────────►  │  (mysqli)    │ ◄────► │ 2 bancos │
   └──────────────┘     HTTP (POST reset estado=0)          └──────────────┘        └──────────┘
                                                                   ▲
                         HTTP (POST estado=1 / horários / tempo)   │
   ┌──────────────┐  ──────────────────────────────────────────────┘
   │   Web (SPA   │
   │  estática)   │
   └──────────────┘
```

Fluxo de dados:

- **Estado do motor** é uma flag compartilhada (`0`/`1`) no MySQL. A web grava
  `1` para "alimentar agora"; o ESP32 lê esse valor, executa o pulso e grava
  `0` de volta para encerrar o comando.
- **Horários** e **tempo de alimentação** são lidos pelo ESP32 a cada iteração
  do `loop()` e usados para decidir o acionamento por agendamento.

O estado é **global e único** — não há conceito de usuário/dispositivo. Uma
instância da API atende um único alimentador.

---

## Stack

| Camada    | Tecnologia                                                                 |
|-----------|----------------------------------------------------------------------------|
| Firmware  | C++ (Arduino), `WiFi.h`, `HTTPClient`, `ArduinoJson`, `time.h` (SNTP)      |
| Backend   | PHP procedural com `mysqli`, JSON sobre HTTP, CORS aberto (`*`)            |
| Banco     | MySQL — dois schemas: `horariosDeAlimentacao` e `estado_motor`            |
| Frontend  | HTML/CSS/JS puro, empacotado com Parcel 2                                  |

---

## Estrutura do repositório

```
DOGFEEDER/
├── ESP32 CODE/CODE-1.0/CODE-1.0.ino     # firmware (loop de polling + acionamento)
├── API PHP/
│   ├── Motor estados/
│   │   ├── estadomotor.php               # GET  estado atual do motor
│   │   └── postestadomotor.php           # POST atualiza estado (CORS)
│   ├── getinfo horarios/get.php          # GET  últimos 3 horários
│   ├── insertinfo horarios/insertdata.php# POST insere novos horários
│   └── tempoalimentacao/
│       ├── gettempoalimentacao.php       # GET  tempo de alimentação (array)
│       └── inserttempoalimentacao.php    # POST atualiza tempo de alimentação
├── SITE/                                  # interface web (Parcel)
│   ├── *.html / *.css / script.js
│   ├── public/                            # assets
│   └── package.json
└── DATABASE/                              # scripts SQL de referência
```

---

## Modelo de dados

Dois bancos separados. **Atenção:** `get.php` referencia o banco como
`horariosDeAlimentacao`, enquanto `insertdata.php` e os scripts de tempo usam
`horariosdealimentacao` (minúsculo). Em MySQL no Windows isso é equivalente,
mas em Linux (case-sensitive) são bancos distintos — padronize o nome ao
implantar.

```sql
-- Banco 1: horários e duração do pulso
CREATE DATABASE IF NOT EXISTS horariosDeAlimentacao;
USE horariosDeAlimentacao;

CREATE TABLE Horario1 (Id INT PRIMARY KEY AUTO_INCREMENT, horaAlimentacao1 INT, minutoAlimentacao1 INT);
CREATE TABLE Horario2 (Id INT PRIMARY KEY AUTO_INCREMENT, horaAlimentacao2 INT, minutoAlimentacao2 INT);
CREATE TABLE Horario3 (Id INT PRIMARY KEY AUTO_INCREMENT, horaAlimentacao3 INT, minutoAlimentacao3 INT);
CREATE TABLE tempo    (Id INT PRIMARY KEY AUTO_INCREMENT, tempoalimentacao INT);

-- 'tempo' é atualizado via UPDATE; precisa de uma linha inicial
INSERT INTO tempo (tempoalimentacao) VALUES (5);

-- Banco 2: estado do motor
CREATE DATABASE IF NOT EXISTS estado_motor;
USE estado_motor;

CREATE TABLE IF NOT EXISTS liga_desliga (id INT AUTO_INCREMENT PRIMARY KEY, estado INT);

-- 'liga_desliga' é atualizado via UPDATE; precisa de uma linha inicial
INSERT INTO liga_desliga (estado) VALUES (0);
```

Observações sobre o comportamento das tabelas (derivado do código PHP):

- `Horario1/2/3` crescem por `INSERT` a cada gravação; as leituras usam sempre
  `ORDER BY Id DESC LIMIT 1` (último registro). Não há limpeza — a tabela
  acumula histórico.
- `tempo` e `liga_desliga` são modificadas por `UPDATE`, portanto **exigem uma
  linha pré-existente** (os `INSERT` iniciais acima). Sem isso, gravar tempo ou
  estado não tem efeito.

---

## API HTTP

O firmware e o frontend chamam as rotas sob o prefixo `/dog/...`. Esses
caminhos **não correspondem 1:1** às pastas em disco (que contêm espaços e
nomes diferentes), então o servidor precisa expô-los via `Alias`/`Rewrite`
(Apache) ou configuração equivalente. Mapeamento esperado:

| Método | Rota usada pelo cliente                      | Arquivo no disco                               |
|--------|----------------------------------------------|------------------------------------------------|
| GET    | `/dog/motor/estadomotor`                     | `Motor estados/estadomotor.php`                |
| POST   | `/dog/motor/postestadomotor`                 | `Motor estados/postestadomotor.php`            |
| GET    | `/dog/getinfo/get`                           | `getinfo horarios/get.php`                     |
| POST   | `/dog/insertinfo/insertdata`                 | `insertinfo horarios/insertdata.php`           |
| GET    | `/dog/tempoalimentacao/gettempoalimentacao.php` | `tempoalimentacao/gettempoalimentacao.php`  |
| POST   | `/dog/tempoalimentacao/inserttempoalimentacao`  | `tempoalimentacao/inserttempoalimentacao.php`|

### `GET /dog/motor/estadomotor`
Retorna a última linha de `liga_desliga`.
```json
{ "id": 1, "estado": 0 }
```

### `POST /dog/motor/postestadomotor`
Atualiza o estado do motor. Corpo:
```json
{ "estado": 1 }
```

### `GET /dog/getinfo/get`
Retorna o último horário de cada tabela (campos renomeados para `hora`/`minuto`).
Cada chave pode ser `null` se a tabela estiver vazia.
```json
{
  "Horario1": { "hora": 8,  "minuto": 0  },
  "Horario2": { "hora": 14, "minuto": 30 },
  "Horario3": { "hora": 20, "minuto": 0  }
}
```

### `POST /dog/insertinfo/insertdata`
Insere uma nova linha em cada tabela de horário.
```json
{
  "horario1": { "hora": 8,  "minuto": 0  },
  "horario2": { "hora": 14, "minuto": 30 },
  "horario3": { "hora": 20, "minuto": 0  }
}
```

### `GET /dog/tempoalimentacao/gettempoalimentacao.php`
Retorna **um array** com todos os valores da tabela `tempo`. O firmware lê
apenas o índice `[0]`.
```json
[5]
```

### `POST /dog/tempoalimentacao/inserttempoalimentacao`
Atualiza a duração do pulso (em segundos). Corpo:
```json
{ "tempoalimentacao": 5 }
```

---

## Firmware (ESP32)

Arquivo: [ESP32 CODE/CODE-1.0/CODE-1.0.ino](ESP32%20CODE/CODE-1.0/CODE-1.0.ino)

**Inicialização (`setup`)**
- Configura `GPIO 12` como saída (`ledPin`), nível de repouso `HIGH`.
- Conecta ao Wi-Fi (bloqueante até obter `WL_CONNECTED`).
- Sincroniza o relógio via SNTP: `pool.ntp.org`, offset `GMT-3`, sem horário de
  verão (`daylightOffset_sec = 0`).

**Loop principal** — a cada iteração, sem `delay`, o firmware:
1. `GET /dog/motor/estadomotor` → lê a flag de comando manual.
2. `GET .../gettempoalimentacao.php` → lê a duração do pulso (s) e converte para
   ms (`intervalo`, default 5000).
3. Atualiza hora/minuto locais a partir do relógio sincronizado.
4. `GET /dog/getinfo/get` → carrega os três horários.
5. Avalia as condições de acionamento (manual e por horário).

**Acionamento do motor** — implementado como pulso não-bloqueante via
`millis()`: o pino sai do repouso (`HIGH`) para `LOW` por `intervalo` ms e
retorna a `HIGH`, caracterizando um acionamento ativo-baixo de duração
`tempoalimentacao` segundos.

- **Manual:** quando `estado == "1"`, executa o pulso e, ao final, envia
  `POST .../postestadomotor` com `{"estado": 0}` para limpar o comando.
- **Por horário:** quando `horaAtual`/`minutoAtual` coincidem com um horário
  programado e a flag `demosComidaN` está em `0`, executa o pulso e marca
  `demosComidaN = 1`.

> Por iteração o firmware dispara **três requisições HTTP** sem nenhum atraso
> entre os ciclos — o `loop` roda tão rápido quanto a rede permitir. Em
> produção, recomenda-se adicionar um `delay` (p.ex. 1 s) para reduzir carga na
> rede e na API.

---

## Frontend

Site estático em [SITE/](SITE/), empacotado com Parcel. A lógica principal
está em [SITE/script.js](SITE/script.js):

- Navegação entre telas (`index`, `Menu`, `configure`, `login`, `account`).
- Botão **Feed now** → `POST /dog/motor/postestadomotor` com `{"estado": 1}`
  via `XMLHttpRequest`. O reset para `0` é responsabilidade do ESP32, não do
  frontend.

A tela de configuração grava os três horários (em ordem crescente) e a duração
do pulso (limitada a 10 s na UI).

---

## Execução

### Pré-requisitos
- ESP32 com Wi-Fi e Arduino IDE (bibliotecas: `WiFi`, `HTTPClient`, `ArduinoJson`).
- PHP 7+ e MySQL (ex.: XAMPP/WAMP) com mod de reescrita habilitado.
- Node.js + npm para o frontend.

### 1. Banco de dados
Execute o SQL da seção [Modelo de dados](#modelo-de-dados), incluindo as linhas
iniciais de `tempo` e `liga_desliga`.

### 2. API PHP
1. Sirva a pasta `API PHP/` no servidor web e configure o roteamento `/dog/...`
   conforme a tabela de [API HTTP](#api-http).
2. Ajuste as credenciais MySQL no topo de cada `.php`
   (`$servername`/`$username`/`$password`/`$dbname`).

### 3. Firmware
No `.ino`, configure antes do upload:
```cpp
const char* ssid     = "SEU_SSID";
const char* password = "SUA_SENHA";

const char* serverName1 = "http://SEU_IP:8080/dog/motor/estadomotor";
const char* serverName2 = "http://SEU_IP:8080/dog/getinfo/get";
const char* serverName3 = "http://SEU_IP:8080/dog/tempoalimentacao/gettempoalimentacao.php";

const int ledPin = 12;  // pino do driver do motor
```
A URL de POST do reset de estado está embutida em `enviarSolicitacaoPOST()` e
também precisa apontar para o seu IP.

### 4. Frontend
```bash
cd SITE
npm install
npm start          # dev server (Parcel)
npm run build      # build de produção em ./build
```
Atualize o IP do servidor nas chamadas HTTP de `script.js` (e demais arquivos
que apontam para `192.168.100.103:8080`).

---

## Limitações e considerações de segurança

Este projeto é um protótipo acadêmico. Pontos relevantes antes de qualquer uso
fora de uma rede local controlada:

- **Credenciais versionadas:** o `.ino` contém SSID/senha de Wi-Fi reais e os
  PHP usam `root` sem senha. Troque e remova do histórico antes de publicar.
- **SQL injection:** todos os endpoints interpolam a entrada diretamente na
  query (`UPDATE ... SET estado = $estado`). Use *prepared statements*.
- **Sem autenticação e CORS liberado (`*`):** qualquer cliente na rede pode
  acionar o motor ou reconfigurar horários.
- **Agendamento não reinicia diariamente:** as flags `demosComida1/2/3` são
  marcadas como `1` e **nunca voltam para `0`** no código atual. Cada horário
  dispara no máximo uma vez por ciclo de energia do ESP32 — não há reset à
  meia-noite. Para alimentação diária recorrente, é preciso zerar as flags
  quando o minuto deixa de coincidir.
- **Estado único e global:** não há suporte a múltiplos pets, dispositivos ou
  usuários.
- **Polling agressivo:** loop sem `delay` (ver seção de firmware).

---

## Licença

ISC.
