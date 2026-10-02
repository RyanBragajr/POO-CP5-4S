# Checkpoint 5 — Bug Hunt PetFiap

> Copie este arquivo para a raiz do seu repositório com o nome **README.md**
> e preencha todas as seções.

## Identificação

**Grupo:** ___

| Integrante | RM | Turma |
|---|---|---|
| | | |
| | | |
| | | |
| | | |

| Campo | |
|---|---|
| **Total de bugs corrigidos** | 12 / 12 |
| **Total de ajustes de Clean Code** | 6 / 6 |
| **Total de testes novos escritos** | 6 / 6 |
| **Suíte final (Run As → JUnit Test)** | ___ testes, ___ falhas |

---

## Parte 1 — Bugs encontrados

> Uma linha por bug, na ordem em que você os encontrou. Use a numeração dos seus
> commits (`fix: bug01 ...`). Preencha TODAS as colunas — metade da nota está aqui.

| # | Sintoma observado (o que fiz/vi) | Causa raiz (arquivo e linha aproximada) | Correção aplicada | Conceito da disciplina |
|---|---|---|---|---|
| bug01 | `GeradorProtocoloTest`: `assertSame` falhou e a sequência veio 1, 1, 1 em vez de 1, 2, 3 | `GeradorProtocolo.getInstancia()` — retornava `new GeradorProtocolo()` sem guardar no campo estático `instancia` | Atribuí `instancia = new GeradorProtocolo()` e deixei `getInstancia()`/`proximo()` `synchronized` (o comentário prometia thread-safe) | Padrão Singleton (Aula 14) |
| bug02 | `AtendimentoBuilderTest.deveMontarAtendimentoCompleto`: `expected: <Rex> but was: <null>` | `AtendimentoBuilder.comPet()` — `petNome = petNome;` atribuía o parâmetro a ele mesmo | `this.petNome = petNome;` | `this` e escopo de variáveis (shadowing) |
| bug03 | `deveRecusarMontagemSemNomeDoPet` / `SemPorte`: a exceção esperada não foi lançada | `AtendimentoBuilder.construir()` — não validava nada (o comentário jogava a validação para o controller) | Criei `validarCamposObrigatorios()` chamado no início de `construir()`, lançando `IllegalArgumentException` para nome ou porte ausentes | Builder: objeto só nasce válido; exceções |
| bug04 | `AtendimentoFactoryTest.deveCriarTosaQuandoTipoForTosa`: veio `Banho` em vez de `Tosa` | `AtendimentoFactory.criar()` — `case "TOSA"` instanciava `new Banho(...)` (copia e cola) | `case "TOSA" -> new Tosa(...)` | Padrão Factory; polimorfismo |
| bug05 | `devePreencherOsDadosDoPetNaConsulta`: `getPetNome()` veio `null` | `ConsultaVeterinaria(...)` — o construtor chamava `super()` sem argumentos e descartava os dados (inclusive o status AGENDADO) | `super(protocolo, petNome, petPorte, tutorNome, dataHora)` | Herança e construtores (`super`) |
| bug06 | `AgendaServiceTest.deveRecusarAgendamentoComHorarioJaOcupado`: `HorarioOcupadoException` não lançada, `save` foi chamado | `AgendaService.agendar()` — comparava `petNome` e `dataHora` com `==` (referência), e os objetos eram distintos com valores iguais | `Objects.equals(...)` nas duas comparações | `==` vs `.equals()` (Aula 7) |
| bug07 | `deveLancarExcecaoQuandoAtendimentoNaoExiste`: nada foi lançado, o método devolvia `null` | `AgendaService.buscarPorId()` — `catch (Exception e) { return null; }` engolia `AtendimentoNaoEncontradoException` | Removi o `try/catch`; o `orElseThrow` chega ao chamador (e ao `catch` do controller → 404) | Exceções customizadas unchecked (Aula 11) |
| bug08 | Ao escrever o teste01: banho de porte PEQUENO custava 100 e o GRANDE 60 | `Banho.calcularPreco()` — valores invertidos entre PEQUENO e GRANDE | PEQUENO 60, MEDIO 80, GRANDE 100 (contrato) | Regra de negócio no model; polimorfismo |
| bug09 | Ao escrever o teste02: `Tosa.getDuracaoMinutos()` devolvia 30 em vez de 60 — e compilava sem erro | `Tosa` — declarava `getDuracaoMinutos(String porte)` (sobrecarga, assinatura nova) em vez de sobrescrever `getDuracaoMinutos()` | Troquei pela assinatura sem parâmetro e adicionei `@Override` | Sobrescrita vs sobrecarga (Aula 7) |
| bug10 | Ao escrever o teste03: cancelar um atendimento CONCLUIDO funcionava, sem recusa | `Atendimento.cancelar()` — só fazia `status = "CANCELADO"`, sem checar o status atual | Mesma guarda do `concluir()`: só cancela se AGENDADO, senão `StatusInvalidoException` | Encapsulamento das regras de status no model |
| bug11 | Ao escrever o teste04: agendar para ontem passava (e ainda consultava o banco) | `AgendaService.agendar()` — nenhuma validação de data/hora no passado | Primeira linha do método: se `dataHora` é anterior a `now()`, lança `IllegalArgumentException` antes de chamar o repository | Validação de entrada; falhar cedo |
| bug12 | Só aparece lendo o código (e ao rodar a API): `save()` falharia com "ids for this class must be manually assigned" | `Atendimento` — `@Id` sem `@GeneratedValue`, e nada no código atribui o id | `@GeneratedValue(strategy = GenerationType.AUTO)` | JPA: mapeamento de entidade e chave primária (Aula 13) |

## Parte 2 — Ajustes de Clean Code

| # | Onde estava | Qual princípio/boas práticas era violado | O que eu mudei |
|---|---|---|---|
| clean01 | `AtendimentoFactory.criar(int p, String t, String n, String po, String tu, LocalDateTime d)` | Nomes significativos / revelar a intenção | Renomeei para `protocolo, tipo, petNome, petPorte, tutorNome, dataHora` |
| clean02 | `System.out.println` no construtor do `GeradorProtocolo` e o "Recibo" no `AgendaService.agendar()` | Sem saída de debug em código de produção; método com uma só responsabilidade | Removi os `println`; `agendar()` agora retorna `repository.save(novo)` |
| clean03 | `AtendimentoController.calcularDescontoFidelidade()` privado, nunca chamado, com comentário de "futuro" | Código morto / YAGNI | Removi o método e o bloco de comentário |
| clean04 | Strings `"AGENDADO"`, `"CONCLUIDO"`, `"CANCELADO"` repetidas em `Atendimento` e `AgendaService` | Números/strings mágicos; DRY | Constantes `STATUS_AGENDADO/CONCLUIDO/CANCELADO` em `Atendimento`, usadas nos dois lugares |
| clean05 | `ResponseEntity.status(201)` e `status(409)` no controller | Números mágicos | Troquei por `HttpStatus.CREATED` e `HttpStatus.CONFLICT` |
| clean06 | Literais `"PEQUENO"`/`"MEDIO"` repetidos em `Banho` e `Tosa` | Strings mágicas; DRY | Constantes `PORTE_PEQUENO` e `PORTE_MEDIO` em `Atendimento` |

## Parte 3 — Testes novos (regras que estavam sem cobertura)

> Uma linha por teste novo (`test: ...`). "Regra coberta" é o comportamento do
> contrato (seção 3 do enunciado) que o teste protege. Em "Resultado", diga se o
> teste ficou vermelho ao ser escrito (revelou bug — qual?) ou verde de cara
> (regra já estava correta).

| # | Teste escrito (classe.método) | Regra coberta | Resultado ao escrever (vermelho/verde) |
|---|---|---|---|
| teste01 | `BanhoTest.deveCobrarPrecoPorPorteQuandoForBanho` | Banho: R$ 60 / 80 / 100 por porte | Vermelho — revelou o bug08 (preços invertidos) |
| teste02 | `TosaTest.deveDurar60Minutos` | Tosa dura 60 min | Vermelho — revelou o bug09 (sobrecarga em vez de sobrescrita) |
| teste03 | `AgendaServiceTest.deveRecusarCancelamentoDeAtendimentoJaConcluido` | cancelar() de CONCLUIDO é recusado (`StatusInvalidoException`) e nada é salvo | Vermelho — revelou o bug10 |
| teste04 | `AgendaServiceTest.deveRecusarAgendamentoQuandoDataForNoPassado` | Data/hora no passado → `IllegalArgumentException`, banco nem é consultado | Vermelho — revelou o bug11 |
| teste05 | `AgendaServiceTest.deveCancelarAtendimentoAgendado` | cancelar() de AGENDADO vira CANCELADO | Verde de cara — regra já estava correta |
| teste06 | `AgendaServiceTest.deveRecusarConclusaoDeAtendimentoCancelado` | concluir() de CANCELADO é recusado | Verde de cara — regra já estava correta |

---

## Parte 4 — Perguntas de reflexão

> Responda com suas palavras, 5 a 10 linhas cada, **usando o código real do
> projeto como exemplo**. Respostas genéricas de tutorial não pontuam.

### 1. A suíte como contrato (Aula 15)
O projeto chegou com 20 testes, 9 vermelhos. Descreva como você usou as
mensagens de falha (ex.: `expected: <Rex> but was: <null>`) para caçar os bugs.
O que a suíte de testes tem de melhor do que testar tudo na mão com curl?

### 2. Mock e injeção de dependência (Aulas 13 a 15)
No `AgendaServiceTest`, o `@Mock` cria um `AtendimentoRepository` falso e o
`@InjectMocks` o injeta no service. Explique a relação disso com o `@Autowired`
que o Spring faz em produção — quem "injeta" em cada mundo, e por que o teste
consegue rodar sem banco e sem subir o Spring?

### 3. `==` vs `.equals()` (Aula 7)
Um dos bugs fazia o agendamento duplicado passar pela verificação de conflito.
Explique por que `==` entre Strings e `LocalDateTime` falhou aqui, por que ele
"funciona por sorte" com literais como `"Rex"`, e o que a sua correção mudou.

### 4. Sobrescrita vs sobrecarga (Aula 7)
Um dos bugs compilava sem nenhum erro: um método parecia sobrescrever
`getDuracaoMinutos`, mas na verdade criava uma assinatura nova. Explique a
diferença entre override e overload nesse caso e por que a anotação `@Override`
teria impedido o bug.

### 5. Singleton manual vs bean do Spring (Aula 14)
O `GeradorProtocolo` é um Singleton escrito à mão e causou um dos bugs.
Explique o que ele garante, qual foi o bug, e por que o `AgendaService`
(`@Service`) não corre o mesmo risco no container do Spring.

### 6. Cobertura de testes: onde parar? (Aula 15)
Dos 6 testes novos que você escreveu, alguns ficaram vermelhos (revelaram
bugs) e outros verdes de cara (regras já corretas). Vale a pena manter os que
ficaram verdes? Em um projeto real com prazo, o que você priorizaria testar:
caminho feliz, caminhos de erro, ou 100% de cobertura? Justifique.

---

## Parte 5 — Espaço livre (opcional)

Alguma dificuldade, dúvida ou comentário sobre o checkpoint?

```

```
