# Monitor de Notas Mais Vendas

Monitor operacional, desenvolvido em TLPP para o TOTVS Protheus, que apresenta
as notas fiscais elegíveis ao fluxo Mais Vendas e consolida os estados de
faturamento, entrega e processamento financeiro.

O monitor é somente leitura: ele consulta os dados do Protheus e da
`SUPPAYINVOICE`, mas não altera notas fiscais, títulos ou registros da
integração.

## Principais recursos

- Período inicial limitado aos últimos 30 dias para reduzir o tempo de abertura.
- Filtros de período e situação aplicados diretamente na consulta SQL.
- Consulta de todo o histórico quando as datas são deixadas vazias.
- Troca de filial sem alterar a empresa ou a filial global do ambiente.
- Respeito às filiais autorizadas por `FWSQLUsrFilial()`.
- Situação consolidada do faturamento e do financeiro.
- Legendas visuais e descrições textuais dos estados.
- Pesquisas por NF, filial, CPF/CNPJ, implantação, transação e chave da NF-e.
- Exibição da data limite de entrega para operações DDE.
- Identificação da versão e da data do objeto compilado no RPO.

## Estrutura

```text
.
├── SUPNFMON.tlpp
├── README.md
└── docs
    └── GUIA_TECNICO.md
```

- `SUPNFMON.tlpp`: rotina principal `U_SUPNFMON`.
- [`docs/GUIA_TECNICO.md`](docs/GUIA_TECNICO.md): arquitetura, consulta,
  estados, publicação e roteiro de validação.

## Requisitos

- TOTVS Protheus com suporte a TLPP.
- TOTVS Language Server ou TDS para compilação.
- Acesso via `SIGAMDI`, com empresa e filial selecionadas.
- Tabelas padrão `SF2`, `SD2`, `SC5`, `SE4` e `SA1`.
- Tabela de integração `SUPPAYINVOICE` disponível no banco.
- Campo `E4_XSUPPAY` configurado nas condições de pagamento do Mais Vendas.

## Instalação no Protheus

1. Clone o repositório e abra a pasta no VS Code.
2. Configure o servidor Protheus e o diretório de includes no TOTVS Language
   Server.
3. Compile `SUPNFMON.tlpp` no RPO correto.
4. Cadastre `U_SUPNFMON` no menu desejado.
5. Inicie o `SIGAMDI`, selecione empresa e filial e abra o monitor pelo menu.

> O fonte deve permanecer em Windows-1252 (CP1252), com CRLF e sem BOM no
> diretório de trabalho. O `.gitattributes` permite armazenar o conteúdo em
> UTF-8 no GitHub e restaurá-lo como CP1252 no checkout.

## Fluxo de branches

- `main`: versão base ou já validada.
- `ajustes`: desenvolvimento e validação das próximas alterações.

O primeiro pacote funcional é publicado na `main`. Depois disso, novos ajustes
devem ser feitos na branch `ajustes` e promovidos para a `main` por Pull Request.

### Próximas alterações

```powershell
git switch ajustes
git pull origin ajustes

# editar e validar
git add .
git commit -m "perf(supnfmon): descreva o ajuste"
git push origin ajustes
```

Depois da validação, abra o Pull Request:

```powershell
gh pr create --base main --head ajustes --fill
```

Revise o conteúdo e faça o merge somente depois do teste funcional no
Protheus.

## Validação mínima

- Compilar sem erros no RPO.
- Abrir pelo menu do `SIGAMDI`.
- Confirmar período inicial de 30 dias.
- Aplicar cada filtro de situação.
- Limpar as datas e consultar o histórico.
- Trocar para outra filial autorizada.
- Testar as pesquisas do browse.
- Confirmar legendas, atualização e fechamento da tela.

## Situação do projeto

Versão do fonte: `3.5`.

A validação estática e de encoding faz parte do repositório inicial. A
compilação no RPO e o teste funcional precisam ser executados no ambiente
Protheus antes de considerar a versão homologada.
