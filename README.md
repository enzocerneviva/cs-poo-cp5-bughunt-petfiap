# Checkpoint 5 — Bug Hunt PetFiap

## Identificação

**Grupo:** Devzeiros

| Integrante                | RM     | Turma |
|---------------------------|--------|-------|
| Enzo Cardilli Cerneviva   | 563480 | 2CCPX |
| Matheus Lara Carneiro     | 564049 | 2CCPX |
| Victor Hugo Almeida Bahia | 564633 | 2CCPX |

| Campo | |
|---|---|
| **Total de bugs corrigidos** | 12 / 12 |
| **Total de ajustes de Clean Code** | 6 / 6 |
| **Total de testes novos escritos** | 6 / 6 |
| **Suíte final (Run As → JUnit Test)** | 26 testes, 0 falhas |

---

## Parte 1 — Bugs encontrados

| # | Sintoma observado (o que fiz/vi) | Causa raiz (arquivo e linha aproximada) | Correção aplicada | Conceito da disciplina |
|---|---|---|---|---|
| bug01 | Nenhum teste vermelho. Ao comparar a tabela de preços do contrato com o código, o banho de porte PEQUENO custava R$ 100 e o de porte GRANDE R$ 60 | `Banho.calcularPreco()` (~linha 27–32): os valores de PEQUENO e GRANDE estavam trocados | Inverti os retornos: PEQUENO = 60, MEDIO = 80, GRANDE = 100 | Polimorfismo: a regra de preço mora na subclasse; teste verde não prova ausência de bug |
| bug02 | Nenhum teste vermelho (o repository é mock nos testes). Achado ao ler a entidade: nada gera o id de um atendimento novo, então o `save()` do JPA falharia ao rodar a API | `Atendimento.java` (~linha 14): `@Id` sem `@GeneratedValue` | Adicionei `@GeneratedValue(strategy = GenerationType.IDENTITY)` ao id | Mapeamento objeto-relacional (JPA/Hibernate), chave primária gerada pelo banco |
| bug03 | `deveCriarTosaQuandoTipoForTosa` vermelho: `expected: <Tosa> but was: <Banho>` | `AtendimentoFactory.criar` (~linha 17): `case "TOSA"` instanciava `new Banho(...)` | Troquei por `new Tosa(...)` | Padrão Factory e polimorfismo: o compilador aceita qualquer subclasse de `Atendimento`, o erro é de regra de negócio |
| bug04 | `devePreencherOsDadosDoPetNaConsulta` vermelho: `expected: <Mimi> but was: <null>` | `ConsultaVeterinaria` (~linha 16–18): o construtor chamava `super()` sem argumentos e descartava os dados recebidos | Passei `protocolo, petNome, petPorte, tutorNome, dataHora` para `super(...)` | Construtores e herança: a superclasse só é inicializada com o que o `super(...)` recebe |
| bug05 | Nenhum teste vermelho. Pelo contrato a Tosa dura 60 min, mas o sistema devolvia 30 min | `Tosa.java` (~linha 40): existia `getDuracaoMinutos(String porte)`, que é outro método (sobrecarga) e não sobrescrevia o do pai | Troquei por `@Override public int getDuracaoMinutos()` retornando 60 | Sobrescrita (override) × sobrecarga (overload); a anotação `@Override` evita o erro em tempo de compilação |
| bug06 | `deveLancarExcecaoQuandoAtendimentoNaoExiste` vermelho: a exceção esperada não foi lançada (o método devolvia `null`) | `AgendaService.buscarPorId` (~linha 36–42): `catch (Exception e) { return null; }` engolia a `AtendimentoNaoEncontradoException` do `orElseThrow` | Removi o `try/catch`: o método devolve direto o `orElseThrow(...)` | Tratamento de exceções: não engolir erro, falhar cedo (fail fast); exceção unchecked |
| bug07 | `deveManterUmaUnicaInstancia` e `deveGerarProtocolosSequenciais` vermelhos: instâncias diferentes e `expected: <2> but was: <1>` | `GeradorProtocolo.getInstancia()` (~linha 17–22): criava `new GeradorProtocolo()` e devolvia sem guardar no campo `instancia`; além disso o comentário prometia thread-safe sem `synchronized` | Guardei a instância em `instancia`, e marquei `getInstancia()` e `proximo()` como `synchronized` | Padrão Singleton: construtor privado + campo estático guardando a instância; segurança em concorrência |
| bug08 | `deveMontarAtendimentoCompleto` vermelho: `expected: <Rex> but was: <null>` | `AtendimentoBuilder.comPet` (~linha 24): `petNome = petNome;` atribuía o parâmetro a ele mesmo (faltava `this.`) | Troquei por `this.petNome = petNome;` | Escopo de variáveis e `this`: o parâmetro ocultava o atributo (shadowing) |
| bug09 | `deveRecusarMontagemSemNomeDoPet` e `deveRecusarMontagemSemPorte` vermelhos: `IllegalArgumentException` esperada, nada foi lançado | `AtendimentoBuilder.construir` (~linha 41–43): não validava nada (o comentário jogava a responsabilidade para o controller) | `construir` agora lança `IllegalArgumentException` se o nome do pet ou o porte forem nulos ou em branco | Builder: o objeto só nasce válido; exceções para recusar entrada inválida |
| bug10 | `deveRecusarAgendamentoComHorarioJaOcupado` vermelho: `HorarioOcupadoException` esperada, nada foi lançado | `AgendaService.agendar` (~linha 23): `==` comparava nome do pet e `LocalDateTime` por referência | Troquei por `.equals(...)` nas duas comparações | `==` compara referências, `.equals()` compara valores (Strings, `LocalDateTime`) |
| bug11 | Nenhum teste vermelho. O contrato manda recusar data/hora no passado e o sistema agendava normalmente | `AgendaService.agendar` (~linha 20): nenhuma validação da data | No início de `agendar`, lança `IllegalArgumentException` se `dataHora` for anterior a `LocalDateTime.now()` (antes de consultar o banco) | Validação defensiva e regra de negócio na camada de serviço |
| bug12 | Nenhum teste vermelho. Era possível cancelar um atendimento já CONCLUIDO ou já CANCELADO | `Atendimento.cancelar()` (~linha 63–65): mudava o status sem verificar o estado atual | `cancelar()` só permite a partir de AGENDADO; caso contrário lança `StatusInvalidoException` | Encapsulamento da máquina de estados no model; exceção customizada |

## Parte 2 — Ajustes de Clean Code

| # | Onde estava | Qual princípio/boas práticas era violado | O que eu mudei |
|---|---|---|---|
| clean01 | `AgendaService.agendar`: variáveis `doPet` e `a` | Nomes que revelam intenção | Renomeei para `atendimentosDoPet` e `atendimentoExistente` |
| clean02 | `AtendimentoFactory.criar`: parâmetros `p, t, n, po, tu, d` | Nomes que revelam intenção (quatro `String` seguidas, fácil de trocar sem perceber) | Renomeei para `protocolo, tipo, petNome, petPorte, tutorNome, dataHora` |
| clean03 | `AgendaService.agendar`: `System.out.println("Recibo: ...")` | Responsabilidade única: a regra de agenda não deve escrever no console | Removi o `println` e o método passou a retornar `repository.save(novo)` |
| clean04 | `AtendimentoController`: método privado `calcularDescontoFidelidade` nunca usado, com comentário de "futuro" | Código morto / YAGNI: o histórico fica no git, não no código | Removi o método e o bloco de comentário |
| clean05 | `AtendimentoController`: `@Autowired` direto no campo `service` | Dependências explícitas e imutáveis; classe testável sem o Spring | Troquei por campo `final` com injeção pelo construtor |
| clean06 | `AtendimentoFactory`: strings mágicas `"BANHO"`, `"TOSA"`, `"CONSULTA"` repetidas no `switch` | DRY: a constante já existe em cada subclasse | Passei a usar `Banho.TIPO`, `Tosa.TIPO` e `ConsultaVeterinaria.TIPO` nos `case` |

## Parte 3 — Testes novos (regras que estavam sem cobertura)

| # | Teste escrito (classe.método) | Regra coberta | Resultado ao escrever (vermelho/verde) |
|---|---|---|---|
| teste01 | `BanhoTest.deveCobrarPrecoConformeOPorte` | Preço do banho por porte: PEQUENO R$ 60, MEDIO R$ 80, GRANDE R$ 100 | Vermelho contra o código original: revelou o bug01 (`expected: <60.0> but was: <100.0>`) |
| teste02 | `TosaTest.deveDurar60Minutos` | Duração da tosa: 60 minutos | Vermelho contra o código original: revelou o bug05 (`expected: <60> but was: <30>`) |
| teste03 | `ConsultaVeterinariaTest.deveCobrarPrecoFixoIndependenteDoPorte` | Consulta custa R$ 150 fixo, o porte não muda o preço | Verde de cara: a regra já estava correta |
| teste04 | `AgendaServiceTest.deveRecusarAgendamentoComDataNoPassado` | Agendar com data/hora no passado é recusado com `IllegalArgumentException`, sem consultar nem salvar no banco | Vermelho contra o código original: revelou o bug11 (nenhuma exceção lançada) |
| teste05 | `AgendaServiceTest.deveCancelarAtendimentoAgendado` | `cancelar()` em atendimento AGENDADO vira CANCELADO e é salvo | Verde de cara: a regra já estava correta |
| teste06 | `AgendaServiceTest.deveRecusarCancelamentoDeAtendimentoJaConcluido` | `cancelar()` em atendimento CONCLUIDO é recusado com `StatusInvalidoException` e nada é salvo | Vermelho contra o código original: revelou o bug12 (nenhuma exceção lançada) |

---

## Parte 4 — Perguntas de reflexão

### 1. A suíte como contrato (Aula 15)
Começamos rodando a suíte e lendo cada um dos 9 vermelhos como um sintoma. A parte `expected ... but was ...` apontava direto para a região do código: `expected: <Mimi> but was: <null>` no `devePreencherOsDadosDoPetNaConsulta` nos levou ao construtor da `ConsultaVeterinaria` (o `super()` vazio); `expected: <Tosa> but was: <Banho>` nos levou ao `switch` da factory; `expected: <2> but was: <1>` no `deveGerarProtocolosSequenciais` mostrou que o contador reiniciava a cada chamada, o que nos levou ao Singleton. Cada teste vermelho é uma pergunta precisa, e a causa raiz sempre estava uma ou duas camadas acima do sintoma (model, builder/factory, service). Em comparação com testar tudo na mão com `curl`, a suíte roda em segundos, sem banco e sem rede, é repetível, protege contra regressão (rodamos tudo depois de cada correção) e documenta o contrato em forma de código. O `curl` só exercita um caminho por vez e depende de alguém lembrar de conferir o resultado. Também aprendemos o limite da suíte: 5 dos 12 bugs (preço do Banho, `@GeneratedValue`, duração da Tosa, data no passado e cancelar concluído) não derrubavam nenhum dos 20 testes. Por isso os testes novos importam.

### 2. Mock e injeção de dependência (Aulas 13 a 15)
Em produção, o `AgendaService` declara `@Autowired private AtendimentoRepository repository`, e quem injeta é o contêiner do Spring: ele cria o bean do repository (uma implementação gerada pelo Spring Data, ligada ao banco) e coloca no campo. No `AgendaServiceTest`, o `@Mock` pede ao Mockito que gere uma implementação falsa de `AtendimentoRepository`, que devolve "vazio" (`null`, lista vazia, `Optional.empty()`) até ser programada com `when(...).thenReturn(...)`. O `@InjectMocks` faz `new AgendaService()` e coloca esse dublê no campo `repository`, o mesmo papel do `@Autowired`, só que feito pelo Mockito. O teste roda sem banco e sem subir o Spring porque o service depende só da interface `AtendimentoRepository`, não de uma implementação concreta; qualquer objeto que cumpra a interface serve. É a ideia da injeção de dependência: o service não cria o que usa, ele recebe. O `verify(repository, never()).save(any())` completa o quadro: o teste confere não só o resultado mas também se o service chamou, ou não, o repository.

### 3. `==` vs `.equals()` (Aula 7)
O `==` compara referências (se são o mesmo objeto na memória), e `.equals()` compara valores. Na verificação de conflito do `AgendaService.agendar`, o `LocalDateTime` do atendimento já salvo e o do novo são objetos diferentes mesmo com a mesma data e hora, então `==` dava `false` e o agendamento duplicado passava. No `deveRecusarAgendamentoComHorarioJaOcupado` isso aparece explicitamente: o teste faz `LocalDateTime.parse(existente.getDataHora().toString())`, que cria um outro objeto com o mesmo valor, como acontece no mundo real quando duas requisições chegam com os mesmos dados. Com Strings o `==` "funciona por sorte" quando as duas vêm de literais (como `"Rex"` nos testes), porque literais iguais compartilham o mesmo objeto no *string pool*; mas uma String que vem de um parâmetro de requisição ou do banco é um objeto novo e o `==` falha. A correção foi trocar as duas comparações por `.equals(...)`, que compara o conteúdo.

### 4. Sobrescrita vs sobrecarga (Aula 7)
A `Tosa` tinha `public int getDuracaoMinutos(String porte)`. Como a assinatura tem um parâmetro a mais, o Java a trata como outro método (sobrecarga), e não como uma sobrescrita do `getDuracaoMinutos()` herdado de `Atendimento`. Quando o código chama `atendimento.getDuracaoMinutos()` por uma referência do tipo `Atendimento`, o polimorfismo só enxerga o método sem parâmetros, que devolve 30; os 60 minutos da tosa nunca eram usados. Compilava sem erro e nenhum teste entregue verificava a duração da Tosa. Com `@Override` o compilador teria recusado a classe na hora, porque a anotação é uma promessa de que o método sobrescreve algo do pai. A correção foi remover o parâmetro e adicionar `@Override`.

### 5. Singleton manual vs bean do Spring (Aula 14)
O `GeradorProtocolo` garante, com construtor privado e um campo estático, que exista uma única instância e uma numeração global sequencial dos protocolos. O bug estava em `getInstancia()`: quando `instancia == null` ele criava `new GeradorProtocolo()` e devolvia, mas nunca guardava no campo; assim o campo ficava `null` para sempre, cada chamada criava um objeto novo e o contador recomeçava, gerando sempre protocolo 1. Também havia um segundo problema, o comentário dizia thread-safe, mas sem `synchronized` duas threads poderiam criar duas instâncias ou perder um incremento do contador; marcamos `getInstancia()` e `proximo()` como `synchronized`. O `AgendaService` com `@Service` não corre esse risco porque, no Spring, o escopo padrão do bean é singleton: o contêiner cria o objeto uma vez e injeta a mesma referência onde for pedido. A unicidade é responsabilidade do contêiner, não de um `if` escrito à mão. Ainda assim, o `AgendaService` precisa continuar sem estado mutável compartilhado para ser seguro com várias requisições.

### 6. Cobertura de testes: onde parar? (Aula 15)
Vale manter os que ficaram verdes. `deveCobrarPrecoFixoIndependenteDoPorte` e `deveCancelarAtendimentoAgendado` protegem regras de dinheiro e de estado; se alguém "otimizar" o preço da consulta por porte ou mexer no `cancelar()` no futuro, eles avisam. Verde de cara não significa inútil: significa que a regra está correta hoje e passa a ser vigiada. Em um projeto real com prazo, eu priorizaria as regras de negócio críticas e seus caminhos de erro (preço, status, conflito de horário, data no passado), porque foi exatamente aí que moraram os bugs escondidos deste checkpoint: os 4 testes novos que ficaram vermelhos eram todos de preço, duração, data inválida e transição de status. O caminho feliz entra com um teste básico por regra, e perseguir 100% de cobertura é uma métrica, não um objetivo: ela pode ser alcançada testando getters triviais sem proteger nada que importe.

---

## Parte 5 — Espaço livre (opcional)

Melhorias futuras que identificamos e **não** fizemos, porque estão fora do contrato ou mudariam a assinatura usada pelos testes entregues:

- **Porte sem padronização:** qualquer `String` é aceita como porte (ex.: `"GRANDÃO"`) e, por cair no `else` do cálculo, é cobrada como GRANDE em silêncio. O ideal seria um `enum Porte` com validação, mas isso alteraria assinaturas que os testes entregues usam com `String`.
- **Status como `String`:** `"AGENDADO"`, `"CONCLUIDO"` e `"CANCELADO"` estão espalhados como texto; um `enum` ou constantes evitaria erro de digitação.
- **Protocolo consumido antes da validação:** o controller chama `proximo()` antes de `construir()`, então uma requisição inválida "queima" um número de protocolo e deixa buracos na sequência.
