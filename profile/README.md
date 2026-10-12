# Ecossistema Inventio Lab — implementação, arquitetura e entrega

> Fotografia técnica verificada em **12/10/2026**. Este documento não é um dashboard em tempo real: ele registra o que foi encontrado nas branches principais, na documentação dos projetos, nas execuções do GitHub Actions e em verificações HTTP não destrutivas feitas nessa data.

Este mapa cobre os quatro sistemas mantidos nos três repositórios da organização Inventio Lab:

- [Site Inventio Lab](https://github.com/Inventio-Lab/inventio-lab-site)
- [Comunio CRM](https://github.com/Inventio-Lab/crm-igrejas)
- Comunio SaaS, aplicação independente mantida em `saas-church/` dentro do repositório `crm-igrejas`
- [Igreja Batista Âncora](https://github.com/Inventio-Lab/igreja-batista-ancora)

## Legenda de estado

| Cor | Significado |
|---|---|
| Verde | Publicado e validado por evidência técnica recente |
| Amarelo | Publicado, mas com pendências relevantes conhecidas |
| Azul | Planejado ou próximo passo; não confundir com produção |
| Cinza | Repositório, contexto ou alternativa que não representa o caminho ativo |

Os estados são independentes: **implementado** significa que o caminho existe no código; **validado** indica testes automatizados ou uma verificação operacional documentada; **publicado** indica evidência de deploy e endereço acessível; **verificado operacionalmente** exige uma checagem além da presença do código. Uma resposta HTTP `200` confirma disponibilidade da página, não todos os fluxos autenticados ou integrações.

## 1. Portfólio, maturidade e roadmap

```mermaid
flowchart LR
    siteRepo["Repo inventio-lab-site"] --> siteSystem["Site Inventio Lab"]
    siteSystem --> siteStatus["Código, CI, deploy e HTTP 200"]

    crmRepo["Repo crm-igrejas"] --> crmSystem["Comunio CRM"]
    crmRepo --> saasSystem["Comunio SaaS em saas-church"]
    crmSystem --> crmStatus["Código, CI, AWS verificada e HTTP 200"]
    saasSystem --> saasStatus["Primeira entrega, CI, deploy e HTTP 200"]

    anchorRepo["Repo igreja-batista-ancora"] --> anchorSystem["Igreja Batista Âncora"]
    anchorSystem --> anchorStatus["Código, publicação e HTTP 200"]

    saasStatus -.-> saasRoadmap["Roadmap: financeiro, agenda, escalas, grupos, cobranças e mensagens"]
    anchorStatus -.-> anchorGaps["Pendências: formulário, proteção admin, segredos e CI/CD"]

    classDef context fill:#D9D9D9,stroke:#B3B3B3
    classDef ready fill:#CDF4D3,stroke:#66D575
    classDef caution fill:#FFECBD,stroke:#FFC943
    classDef planned fill:#C2E5FF,stroke:#3DADFF
    class siteRepo,crmRepo,anchorRepo context
    class siteSystem,siteStatus,crmSystem,crmStatus,saasSystem,saasStatus ready
    class anchorSystem,anchorStatus,anchorGaps caution
    class saasRoadmap planned
```

### Leitura do estado

- **Site Inventio Lab:** frontend institucional, formulário real por API/SES, analytics com consentimento, infraestrutura e deploy automatizado. A execução de validação e publicação da `main` foi concluída com sucesso em 10/10/2026.
- **Comunio CRM:** funil comercial, contatos, atividades, contratos, entregas, anexos e importação estão implementados. O endereço está publicado e a verificação de infraestrutura AWS via OIDC foi concluída com sucesso em 10/10/2026.
- **Comunio SaaS:** a primeira entrega cobre membresia, ministérios, importação/exportação, fotos, isolamento por igreja, login Google e autocadastro com Turnstile. O workflow validou e publicou a aplicação em 10/10/2026. Os módulos mostrados como roadmap não fazem parte dessa primeira entrega.
- **Igreja Batista Âncora:** site, conferência, captação, painel, campanhas e integrações serverless existem e o domínio está online. O próprio repositório registra que o contrato do formulário de interesse precisa ser alinhado, que o authorizer JWT da rota administrativa depende de configuração externa, que credenciais Cora ainda são empacotadas no deploy atual e que não há suíte automatizada de testes nem workflow de CI/CD equivalente aos demais sistemas.

## 2. Arquitetura e tecnologias

O diagrama agrupa cada produto como uma unidade de alto nível para permitir a comparação. As caixas de dados representam recursos exclusivos de cada sistema; não há banco ou API compartilhada entre os produtos.

```mermaid
flowchart LR
    subgraph client ["Pessoas e canais"]
        publicUser["Visitantes e contatos"]
        operatorUser["Operadores e administradores"]
    end

    subgraph gateway ["Domínio e entrega"]
        inventioEdge["Inventio: Route 53, ACM, CloudFront e S3"]
        crmEdge["CRM: Route 53, ACM, CloudFront e API Gateway"]
        saasEdge["SaaS: Route 53, ACM, CloudFront e API Gateway"]
        anchorEdge["Âncora: domínio, CloudFront, S3 e API Gateway"]
    end

    subgraph service ["Aplicações e backend"]
        inventioRuntime["Inventio: HTML, CSS, JS, Node 22, Lambda, Terraform e GA"]
        crmRuntime["CRM: React, TypeScript, Vite, Lambda e EventBridge"]
        saasRuntime["SaaS: React, TypeScript, Vite e Lambda ARM64"]
        anchorRuntime["Âncora: React 19, TanStack, Tailwind, Lambda e Scheduler"]
    end

    subgraph datastore ["Dados exclusivos"]
        crmData["CRM: DynamoDB e anexos S3"]
        saasData["SaaS: DynamoDB e fotos S3"]
        anchorData["Âncora: cinco tabelas DynamoDB"]
    end

    subgraph external ["Integrações externas"]
        sesExternal["Amazon SES"]
        identityExternal["Cognito e Google"]
        turnstileExternal["Cloudflare Turnstile"]
        coraExternal["Cora com mTLS"]
        cloudflareAlternative["Cloudflare SSR alternativo não ativo"]
    end

    publicUser -->|"HTTPS"| inventioEdge
    publicUser -->|"HTTPS"| saasEdge
    publicUser -->|"HTTPS"| anchorEdge
    operatorUser -->|"HTTPS"| crmEdge
    operatorUser -->|"HTTPS"| saasEdge
    operatorUser -->|"HTTPS"| anchorEdge

    inventioEdge -->|"Entrega e API"| inventioRuntime
    crmEdge -->|"Entrega e API"| crmRuntime
    saasEdge -->|"Entrega e API"| saasRuntime
    anchorEdge -->|"Entrega e API"| anchorRuntime

    crmRuntime -->|"Lê e grava"| crmData
    saasRuntime -->|"Lê e grava"| saasData
    anchorRuntime -->|"Lê e grava"| anchorData

    inventioRuntime -.->|"Inventio: envia contato"| sesExternal
    crmRuntime -.->|"CRM: autentica"| identityExternal
    saasRuntime -.->|"SaaS: autentica"| identityExternal
    saasRuntime -.->|"SaaS: protege cadastro"| turnstileExternal
    anchorRuntime -.->|"Âncora: autentica"| identityExternal
    anchorRuntime -.->|"Âncora: envia e-mail"| sesExternal
    anchorRuntime -.->|"Âncora: pagamentos preparados"| coraExternal
    anchorRuntime -.->|"Âncora: alvo alternativo"| cloudflareAlternative
```

### Tecnologias transversais

- **AWS:** os quatro sistemas usam hospedagem e/ou serviços serverless na AWS, mas mantêm recursos e dados próprios. Route 53 e ACM sustentam os domínios; CloudFront e S3 entregam os frontends; API Gateway e Lambda atendem os backends.
- **Identidade:** Comunio CRM, Comunio SaaS e o painel da Igreja Âncora usam Cognito com Google. O site institucional da Inventio não exige login.
- **Infraestrutura como código:** Site Inventio, Comunio CRM e Comunio SaaS usam Terraform. Igreja Âncora usa scripts PowerShell/AWS para a publicação principal; a configuração Cloudflare é uma alternativa SSR, não o caminho de produção documentado.
- **Qualidade:** Site Inventio usa testes nativos do Node e Playwright; CRM e SaaS usam TypeScript, Vitest e Playwright; Igreja Âncora documenta lint e build, mas não possui suíte automatizada configurada no `package.json`.

## 3. Caminho de implementação, validação e deploy

```mermaid
flowchart LR
    subgraph siteFlow ["Site Inventio Lab"]
        siteReference["Figma aprovado"] --> siteCode["HTML, CSS, JS e Lambda"]
        siteCode --> siteChecks["Unitários, build e Playwright"]
        siteChecks --> siteInfra["Terraform validado"]
        siteInfra --> siteRelease["GitHub Actions com OIDC"]
        siteRelease --> siteProduction["AWS e inventiolab.com.br"]
        siteProduction --> siteEvidence["Deploy e HTTP 200 verificados"]
    end

    subgraph crmFlow ["Comunio CRM"]
        crmReference["Produto e identidade Comunio"] --> crmCode["React, TypeScript e API"]
        crmCode --> crmChecks["Tipos, Vitest, build e Playwright"]
        crmChecks --> crmInfra["Terraform e estado próprio"]
        crmInfra --> crmRelease["Deploy manual com OIDC"]
        crmRelease --> crmProduction["AWS e comunio.inventiolab.com.br"]
        crmProduction --> crmEvidence["Infra AWS e HTTP 200 verificados"]
    end

    subgraph saasFlow ["Comunio SaaS"]
        saasReference["Protótipo Figma Make"] --> saasCode["Monorepo em saas-church"]
        saasCode --> saasChecks["Tipos, Vitest, build e Playwright"]
        saasChecks --> saasInfra["Terraform e estado exclusivo"]
        saasInfra --> saasRelease["Workflow próprio e OIDC"]
        saasRelease --> saasProduction["AWS e saas.inventiolab.com.br"]
        saasProduction --> saasEvidence["Deploy e HTTP 200 verificados"]
    end

    subgraph anchorFlow ["Igreja Batista Âncora"]
        anchorReference["Design institucional"] --> anchorCode["React, TanStack e Lambda"]
        anchorCode --> anchorChecks["Lint e build manual"]
        anchorChecks --> anchorInfra["Scripts PowerShell e AWS CLI"]
        anchorInfra --> anchorProduction["AWS e ibancora.com.br"]
        anchorProduction --> anchorEvidence["HTTP 200, com lacunas conhecidas"]
    end

    subgraph roadmapFlow ["Próximas etapas separadas da produção"]
        saasNext["SaaS: módulos futuros"] --> saasPlan["Financeiro, agenda, escalas, grupos, cobranças e mensagens"]
        anchorNext["Âncora: endurecimento"] --> anchorPlan["Contrato do formulário, JWT, segredos, testes e CI/CD"]
    end

    saasEvidence -.-> saasNext
    anchorEvidence -.-> anchorNext

    style siteFlow fill:#E6F4EA,stroke:#66D575
    style crmFlow fill:#E6F4EA,stroke:#66D575
    style saasFlow fill:#E6F4EA,stroke:#66D575
    style anchorFlow fill:#FFECBD,stroke:#FFC943
    style roadmapFlow fill:#C2E5FF,stroke:#3DADFF
    style siteEvidence fill:#CDF4D3,stroke:#66D575
    style crmEvidence fill:#CDF4D3,stroke:#66D575
    style saasEvidence fill:#CDF4D3,stroke:#66D575
    style anchorEvidence fill:#FFECBD,stroke:#FFC943
    style saasNext fill:#C2E5FF,stroke:#3DADFF
    style saasPlan fill:#C2E5FF,stroke:#3DADFF
    style anchorNext fill:#C2E5FF,stroke:#3DADFF
    style anchorPlan fill:#C2E5FF,stroke:#3DADFF
```

## Evidências da fotografia

| Sistema | Produção verificada | Evidência de entrega ou operação | Limite da evidência |
|---|---|---|---|
| Site Inventio Lab | [inventiolab.com.br](https://inventiolab.com.br) — HTTP 200 em 12/10/2026 | [GitHub Actions: validação e deploy aprovados em 10/10/2026](https://github.com/Inventio-Lab/inventio-lab-site/actions/runs/38074416700) | A checagem HTTP não envia o formulário |
| Comunio CRM | [comunio.inventiolab.com.br](https://comunio.inventiolab.com.br) — HTTP 200 em 12/10/2026 | [GitHub Actions: verificação AWS aprovada em 10/10/2026](https://github.com/Inventio-Lab/crm-igrejas/actions/runs/38070445516) e documentação de implantação do repositório | A sessão autenticada não foi reexecutada nesta fotografia |
| Comunio SaaS | [saas.inventiolab.com.br](https://saas.inventiolab.com.br) — HTTP 200 em 12/10/2026 | [GitHub Actions: validação e deploy aprovados em 10/10/2026](https://github.com/Inventio-Lab/crm-igrejas/actions/runs/38070452246) | A checagem HTTP não cria membros nem igrejas |
| Igreja Batista Âncora | [ibancora.com.br](https://ibancora.com.br) — HTTP 200 em 12/10/2026 | README, scripts e histórico da branch `main` do [repositório](https://github.com/Inventio-Lab/igreja-batista-ancora) | Não há workflow de CI equivalente; fluxos com dados pessoais e pagamentos não foram acionados |

## Regras para atualizar este documento

1. Atualize a data da fotografia e preserve a distinção entre código, validação, publicação e operação.
2. Use apenas evidências da branch principal, documentação vigente, execução de CI/CD ou verificação operacional segura.
3. Não marque roadmap como entregue e não infira integração entre produtos pelo simples uso dos mesmos serviços AWS.
4. Não inclua IDs de conta, ARNs, segredos, tokens, certificados, dados pessoais ou URLs internas sensíveis.
5. Ao mudar um diagrama, atualize a versão correspondente no FigJam para manter as duas fontes alinhadas.
