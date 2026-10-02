# Checkpoint 5 — Bug Hunt PetFiap

> Copie este arquivo para a raiz do seu repositório com o nome **README.md**
> e preencha todas as seções.

## Identificação

**Grupo:** Ryan Amorim e Henrique Mandrick

| Integrante | RM | Turma |
|---|---|---|
| Ryan Amorim de Castro Santana | 564393 | 2CCPW |
| Henrique Mandrick | 562715 | 2CCPW |

| Campo | |
|---|---|
| **Total de bugs corrigidos** | 12 / 12 |
| **Total de ajustes de Clean Code** | 6 / 6 |
| **Total de testes novos escritos** | 6 / 6 |
| **Suíte final (Run As → JUnit Test)** | 26 testes (20 entregues + 6 novos), 0 falhas |

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

**Resposta:**

A suíte de testes funciona como um contrato: ela mostra o que o sistema deve fazer e ajuda a identificar quando o resultado não é o esperado. No teste `deveMontarAtendimentoCompleto`, a mensagem `expected: <Rex> but was: <null>` mostrou que o nome do pet não estava sendo preenchido. Ao investigar, encontrei o problema no `AtendimentoBuilder.comPet`: o código fazia `petNome = petNome`, atribuindo o parâmetro a ele mesmo, em vez de preencher o atributo com `this.petNome = petNome`.

Outro teste retornou 1, 1, 1, quando o esperado era 1, 2, 3. Isso mostrou que o contador estava reiniciando e levou à descoberta do erro no singleton. A vantagem é que os testes rodam em segundos e ficam disponíveis para detectar se esses problemas voltarem, o que chamamos de regressão. Já uma verificação com `curl` depende do banco e exige conferir os resultados manualmente.

### 2. Mock e injeção de dependência (Aulas 13 a 15)
No `AgendaServiceTest`, o `@Mock` cria um `AtendimentoRepository` falso e o
`@InjectMocks` o injeta no service. Explique a relação disso com o `@Autowired`
que o Spring faz em produção — quem "injeta" em cada mundo, e por que o teste
consegue rodar sem banco e sem subir o Spring?

**Resposta:**

O `@Mock` cria uma versão falsa do `AtendimentoRepository`, cujo comportamento posso controlar no teste. O `@InjectMocks` coloca esse repository no campo `repository` do `AgendaService`, permitindo testar o serviço com essa dependência.

Em produção, o Spring faz essa injeção com o `@Autowired`, usando a implementação real. Como o repository é uma interface, posso substituir sua implementação por um mock. Por isso, o teste consegue verificar a lógica do serviço sem precisar de banco de dados nem de iniciar o Spring.

### 3. `==` vs `.equals()` (Aula 7)
Um dos bugs fazia o agendamento duplicado passar pela verificação de conflito.
Explique por que `==` entre Strings e `LocalDateTime` falhou aqui, por que ele
"funciona por sorte" com literais como `"Rex"`, e o que a sua correção mudou.

**Resposta:**

O problema no `AgendaService.agendar()` era comparar `getPetNome()` e `getDataHora()` usando `==`. Para objetos, esse operador verifica se os dois lados apontam para o mesmo objeto na memória, e não se possuem o mesmo conteúdo.

O teste deixou isso claro ao criar outro `LocalDateTime` com `parse`: a data e a hora eram iguais, mas os objetos eram diferentes. Com um literal como `"Rex"`, a comparação podia funcionar por sorte, porque o Java reaproveita Strings iguais no pool. A correção foi usar `Objects.equals`, que compara os valores e também lida com valores nulos.

### 4. Sobrescrita vs sobrecarga (Aula 7)
Um dos bugs compilava sem nenhum erro: um método parecia sobrescrever
`getDuracaoMinutos`, mas na verdade criava uma assinatura nova. Explique a
diferença entre override e overload nesse caso e por que a anotação `@Override`
teria impedido o bug.

**Resposta:**

Na classe `Tosa`, o método `getDuracaoMinutos(String porte)` tinha um parâmetro que não existia no método da superclasse. Por isso, ele criava uma nova assinatura, fazendo uma sobrecarga (overload), em vez de sobrescrever o método original (override).

Assim, quando o método sem parâmetro era chamado, continuava valendo a implementação da superclasse, que retornava 30 minutos. Se tivesse sido usado `@Override` no método com `String porte`, o compilador apontaria o erro, porque não havia um método com aquela assinatura na superclasse para sobrescrever.

### 5. Singleton manual vs bean do Spring (Aula 14)
O `GeradorProtocolo` é um Singleton escrito à mão e causou um dos bugs.
Explique o que ele garante, qual foi o bug, e por que o `AgendaService`
(`@Service`) não corre o mesmo risco no container do Spring.

**Resposta:**

O singleton garante que exista uma única instância do gerador, permitindo manter uma numeração global dos protocolos. O bug estava no `getInstancia()`, que retornava `new GeradorProtocolo()` sem guardar o objeto no campo `instancia`. Assim, cada chamada criava outro gerador, e o contador começava novamente.

No caso do `AgendaService` com `@Service`, o Spring gerencia esse processo: no escopo padrão, ele cria um único bean e o fornece aos pontos que precisam dele. Nesse uso, ninguém precisa chamar `new AgendaService()` manualmente.

### 6. Cobertura de testes: onde parar? (Aula 15)
Dos 6 testes novos que você escreveu, alguns ficaram vermelhos (revelaram
bugs) e outros verdes de cara (regras já corretas). Vale a pena manter os que
ficaram verdes? Em um projeto real com prazo, o que você priorizaria testar:
caminho feliz, caminhos de erro, ou 100% de cobertura? Justifique.

**Resposta:**

Dos meus seis testes, quatro ficaram vermelhos e revelaram os bugs 08 a 11. Os outros dois, `deveCancelarAtendimentoAgendado` e `deveRecusarConclusaoDeAtendimentoCancelado`, ficaram verdes. Mesmo sem encontrar um erro naquele momento, eles são úteis porque protegem comportamentos que já funcionam e avisam se uma alteração causar uma regressão.

Com pouco tempo, eu priorizaria o caminho feliz e os caminhos de erro das regras de negócio, em vez de tentar alcançar 100% de cobertura. No meu caso, o bug10 estava justamente em um caminho de erro, mostrando que testar apenas quando tudo dá certo não seria suficiente.

---

## Parte 5 — Espaço livre (opcional)

**Como a suíte foi validada:** rodei `mvn test` (Maven 3.9.16, JDK 21; o `pom.xml` continua em Java 17). Resultado: 26 testes, 0 falhas, sendo 20 entregues e 6 novos.

**Testes entregues:** nenhum dos 20 foi alterado. Os 6 testes novos foram *adicionados* às classes existentes (`BanhoTest`, `TosaTest` e `AgendaServiceTest`); o diff dessas classes contém apenas linhas inseridas.

**Ordem dos commits:** os 4 testes que ficaram vermelhos (teste01 a teste04) foram commitados *antes* da correção correspondente (bug08 a bug11), para o histórico mostrar o teste falhando e depois a correção. O bug12 (`@Id` sem `@GeneratedValue`) não aparece em nenhum teste: só se descobre lendo a entidade, ou ao tentar `save()` com a API no ar.

**Decisão no bug12:** usei `GenerationType.AUTO` (e não `IDENTITY`) para o mapeamento funcionar tanto no Oracle da FIAP quanto no H2 em memória, sem depender da versão do banco.

**Limitação:** a API não foi executada contra o Oracle da FIAP (as credenciais não foram usadas); a validação foi feita pela suíte unitária, que não usa banco. O `application.properties` permanece com `SEU_RM`/`SUA_SENHA`.

**Observação sobre o aviso do Mockito:** ao rodar os testes aparece um aviso de que o Mockito está se anexando dinamicamente à JVM (*self-attaching*). É apenas um aviso do JDK recente e não afeta o resultado.

**Possível melhoria futura (fora do escopo do checkpoint):** o `GeradorProtocolo` reinicia a contagem a cada execução da aplicação e não é persistido, então os protocolos voltam ao 1 depois de reiniciar a API. Em um sistema real, o número viria do banco (sequence).
