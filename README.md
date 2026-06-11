# 🅿️ ParkFlow

> Sistema de Gestão de Estacionamento — Back-end .NET + Angular

![.NET](https://img.shields.io/badge/.NET-10.0-512BD4?logo=dotnet)
![Angular](https://img.shields.io/badge/Angular-17+-DD0031?logo=angular)
![Docker](https://img.shields.io/badge/Docker-compose-2496ED?logo=docker)
![License](https://img.shields.io/badge/Licença-MIT-green)

ParkFlow é um sistema completo de gestão de estacionamento composto por duas APIs RESTful em ASP.NET Core e um front-end em Angular. O projeto automatiza o fluxo de entrada e saída de veículos, consulta dados de placa, calcula tarifas por tempo de permanência e registra formas de pagamento.

> 🏛️ Arquitetado com **Clean Architecture**, separação clara de responsabilidades e boas práticas de mercado — do código aos testes e ao pipeline de CI/CD.

-----

## 🏗️ Arquitetura

O sistema é dividido em três camadas independentes:

|Componente           |Tecnologia                            |Responsabilidade                                              |
|---------------------|--------------------------------------|--------------------------------------------------------------|
|`ParkFlow.PlateAPI`  |ASP.NET Core 10 + EF Core + SQLite    |Simula API de consulta de placas (mock da PlacaFipe)          |
|`ParkFlow.ParkingAPI`|ASP.NET Core 10 + EF Core + SQL Server|Gestão de estacionamento, tarifas, vagas e relatórios         |
|`ParkFlow.Web`       |Angular 17+                           |Interface do operador — entrada, saída, pagamento e relatórios|

```
Angular  →  ParkingAPI  →  PlateAPI (mock / real)
```

> 💡 A separação entre as APIs permite substituir a PlateAPI mock pela API real (PlacaFipe ou similar) em produção sem alterar nenhuma outra camada.

-----

## 📁 Estrutura do Repositório

```
parkflow/
├── ParkFlow.PlateAPI/
├── ParkFlow.ParkingAPI/
├── ParkFlow.Web/
├── docker-compose.yml
└── README.md
```

-----

## 🔍 ParkFlow.PlateAPI

Simula uma API de consulta de placas veiculares. Em V1, responde a partir de um banco SQLite pré-populado com dados fictícios.

### Entidade: Vehicle

|Campo            |Tipo      |Descrição                                                 |
|-----------------|----------|----------------------------------------------------------|
|`Id`             |int       |Chave primária                                            |
|`Plate`          |string (7)|Placa no formato Mercosul ou antigo (ex: ABC1234, ABC1D23)|
|`Manufacturer`   |string    |Fabricante (ex: Volkswagen, Toyota)                       |
|`Model`          |string    |Modelo do veículo (ex: Gol, Corolla)                      |
|`Color`          |string    |Cor predominante                                          |
|`ManufactureYear`|int       |Ano de fabricação                                         |

### Endpoints

|Método|Rota                   |Descrição                                                     |
|------|-----------------------|--------------------------------------------------------------|
|`GET` |`/api/vehicles/{plate}`|Consulta veículo pela placa — retorna `200` com dados ou `404`|
|`GET` |`/api/vehicles`        |Lista todos os veículos (dev/debug)                           |

### Exemplo de Resposta

```json
GET /api/vehicles/ABC1234

{
  "id": 1,
  "plate": "ABC1234",
  "manufacturer": "Volkswagen",
  "model": "Gol",
  "color": "Prata",
  "manufactureYear": 2019
}
```

-----

## 🚗 ParkFlow.ParkingAPI

API principal do sistema. Gerencia o fluxo de entrada e saída, calcula tarifas, controla vagas e expõe relatórios operacionais.

### Entidades

<details>
<summary><strong>Operator</strong> — Autenticação</summary>

|Campo         |Tipo    |Descrição                   |
|--------------|--------|----------------------------|
|`Id`          |int     |Chave primária              |
|`Name`        |string  |Nome completo               |
|`Email`       |string  |E-mail de login             |
|`PasswordHash`|string  |Hash da senha (bcrypt)      |
|`Role`        |string  |Perfil: `Admin` / `Operator`|
|`CreatedAt`   |DateTime|Data de criação             |
|`IsActive`    |bool    |Conta ativa ou desativada   |

</details>

<details>
<summary><strong>ParkingSpot</strong> — Vagas</summary>

|Campo       |Tipo  |Descrição                                   |
|------------|------|--------------------------------------------|
|`Id`        |int   |Chave primária                              |
|`Identifier`|string|Identificador legível (ex: A-01)            |
|`IsOccupied`|bool  |Vaga ocupada ou disponível                  |
|`Type`      |string|Tipo: `Standard` / `Disabled` / `Motorcycle`|

</details>

<details>
<summary><strong>ParkingSession</strong> — Sessão de Estadia</summary>

|Campo            |Tipo      |Descrição                             |
|-----------------|----------|--------------------------------------|
|`Id`             |int       |Chave primária                        |
|`Plate`          |string (7)|Placa do veículo                      |
|`Manufacturer`   |string    |Fabricante (auto-preenchido ou manual)|
|`Model`          |string    |Modelo (auto-preenchido ou manual)    |
|`Color`          |string    |Cor (auto-preenchido ou manual)       |
|`ManufactureYear`|int       |Ano de fabricação                     |
|`EntryTime`      |DateTime  |Data/hora de entrada                  |
|`ExitTime`       |DateTime? |Data/hora de saída (null até fechar)  |
|`ParkingSpotId`  |int       |FK para vaga                          |
|`OperatorId`     |int       |FK para operador que registrou        |
|`AmountCharged`  |decimal?  |Valor cobrado ao fechar               |
|`PaymentMethod`  |string?   |`Dinheiro` / `Pix` / `Cartão`         |
|`IsPaid`         |bool      |Pagamento confirmado                  |
|`PlateSource`    |string    |`API` / `Manual`                      |

</details>

<details>
<summary><strong>TariffConfig</strong> — Configuração de Tarifas</summary>

|Campo                 |Tipo    |Descrição                                          |
|----------------------|--------|---------------------------------------------------|
|`Id`                  |int     |Chave primária                                     |
|`FirstHourPrice`      |decimal |Preço da primeira hora (padrão: R$ 25,00)          |
|`AdditionalHourPrice` |decimal |Preço de cada hora adicional (padrão: R$ 10,00)    |
|`FreeMinutes`         |int     |Tolerância de entrada gratuita (padrão: 30 min)    |
|`ExitToleranceMinutes`|int     |Tolerância de saída após fechamento (padrão: 5 min)|
|`UpdatedAt`           |DateTime|Última atualização                                 |
|`UpdatedByOperatorId` |int     |FK para operador que atualizou                     |

</details>

### ⚖️ Regras de Tarifação

- ⏱️ Até **30 minutos** de permanência: **gratuito**
- Acima disso: cobra a **primeira hora completa (R$ 25,00)**
- Cada hora adicional iniciada: **+ R$ 10,00** (arredondamento para cima)
- Após o fechamento: **5 minutos de tolerância** de saída sem nova cobrança

#### Exemplos

|Tempo de Permanência|Horas Cobradas|Valor    |
|--------------------|--------------|---------|
|Até 30 min          |0             |R$ 0,00 ✅|
|31 min — 1h00       |1             |R$ 25,00 |
|1h01 — 2h00         |2             |R$ 35,00 |
|2h01 — 3h00         |3             |R$ 45,00 |
|1h40min             |2             |R$ 35,00 |

### Endpoints

<details>
<summary><strong>🔐 Autenticação</strong></summary>

|Método|Rota              |Descrição                         |
|------|------------------|----------------------------------|
|`POST`|`/api/auth/login` |Login do operador — retorna JWT   |
|`POST`|`/api/auth/logout`|Logout (invalida token no cliente)|

</details>

<details>
<summary><strong>🅿️ Vagas</strong></summary>

|Método |Rota                    |Descrição                          |
|-------|------------------------|-----------------------------------|
|`GET`  |`/api/spots`            |Lista todas as vagas com status    |
|`GET`  |`/api/spots/available`  |Lista apenas vagas disponíveis     |
|`PATCH`|`/api/spots/{id}/status`|Atualiza status manualmente (Admin)|

</details>

<details>
<summary><strong>🚘 Sessões de Estadia</strong></summary>

|Método |Rota                         |Descrição                          |
|-------|-----------------------------|-----------------------------------|
|`POST` |`/api/sessions`              |Registra entrada de veículo        |
|`GET`  |`/api/sessions/active`       |Lista sessões em andamento         |
|`GET`  |`/api/sessions/{id}`         |Detalhe de uma sessão              |
|`POST` |`/api/sessions/{id}/checkout`|Fecha sessão e calcula tarifa      |
|`PATCH`|`/api/sessions/{id}/payment` |Confirma forma de pagamento        |
|`GET`  |`/api/sessions/history`      |Histórico com filtros (data, placa)|

</details>

<details>
<summary><strong>💰 Tarifas</strong></summary>

|Método|Rota         |Descrição                            |
|------|-------------|-------------------------------------|
|`GET` |`/api/tariff`|Retorna configuração atual de tarifas|
|`PUT` |`/api/tariff`|Atualiza tarifas (Admin)             |

</details>

<details>
<summary><strong>📊 Relatórios</strong></summary>

|Método|Rota                    |Descrição                                  |
|------|------------------------|-------------------------------------------|
|`GET` |`/api/reports/cashflow` |Relatório de fluxo de caixa por período    |
|`GET` |`/api/reports/vehicles` |Relatório de veículos atendidos por período|
|`GET` |`/api/reports/occupancy`|Taxa de ocupação por período               |

</details>

-----

## 🖥️ ParkFlow.Web (Angular)

### Telas

|Tela                |Descrição                                                       |
|--------------------|----------------------------------------------------------------|
|🔐 Login             |Autenticação do operador                                        |
|📊 Dashboard         |Visão geral: vagas, sessões ativas, resumo do dia               |
|🚗 Entrada de Veículo|Consulta de placa automática + fallback manual + seleção de vaga|
|📋 Sessões Ativas    |Lista com tempo de permanência em tempo real                    |
|💳 Checkout          |Tempo de estadia, valor calculado e forma de pagamento          |
|🕓 Histórico         |Filtro por data e placa                                         |
|📈 Relatórios        |Fluxo de caixa e volume de veículos                             |
|⚙️ Configurações     |Gerenciamento de tarifas e operadores (Admin)                   |

### Fluxo Principal

```
Login → Dashboard → Registrar Entrada → Sessões Ativas → Checkout → Confirmação de Pagamento
```

-----

## 🗺️ Roadmap

### ✅ V1 — MVP

- [x] Autenticação JWT com perfis `Admin` e `Operator`
- [x] Clean Architecture nas duas APIs
- [x] PlateAPI mock com banco SQLite pré-populado
- [x] Registro de entrada com consulta de placa e fallback manual
- [x] Controle de vagas em tempo real
- [x] Cálculo de tarifa com tolerância de entrada e arredondamento por hora
- [x] Checkout com seleção manual de forma de pagamento
- [x] Tolerância de 5 minutos na saída após fechamento
- [x] Relatórios de fluxo de caixa e histórico de veículos
- [x] FluentValidation em todos os inputs
- [x] Serilog com logs estruturados
- [x] Health Checks (`/health`, `/health/ready`)
- [x] Testes unitários com xUnit
- [x] Testes de integração com WebApplicationFactory
- [x] Documentação de API com Scalar (OpenAPI)
- [x] Docker + docker-compose
- [x] Pipeline CI/CD com GitHub Actions
- [x] Front-end Angular cobrindo todas as telas do operador

### 🔜 V2

- [ ] Substituição da PlateAPI mock pela API real (PlacaFipe)
- [ ] Integração com gateway de pagamento sandbox (Mercado Pago / Stripe)
- [ ] Notificações de tempo limite de permanência

### 💡 Backlog Futuro

- [ ] Configuração dinâmica das regras de tarifação pelo Admin
- [ ] Modelos alternativos de cobrança (diária, mensal, tolerância variável)
- [ ] Dashboard analítico com gráficos de ocupação
- [ ] App mobile para consulta do operador

-----

## 🏛️ Clean Architecture

Ambas as APIs seguem os princípios de Clean Architecture com dependências unidirecionais entre as camadas:

```
ParkFlow.[API]/
├── Domain/          # Entidades, enums, interfaces de repositório
├── Application/     # Use cases, DTOs, validações (FluentValidation)
├── Infrastructure/  # EF Core, repositórios, serviços externos
└── API/             # Controllers, middlewares, configuração
```

> As camadas internas (Domain e Application) não conhecem detalhes de infraestrutura ou framework — apenas contratos e regras de negócio puras.

-----

## 🛠️ Stack

|Camada              |Tecnologia                        |Versão|
|--------------------|----------------------------------|------|
|Runtime             |.NET                              |10.0  |
|Framework Web       |ASP.NET Core                      |10.0  |
|Arquitetura         |Clean Architecture                |—     |
|ORM                 |Entity Framework Core             |10.x  |
|Banco (PlateAPI)    |SQLite                            |—     |
|Banco (ParkingAPI)  |SQL Server                        |2019+ |
|Autenticação        |ASP.NET Core Identity + JWT Bearer|—     |
|Validação           |FluentValidation                  |—     |
|Logs                |Serilog                           |—     |
|Health Checks       |ASP.NET Core Health Checks        |—     |
|Testes unitários    |xUnit                             |—     |
|Testes de integração|xUnit + WebApplicationFactory     |—     |
|Documentação        |Scalar (OpenAPI)                  |—     |
|Front-end           |Angular                           |17+   |
|Estilo              |Angular Material / TailwindCSS    |—     |
|Containerização     |Docker + docker-compose           |—     |
|CI/CD               |GitHub Actions                    |—     |

-----

## 🧪 Testes

O projeto cobre dois níveis de testes:

### Testes Unitários (xUnit)

Validam regras de negócio isoladas, como o cálculo de tarifas e as validações com FluentValidation.

```bash
dotnet test ParkFlow.ParkingAPI.Tests.Unit
dotnet test ParkFlow.PlateAPI.Tests.Unit
```

### Testes de Integração (xUnit + WebApplicationFactory)

Sobem a API em memória e testam os fluxos completos — entrada, checkout e pagamento — contra um banco em memória.

```bash
dotnet test ParkFlow.ParkingAPI.Tests.Integration
```

-----

## 🔁 CI/CD — GitHub Actions

O pipeline roda automaticamente a cada push na `main` ou abertura de Pull Request:

```
Push / PR → Build → Testes Unitários → Testes de Integração → Docker Build
```

O badge no topo do README reflete o status atual do pipeline. ✅

-----

## 📋 Observabilidade

- **Serilog** — logs estruturados em JSON, com sink para console e arquivo
- **Health Checks** — endpoints `/health` e `/health/ready` em ambas as APIs, prontos para uso com Docker e orquestradores

-----

## 🚀 Como Rodar

```bash
# Clone o repositório
git clone https://github.com/seu-usuario/parkflow.git
cd parkflow

# Sobe todos os serviços com Docker
docker-compose up --build
```

|Serviço      |URL                    |
|-------------|-----------------------|
|PlateAPI     |`http://localhost:5001`|
|ParkingAPI   |`http://localhost:5000`|
|Web (Angular)|`http://localhost:4200`|

-----

## 📄 Licença

Este projeto está sob a licença MIT. Veja o arquivo <LICENSE> para mais detalhes.
