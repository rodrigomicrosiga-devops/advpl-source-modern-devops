# ADVPL/TLPP Source - Modern DevOps Core 🤖💻

Este repositório centraliza o desenvolvimento de **códigos customizados (ADVPL e TLPP)** para o ecossistema ERP TOTVS Protheus. Ele opera de forma totalmente desacoplada da infraestrutura de contêineres, sendo governado por uma esteira estrita de **GitOps e Continuous Delivery (CD)** via GitHub Actions.

Aqui, a cultura de abrir a IDE conectada diretamente ao servidor de homologação ou produção está **extinta**. Toda e qualquer alteração de código é auditada por análise estática de qualidade e integrada ao RPO via Pull Requests.

---

## 🏗️ Fluxo de Integração Contínua (GitOps Architecture)

O ciclo de vida do código segue um pipeline de validação elástica que garante risco zero de corrupção do repositório de objetos e blindagem contra códigos mal escritos:

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Desenvolvedor
    participant GH as GitHub (Nuvem)
    participant SQ as TOTVS AppAnalyzer (SonarQube)
    participant SH as Self-Hosted Runner (Servidor)
    participant SM as State Machine (run.sh)
    participant RPO as Custom RPO Engine

    Dev->>GH: Abre Pull Request (PR) com fontes (.prw / .tlpp)
    activate GH
    GH->>SQ: Dispara Análise Estática de Código (Clean Code Check)
    activate SQ
    Note over SQ: Valida queries perigosas, loops<br/>infinitos e funções obsoletas.
    SQ-->>GH: Retorna Selo Verde (Aprovado)
    deactivate SQ

    GH->>SH: Delega ordens de build via HTTPS Seguro
    activate SH
    Note over SH: Descarrega os fontes do PR<br/>na área efêmera /tmp/compile_staging
    
    SH->>SM: Dispara ./run.sh postgres compile
    activate SM
    Note over SM: Desativa temporariamente a malha de runtime<br/>(Core, REST, Telnet) para remover travas de I/O.
    
    SM->>RPO: Invoca container especializado e gera backup preventivo
    activate RPO
    Note over RPO: Compilador CLI nativa processa<br/>a lista dinâmica de lote.
    
    alt Compilação com Erro de Sintaxe
        RPO-->>SM: Retorna Falha
        SM->>RPO: Executa ROLLBACK automático do RPO estável anterior
        SM-->>SH: Sinaliza Erro
        SH-->>GH: Trava o PR com Selo Vermelho
    else Compilação com Sucesso Total
        RPO-->>SM: Retorna Sucesso
        SM->>SM: Destrói container temporário do compilador
        SM->>SM: Reergue a malha elástica de runtime original
        SM-->>SH: Sinaliza Sucesso
        SH-->>GH: Aprova o Merge do Pull Request
    end
    deactivate RPO
    deactivate SM
    deactivate SH
    deactivate GH
```

### 📂 Estrutura do Repositório

O desenvolvedor possui liberdade total para organizar as camadas de negócio por pastas, contanto que os arquivos mantenham as extensões oficiais suportadas pelo `pipeline`:

```plaintext
advpl-source-modern-devops/
│
├── .github/
│   └── workflows/
│       └── gate-compile.yml # Central de inteligência e governança do GitHub Actions
│
├── src/                    # Raiz obrigatória de todos os códigos do ERP
│   ├── faturamento/
│   │   └── u_meu_fonte.prw
│   ├── financeiro/
│   │   └── u_rotina_tlpp.tlpp
│   └── genericos/
│       └── u_genericos_001.tlpp   
│
└── README.md               # Este guia técnico de engenharia de software
```

### 🛡️ Regras de Governança Aplicadas (Gatekeepers)

Nenhum código entra no repositório de objetos sem passar pelas diretrizes automáticas do `Code Analysis`:

`Queries Seguras`: Bloqueio imediato de expressões como SELECT * sem cláusulas restritivas (WHERE) adequadas aos índices do SGBD.

`Isolamento de Credenciais`: Proibição estrita de credenciais de banco ou tokens hardcoded nos fontes.

`Integridade de Memória`: Auditoria sobre desalocação de objetos e fechamento obrigatório de áreas de tabelas locais abertas em runtime.