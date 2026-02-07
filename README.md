# 🐕 DOGFEEDER - Alimentador Automático para Pets

![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)
![ESP32](https://img.shields.io/badge/ESP32-WiFi-green.svg)
![PHP](https://img.shields.io/badge/PHP-API-orange.svg)
![License](https://img.shields.io/badge/license-ISC-lightgrey.svg)

## 📋 Sobre o Projeto

O **DOGFEEDER** é um sistema completo de alimentação automática para animais de estimação. O projeto foi desenvolvido para permitir que tutores automatizem a alimentação de seus pets através de uma interface web intuitiva, com controle remoto e programação de horários.

### 🎯 Objetivo

Facilitar o cuidado com animais de estimação, permitindo:
- Alimentação automática em horários programados
- Alimentação manual remota a qualquer momento
- Controle da quantidade de ração dispensada
- Interface web simples e intuitiva

## ✨ Funcionalidades

- ✅ **Alimentação Automática**: Configure até 3 horários diferentes para alimentação automática
- ✅ **Alimentação Manual**: Alimente seu pet remotamente através da interface web
- ✅ **Controle de Quantidade**: Configure o tempo de alimentação (quantidade de ração)
- ✅ **Interface Web Responsiva**: Acesse de qualquer dispositivo conectado à mesma rede
- ✅ **Sincronização de Horário**: ESP32 sincroniza automaticamente com servidor NTP
- ✅ **Controle em Tempo Real**: Ativação/desativação do motor via requisições HTTP

## 🏗️ Arquitetura do Sistema

O projeto é composto por três componentes principais:

```
┌─────────────┐      HTTP Requests      ┌─────────────┐      MySQL      ┌─────────────┐
│   ESP32     │ ◄─────────────────────► │  API PHP    │ ◄────────────► │   Database  │
│  (Hardware) │                         │  (Backend)  │                │             │
└─────────────┘                         └─────────────┘                └─────────────┘
       ▲                                       ▲
       │                                       │
       │ HTTP Requests                         │ HTTP Requests
       │                                       │
┌─────────────┐                         ┌─────────────┐
│  Web App    │ ──────────────────────► │  API PHP    │
│  (Frontend) │                         │  (Backend)  │
└─────────────┘                         └─────────────┘
```

### Componentes

1. **ESP32** - Microcontrolador que controla o motor físico
2. **API PHP** - Backend que gerencia dados e comunicação
3. **Web App** - Interface web para configuração e controle
4. **MySQL** - Banco de dados para armazenar configurações

## 📁 Estrutura do Projeto

```
DOGFEEDER/
│
├── ESP32 CODE/
│   └── CODE-1.0/
│       └── CODE-1.0.ino          # Código principal do ESP32
│
├── API PHP/
│   ├── getinfo horarios/
│   │   ├── get.php               # GET - Retorna horários configurados
│   │   ├── indexget.html
│   │   └── scriptget.js
│   │
│   ├── insertinfo horarios/
│   │   ├── insertdata.php        # POST - Insere novos horários
│   │   ├── insethorarios.html
│   │   └── script.js
│   │
│   ├── Motor estados/
│   │   ├── estadomotor.php       # GET - Retorna estado do motor
│   │   ├── postestadomotor.php   # POST - Atualiza estado do motor
│   │   └── indexpost.html
│   │
│   └── tempoalimentacao/
│       ├── gettempoalimentacao.php        # GET - Retorna tempo de alimentação
│       └── inserttempoalimentacao.php     # POST - Insere tempo de alimentação
│
├── SITE/
│   ├── index.html                # Tela inicial
│   ├── Menu.html                 # Menu principal
│   ├── configure.html            # Configuração de horários
│   ├── login.html                # Tela de login
│   ├── account.html              # Conta do usuário
│   ├── script.js                 # JavaScript principal
│   ├── *.css                     # Arquivos de estilo
│   ├── public/                   # Imagens e assets
│   └── package.json              # Dependências Node.js
│
└── DATABASE/
    ├── Banco Horaios alimentacao.txt    # Script SQL - Horários
    └── Banco estado motor.txt           # Script SQL - Estado do motor
```

## 🚀 Instalação e Configuração

### Pré-requisitos

- **ESP32** com suporte WiFi
- **Servidor PHP** (XAMPP, WAMP, ou similar)
- **MySQL** instalado e configurado
- **Arduino IDE** com bibliotecas:
  - WiFi
  - HTTPClient
  - ArduinoJson
- **Node.js** e **npm** (para o site web)

### 1. Configuração do Banco de Dados

Execute os scripts SQL localizados em `DATABASE/`:

```sql
-- Criar banco de dados para horários
CREATE DATABASE IF NOT EXISTS horariosDeAlimentacao;
USE horariosDeAlimentacao;

-- Tabelas para horários
CREATE TABLE Horario1 (
    Id INT PRIMARY KEY AUTO_INCREMENT,
    horaAlimentacao1 INT,
    minutoAlimentacao1 INT
);

CREATE TABLE Horario2 (
    Id INT PRIMARY KEY AUTO_INCREMENT,
    horaAlimentacao2 INT,
    minutoAlimentacao2 INT
);

CREATE TABLE Horario3 (
    Id INT PRIMARY KEY AUTO_INCREMENT,
    horaAlimentacao3 INT,
    minutoAlimentacao3 INT
);

-- Tabela para tempo de alimentação
CREATE TABLE tempo (
    Id INT PRIMARY KEY AUTO_INCREMENT,
    tempoalimentacao INT
);
```

```sql
-- Criar banco de dados para estado do motor
CREATE DATABASE IF NOT EXISTS estado_motor;
USE estado_motor;

CREATE TABLE IF NOT EXISTS liga_desliga (
    id INT AUTO_INCREMENT PRIMARY KEY,
    estado INT
);

-- Inserir valor inicial
INSERT INTO liga_desliga (estado) VALUES (0);
```

### 2. Configuração da API PHP

1. Copie a pasta `API PHP/` para o diretório do seu servidor web (ex: `htdocs/` no XAMPP)
2. Configure as credenciais do MySQL nos arquivos PHP:
   - `servername`: "localhost"
   - `username`: "root" (ou seu usuário MySQL)
   - `password`: "" (ou sua senha MySQL)
   - `dbname`: Nome do banco de dados

3. Ajuste o endereço IP nos arquivos PHP para o IP do seu servidor:
   ```php
   // Exemplo: http://192.168.100.103:8080/dog/...
   ```

### 3. Configuração do ESP32

1. Abra o arquivo `ESP32 CODE/CODE-1.0/CODE-1.0.ino` no Arduino IDE
2. Configure as credenciais WiFi:
   ```cpp
   const char* ssid = "SEU_WIFI_SSID";
   const char* password = "SUA_SENHA_WIFI";
   ```
3. Configure o endereço IP do servidor:
   ```cpp
   const char* serverName1 = "http://SEU_IP:8080/dog/motor/estadomotor";
   const char* serverName2 = "http://SEU_IP:8080/dog/getinfo/get";
   const char* serverName3 = "http://SEU_IP:8080/dog/tempoalimentacao/gettempoalimentacao.php";
   ```
4. Configure o pino do motor (atualmente usando pino 12 como LED):
   ```cpp
   const int ledPin = 12; // Altere para o pino do seu motor
   ```
5. Faça o upload do código para o ESP32

### 4. Configuração do Site Web

1. Navegue até a pasta `SITE/`:
   ```bash
   cd SITE
   ```

2. Instale as dependências:
   ```bash
   npm install
   ```

3. Configure o endereço IP do servidor nos arquivos JavaScript:
   ```javascript
   // Em script.js e configure.html
   // Altere: http://192.168.100.103:8080/dog/...
   // Para: http://SEU_IP:8080/dog/...
   ```

4. Inicie o servidor de desenvolvimento:
   ```bash
   npm start
   ```

   Ou faça o build para produção:
   ```bash
   npm run build
   ```

## 📖 Como Usar

### Configuração Inicial

1. **Acesse a interface web** através do navegador
2. **Configure os horários de alimentação**:
   - Clique em "Register schedules"
   - Configure 3 horários diferentes (em ordem crescente)
   - Clique em "Configure"

3. **Configure o tempo de alimentação**:
   - Na mesma tela, defina o tempo em segundos (máximo 10 segundos)
   - Clique em "Set"

### Alimentação Automática

O ESP32 verifica automaticamente os horários configurados e aciona o motor quando:
- A hora atual coincide com um dos 3 horários programados
- O motor fica ligado pelo tempo configurado

### Alimentação Manual

1. Acesse o menu principal
2. Clique em **"Feed now"**
3. O motor será acionado imediatamente pelo tempo configurado

## 🔌 API Endpoints

### Motor - Estado

#### GET `/dog/motor/estadomotor`
Retorna o estado atual do motor.

**Resposta:**
```json
{
  "id": 1,
  "estado": 0
}
```

#### POST `/dog/motor/postestadomotor`
Atualiza o estado do motor.

**Body:**
```json
{
  "estado": 1
}
```

### Horários de Alimentação

#### GET `/dog/getinfo/get`
Retorna os 3 horários configurados.

**Resposta:**
```json
{
  "Horario1": {
    "hora": 8,
    "minuto": 0
  },
  "Horario2": {
    "hora": 14,
    "minuto": 30
  },
  "Horario3": {
    "hora": 20,
    "minuto": 0
  }
}
```

#### POST `/dog/insertinfo/insertdata`
Insere novos horários de alimentação.

**Body:**
```json
{
  "horario1": {
    "hora": 8,
    "minuto": 0
  },
  "horario2": {
    "hora": 14,
    "minuto": 30
  },
  "horario3": {
    "hora": 20,
    "minuto": 0
  }
}
```

### Tempo de Alimentação

#### GET `/dog/tempoalimentacao/gettempoalimentacao.php`
Retorna o tempo de alimentação configurado (em segundos).

**Resposta:**
```json
[5]
```

#### POST `/dog/tempoalimentacao/inserttempoalimentacao`
Insere o tempo de alimentação.

**Body:**
```json
{
  "tempoalimentacao": 5
}
```

## 🔧 Tecnologias Utilizadas

### Hardware
- **ESP32** - Microcontrolador com WiFi integrado
- **Motor DC** - Para dispensar a ração

### Backend
- **PHP 7+** - Linguagem de programação
- **MySQL** - Banco de dados relacional
- **REST API** - Arquitetura de API

### Frontend
- **HTML5** - Estrutura
- **CSS3** - Estilização
- **JavaScript (ES6+)** - Lógica e interatividade
- **Parcel** - Build tool

### Firmware
- **Arduino IDE** - Ambiente de desenvolvimento
- **ArduinoJson** - Biblioteca para manipulação JSON
- **WiFi Library** - Conexão WiFi
- **HTTPClient** - Requisições HTTP
- **NTP Client** - Sincronização de horário

## 📸 Imagens do Projeto

### Interface Web
![Telas](https://github.com/user-attachments/assets/edb493bb-f179-464c-a0a5-08fc34f72b28)

### Projeto 3D
![Modelo 3D 1](https://github.com/user-attachments/assets/36a538cf-b547-476c-8221-f3f29bc3860a)
![Modelo 3D 2](https://github.com/user-attachments/assets/0f17ccc6-d330-464a-81d8-4eb8112f94ca)

## 🎥 Vídeo Demonstrativo

Assista ao vídeo completo do projeto:
[![Vídeo](https://img.youtube.com/vi/XxYPfcei0gA/0.jpg)](https://www.youtube.com/watch?v=XxYPfcei0gA)

## 🔍 Funcionamento Técnico

### Fluxo de Alimentação Automática

1. ESP32 conecta ao WiFi e sincroniza horário via NTP
2. A cada loop, ESP32 faz requisição GET para obter horários
3. Compara hora/minuto atual com horários configurados
4. Quando há coincidência, aciona o motor
5. Motor fica ligado pelo tempo configurado (alternando HIGH/LOW)
6. Após completar, marca o horário como executado

### Fluxo de Alimentação Manual

1. Usuário clica em "Feed now" na interface web
2. JavaScript envia POST para API atualizando estado do motor para "1"
3. ESP32 faz requisição GET para verificar estado do motor
4. Detecta estado "1" e aciona o motor imediatamente
5. Após completar, ESP32 envia POST para resetar estado para "0"

## ⚙️ Configurações Importantes

### Fuso Horário
O ESP32 está configurado para GMT-3 (Horário de Brasília):
```cpp
const long gmtOffset_sec = -3 * 3600;
```

### Intervalo de Verificação
O ESP32 verifica os horários a cada loop (aproximadamente a cada segundo).

### Tempo Máximo de Alimentação
O tempo de alimentação está limitado a 10 segundos na interface web.

## 🐛 Troubleshooting

### ESP32 não conecta ao WiFi
- Verifique as credenciais WiFi no código
- Certifique-se de que o ESP32 está no alcance da rede
- Verifique se a rede WiFi está funcionando

### API não responde
- Verifique se o servidor PHP está rodando
- Confirme o endereço IP do servidor
- Verifique as configurações de CORS nos arquivos PHP
- Teste os endpoints manualmente com Postman ou curl

### Motor não aciona
- Verifique a conexão física do motor ao ESP32
- Confirme se o pino está correto no código
- Teste o pino com um LED primeiro
- Verifique se o estado está sendo atualizado no banco de dados

### Horários não funcionam
- Verifique se os horários foram salvos no banco de dados
- Confirme se o ESP32 está sincronizado com NTP
- Verifique os logs do Serial Monitor do Arduino IDE

## 📝 Licença

Este projeto está sob a licença ISC.

## 👥 Autores

- **Equipe DOGFEEDER** - Projeto 2024

## 🙏 Agradecimentos

Projeto desenvolvido como solução para automatização da alimentação de animais de estimação.

---


**Esp32 - Dogfeeder - project 2024**

*Take care of those who love you* 🐕❤️

