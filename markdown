# Feature Toggle – Documentação do Framework 

## 1. Visão Geral
Este framework de Feature Toggle tem como objetivo central permitir a ativação e desativação de funcionalidades de forma padronizada, automatizada e segura, sem a necessidade de novas implantações de código.

Através dele, é possível controlar o comportamento do sistema em tempo de execução, oferecendo maior flexibilidade para evolução do produto, mitigação de riscos e governança técnica.

O framework foi projetado para ser:

- Centralizado
- Configurável via portal ou serviço
- Independente da lógica de negócio
- Consistente entre ambientes

🎯 Objetivos do Framework

Os principais objetivos deste framework são:
- Permitir ativar ou desativar funcionalidades de forma dinâmica
- Reduzir riscos em deploys e liberações
- Possibilitar rollouts graduais e controlados
- Facilitar testes em produção (feature flags)
- Viabilizar estratégias de kill switch
- Garantir padronização na tomada de decisão das features

---

## 2. Fluxo de Operação

Esta seção descreve o fluxo operacional completo para utilização do framework de Feature Toggle, desde a criação da feature até o consumo no código da aplicação.

O objetivo é garantir que todos os times sigam um processo padronizado, previsível e seguro ao utilizar o framework.<br>

1. **Criação da Feature no Portal.**

O primeiro passo para utilizar o framework é a **criação da feature no portal de gerenciamento de Feature Toggles.**<br>
Nesse momento, o usuário (tech lead responsável pela feature) deve:

- Definir uma chave técnica única (feature_key)
- Configurar o comportamento inicial da feature
- Definir regras e estratégia de ativação
- Informar o ambiente e a aplicação consumidora

A feature não é criada no código, mas sim no portal, garantindo controle centralizado.<br>

2. **Estrutura do JSON de Configuração da feature**

Após definir a chave, o usuário deve criar a configuração da feature no portal no formato JSON, respeitando o padrão do framework.

Esse JSON representa a fonte de verdade da feature.<br>

2.1 **Estrutura do JSON**
```json
{
  "feature_key": "nova_tela_checkout",
  "nome": "Nova Tela de Checkout",
  "descricao": "Habilita a nova experiência de checkout",
  "enabled": true,
  "tipo_toggle": "PERCENTUAL",
  "percentual_ativacao": 30,
  "ambiente": "PRD",
  "aplicacao_servico": "portal-cliente",
  "regras": {
    "region": ["BR"],
    "userType": ["PREMIUM"]
  },
  "kill_switch": false,
  "criado_em": "2025-01-10T10:00:00Z",
  "atualizado_em": "2025-01-15T14:30:00Z"
}

```
---

2.2 **Descrição dos Campos**

2.2.1 **feature_key**:

O feature_key é o identificador técnico único da funcionalidade dentro do Feature Toggle Manager.
É o nome pelo qual o código consulta se a feature está ativa ou não.

O código não conhece o nome da feature, ele conhece apenas o feature_key.

**Características essenciais do feature_key**

**Único (unicidade)**

Não pode existir dois toggles com o mesmo feature_key.

**Imutável (não deve mudar)**

Depois que a feature entra em produção:

**Nunca altere o feature_key**

Por quê?
- Está hardcoded no código
- Mudanças quebram integrações
- Pode gerar bugs silenciosos

Se precisar mudar, crie um novo toggle e descontinue o antigo.

---

2.2.2 **enabled**:

O campo enabled define se a feature está ativa globalmente, considerando ambiente e contexto.
É o interruptor principal da funcionalidade.

Em termos simples

- enabled = true → a feature pode rodar
- enabled = false → a feature não roda, independentemente de qualquer regra

**Regra de ouro do enabled**

Se enabled = false:
- Regras são ignoradas
- Usuários específicos são ignorados

Nada mais importa. 
enabled é o botão liga/desliga global da feature.
Se estiver desligado, a feature simplesmente não existe para o sistema.

---

2.2.3 **tipo_toggle**:

O tipo_toggle define COMO o sistema decide se a funcionalidade estará ativa, depois que o enabled permitir.

- enabled → diz SE a feature pode existir
- tipo_toggle → diz COMO decidir quem usa

Tipos de tipo_toggle

📌 **ON_OFF**

 O que é

A feature funciona apenas pelo valor do enabled.

📄 **Exemplo**

```json
{
  "feature_key": "exportar_relatorio_csv",
  "enabled": true,
  "tipo_toggle": "ON_OFF"
}
```

Comportamento

- enabled = true → todos usam
- enabled = false → ninguém usa

📌 **CONDICIONAL**

O que é

A feature só é ativada se condições específicas forem atendidas.

📄 Exemplo

```json
{
  "feature_key": "desconto_especial",
  "enabled": true,
  "tipo_toggle": "CONDICIONAL",
  "regras": {
    "userType": ["PREMIUM"],
    "region": ["BR"]
  }
}
```

Comportamento

- Usuário PREMIUM no BR → true
- Qualquer outro → false

✅ Boas práticas do tipo_toggle

- Enum fechado (ON_OFF, CONDICIONAL)
- Campo obrigatório
- Validação por tipo
- Nunca decidir no código de negócio

**Resumo**
> tipo_toggle define a estratégia de decisão da feature.<br>
> Ele diz COMO a feature será ligada, não SE ela pode existir.

---

2.2.4 **regras**:

A regra define **QUEM** pode usar a funcionalidade quando o tipo_toggle é CONDICIONAL.

**Relação entre os conceitos**

- enabled → permite a feature existir
- tipo_toggle → define a estratégia
- regras → define para quem a feature está ativa

> “Essa funcionalidade está ligada,<br>
> mas só para usuários que atendem certas condições.”

**Estrutura básica de uma regra**

```json
{
  "userType": ["ADMIN", "PREMIUM"],
  "region": ["BR"]
}
```
---

## 3. Validação e persitência 

Ao salvar o JSON no portal, o framework executa validações automáticas, como:

- Unicidade do feature_key
- Validação do tipo_toggle
- Verificação de campos obrigatórios
- Compatibilidade entre tipo_toggle e regras

Somente após essas validações a feature é persistida no objeto x através de um lambda para consumo do codigo backend.

---

## 4. Consumo da Feature no Código

O consumo de uma Feature Toggle é feito sempre por meio do framework, nunca diretamente pelo Portal Manager ou QuickConfig.

Para avaliar se uma funcionalidade está ativa, o consumidor deve informar:
- feature_key → identificador técnico da feature
- userId → identificador do usuário (funcionário) que está tentando usar a funcionalidade

O framework é responsável por:

- Buscar a configuração da feature
- Avaliar o estado (enabled)
- Aplicar a estratégia (tipo_toggle)
- Considerar regras e segmentações
- Retornar um valor booleano (true ou false)

> O código de negócio não decide nada.<br>
> Ele apenas pergunta ao framework:<br>
> “Essa feature está ativa para esse usuário?”

Exemplo:

```json
Boolean isEnabled = FeatureToggleService.isEnabled(
    'nova_tela_checkout',
    UserInfo.getUserId()
);

if (isEnabled) {
    mostrarNovaTelaCheckout();
} else {
    mostrarTelaAntigaCheckout();
}
```
🔁 Fluxo de avaliação interna

Ao receber a chamada, o framework executa a seguinte sequência:

- Localiza a feature pelo feature_key 
- Verifica se enabled == true <br>
    → se false, retorna false imediatamente 
- Avalia o tipo_toggle <br>
    → se for ON_OFF, o resultado é exclusivamente o valor do enable <br>
    → se for CONDICIONAL o framework aplica as regras configuradas na feature 
- Regras e segmentações com base no userId 

  - **1º verificação por usuário (userId)**
     o framework verifica se o userId informado esta presente no array de usuários permitido na configuracao da feature. <br>
     ```json
        "regras": {
          "users": ["x", "y"]
        }
    ```
    > Se o userId estiver na lista → feature ativada (true)<br>
    > Caso contrário → continua a avaliação

  - **2º verificacao por terrir**
    Se o usuário não estiver explicitamente listado, o framework verifica se o usuario esta em um dos territórios permitidos no array.
    ```json
        "regras": {
          "users": ["x", "y"]
        }
    ```
    > Se o território do usuário estiver na lista → feature ativada (true) <br>
    > Caso contrário → continua a avaliação

  - **3º verificacao por Subterrit**
    Caso não haja correspondência no nível de território, o framework avalia se o usuario esta nos subterritórios associados ao array.
    ```json
        "regras": {
          "users": ["x", "y"]
        }
    ```
    > Se algum subterritório do usuário estiver na lista → feature ativada (true) <br>
    > Caso contrário → feature desativada (false)

---

## 5. Resumo da Avaliação Interna do Framework

Quando o método isEnabled(feature_key) é chamado, o framework executa o seguinte fluxo interno:

- Verifica se a feature existe
- Verifica se enabled == true
- Avalia o tipo_toggle
- Aplica regras, se necessário
- Retorna true ou false

---

## 6. Atualizações e Operação Contínua

Após a criação inicial, o usuário pode:

- Ativar ou desativar a feature
- Ajustar regras
- Alterar usuarios para verem a feature
- Executar rollback imediato

Tudo isso sem novo deploy de código.

---
## 7. Resumo do fluxo

```json
Portal → Criação do JSON
        ↓
Lambda valida e persiste
        ↓
Aplicação consulta via feature_key
        ↓
Framework avalia regras
        ↓
Retorna true / false

```
---

## 8. Diagrama de Arquitetura

🔗 [Clique aqui para visualizar o desenho da arquitetura](http://desenhoArquiteutra)

---


## 9. Estratégia de Cache

O Salesforce atua como **cache centralizado de feature toggles**.

Nenhuma aplicação consome o Portal diretamente. <br>
Todas as decisões de feature são feitas a partir dos dados persistidos
no Salesforce.

---

## 10. Estratégia de Falha (Fail-safe)

| Situação                    | Comportamento                                            
| --------                    | --------------------------------------------------------
| Lambda Falha                | `Último valor válido permanece`
| QuickConfig indisponível    | `Feature mantém estado anterior`
| Toggle não encontrado       | `Feature desativada`   
| JSON inválido               | `Toggle ignorado`         

> Em caso de erro, o sistema **nunca habilita uma feature indevidamente**.<br>
> O comportamento padrão é sempre **feature desligada**.

---

## Governança e Boas Práticas

- Toda feature deve ter um responsável (tech lead).
- Feature antiga deve ser removida após rollout completo.

