# Guia técnico do SUPNFMON

## 1. Objetivo

O `SUPNFMON` é um monitor somente leitura para acompanhar notas fiscais que
utilizam condições de pagamento configuradas para o Mais Vendas. A rotina parte
da nota fiscal do Protheus e associa, quando existente, o registro mais recente
da `SUPPAYINVOICE`.

O desenho preserva duas necessidades importantes:

1. Notas ainda não implantadas continuam visíveis.
2. A filial consultada pode ser alterada na tela sem mudar `cFilAnt` ou o
   ambiente global do usuário.

## 2. Fluxo principal

```text
U_SUPNFMON
    ├── valida empresa e filial
    ├── aplica período inicial de 30 dias
    ├── BuildQuery
    │   ├── restringe empresa + filial
    │   ├── restringe período
    │   ├── valida condição Mais Vendas
    │   └── calcula classe da legenda
    └── ShowBrowse
        ├── pesquisas
        ├── colunas
        ├── legendas
        └── filtros, filial e atualização
```

O browse usa a sequência:

```tlpp
oBrowse:SetDataQuery()
oBrowse:SetAlias(cAlias)
oBrowse:SetQuery(cQuery)
```

Essa ordem é importante para o ciclo de vida do alias em consultas SQL.

## 3. Origem dos dados

| Tabela | Responsabilidade |
|---|---|
| `SF2` | Fonte principal das notas fiscais de saída |
| `SA1` | Nome e CPF/CNPJ do cliente |
| `SD2` | Relação da nota com o pedido de venda |
| `SC5` | Condição de pagamento usada no pedido |
| `SE4` | Identifica condições habilitadas por `E4_XSUPPAY = 'S'` |
| `SUPPAYINVOICE` | Estados e identificadores da implantação Mais Vendas |

A `SUPPAYINVOICE` é associada por empresa/filial, documento e série. O monitor
seleciona o maior `R_E_C_N_O_` quando houver mais de um registro para a mesma
chave funcional.

## 4. Estados apresentados

| Situação funcional | Regra principal | Legenda |
|---|---|---|
| Não implantada | Não existe registro na `SUPPAYINVOICE` | Amarela |
| Cancelada/Excluída | `SUP_STATUS` em `2`, `4` ou `5` | Cinza |
| Erro no faturamento | `SUP_STATUS = 'X'` | Vermelha |
| Aguardando Supplier | `SUP_STATUS = 'T'` | Amarela |
| Aguardando envio | `SUP_STATUS = '0'` | Amarela |
| DDE aguardando entrega | Enviado, exige entrega e ainda não possui data | Amarela |
| Aguardando financeiro | Enviado e `SUP_PROCFI` vazio ou `0` | Amarela |
| Erro no financeiro | Enviado e `SUP_PROCFI = '2'` | Vermelha |
| Implantada | Enviado e `SUP_PROCFI = '1'` | Verde |

Os filtros de situação usam a classe da própria legenda: `O`, `E`, `C` ou `P`.

## 5. Decisões de desempenho

### Período inicial

A abertura usa `Date() - 30` até `Date()`. O usuário pode alterar o intervalo
ou limpar as duas datas para consultar todo o histórico.

### Filtros no banco

Período, filial e situação são incluídos na consulta antes de o resultado ser
entregue ao `FWFormBrowse`. Isso evita carregar uma grande quantidade de linhas
para depois filtrá-las apenas na interface.

### Empresa e filial da integração

O relacionamento passou a comparar diretamente:

```sql
INV.SUP_COMP = empresa + filial
```

Isso elimina `SUBSTR`/`SUBSTRING` sobre a coluna e favorece o uso de índice.

### Restrição da filial

A filial selecionada usa igualdade, além do filtro de autorização retornado por
`FWSQLUsrFilial("SF2")`.

### Próxima medição recomendada

Em ambientes com alto volume, avaliar o plano de execução e um índice secundário
na `SUPPAYINVOICE` com a seguinte ordem aproximada:

```text
SUP_COMP, SUP_DOC, SUP_SERINF, D_E_L_E_T_, R_E_C_N_O_
```

Esse índice não faz parte deste repositório e não deve ser criado em produção
sem comparação de plano, tempo, leituras e impacto nas gravações.

## 6. Organização do fonte

O arquivo foi dividido por responsabilidade:

- Ambiente e versão: `GetCompanyCode`, `GetCurrentBranch`, `GetSourceBuild`.
- Consulta: `BuildQuery`, `GetLegendFilter`.
- Browse: `ShowBrowse`, `BuildSearchConfig`, `BuildBrowseColumns`.
- Grupos de colunas: status, nota, cliente, comercial e integração.
- Interação: `ChangeFilters`, `ChangeBranch`, `RefreshBrowse`.
- Apresentação: título, período, legendas e tradução dos estados.

As funções públicas e auxiliares possuem blocos ProtheusDOC e as variáveis
locais mantêm prefixos de tipo.

## 7. Encoding

O compilador Protheus espera CP1252. O diretório de trabalho deve manter:

- Windows-1252/CP1252.
- Finais de linha CRLF.
- Ausência de BOM.

O `.gitattributes` usa `working-tree-encoding=CP1252`, permitindo que o Git
armazene o blob como UTF-8 para visualização correta no GitHub e gere o arquivo
CP1252 durante o checkout.

Para conferir o arquivo no PowerShell:

```powershell
$path = ".\SUPNFMON.tlpp"
$bytes = [IO.File]::ReadAllBytes($path)
$utf8 = [Text.UTF8Encoding]::new($false, $true)

try {
    $null = $utf8.GetString($bytes)
    "Atenção: arquivo ainda é UTF-8"
} catch {
    "OK: arquivo não é UTF-8; provável CP1252"
}
```

## 8. Publicação no GitHub

### Primeiro pacote na main

```powershell
git switch main
git add .
git commit -m "feat(supnfmon): publicar monitor otimizado"
git push origin main
```

### Desenvolvimento futuro na ajustes

```powershell
git switch ajustes
git pull origin ajustes

# fazer alterações e validar
git add .
git commit -m "perf(supnfmon): otimizar consulta"
git push origin ajustes
```

### Promoção para a main

```powershell
gh pr create --base main --head ajustes --fill
```

Depois da revisão e do teste no Protheus, faça o merge do Pull Request. Em
seguida, sincronize a branch de trabalho:

```powershell
git switch ajustes
git pull origin ajustes
git merge origin/main
git push origin ajustes
```

## 9. Roteiro de compilação e validação

1. Selecionar o servidor e o ambiente corretos no TOTVS Language Server.
2. Confirmar o diretório de includes.
3. Compilar `SUPNFMON.tlpp` no RPO.
4. Confirmar que o título apresenta `v3.5` e uma data de RPO atual.
5. Iniciar o `SIGAMDI` e selecionar empresa/filial.
6. Abrir `U_SUPNFMON` pelo menu.
7. Confirmar somente registros dos últimos 30 dias.
8. Aplicar cada situação disponível na janela `Filtros`.
9. Limpar as datas e testar o histórico.
10. Trocar para uma filial autorizada e confirmar a atualização do título.
11. Testar pesquisas, legendas e botão `Atualizar`.
12. Fechar a rotina e confirmar que não restaram aliases abertos.

## 10. Limites da versão 3.5

- O monitor não executa reprocessamento.
- O monitor não atualiza diretamente a `SUPPAYINVOICE`.
- A criação de índice depende de análise no banco do cliente.
- A compilação e o teste funcional no RPO continuam obrigatórios.
