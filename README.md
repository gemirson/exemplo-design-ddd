# exemplo-design-ddd
Responsável por fornecer uma camada solida para design com DDD

## Assunto: PROJETO - Extensão da Entidade Carteira para Operações Pré e Pós-Fixadas
Para: Time de Desenvolvimento
De: Engenheiro Linus
Data: 21 de junho de 2025

Colegas,

Recebemos o requisito de evoluir nossa entidade Carteira para suportar dois tipos fundamentalmente diferentes de contratos de parcelamento: pré-fixado e pós-fixado. A principal complexidade é que cada tipo de parcela possui atributos e lógicas de cálculo de valor distintos.

Uma abordagem ingênua nos levaria a poluir nossas classes com condicionais (if/else), violando o OCP e tornando o código frágil. A seguir, apresento um design que utiliza polimorfismo para criar um modelo robusto, coeso e, mais importante, aberto a futuras extensões (como um sistema híbrido, por exemplo) e fechado para modificações em seu comportamento principal.

O Design Proposto: Herança e Polimorfismo
A estratégia central é tratar Parcela como um conceito abstrato. Uma Carteira conterá uma lista de Parcelas, mas não precisará saber o tipo concreto de cada uma. Ela simplesmente confiará que cada parcela sabe como se comportar (neste caso, como calcular seu próprio valor atualizado).

### 1. A Entidade Abstrata: Parcela
Criaremos uma classe abstract que define o contrato comum a todas as parcelas.

Atributos Comuns: Toda parcela tem um número, uma data de vencimento e um status.
Comportamento Abstrato: O comportamento que varia é o cálculo do valor. Definiremos um método abstract getValorAtualizado().
<!-- end list -->

```java

// Parcela.java
import java.math.BigDecimal;
import java.time.LocalDate;

public abstract class Parcela {
    protected int numero;
    protected LocalDate dataVencimento;
    protected StatusParcela status;
    // Referência à raiz do agregado, se necessário
    // protected Carteira carteira;

    public Parcela(int numero, LocalDate dataVencimento) {
        this.numero = numero;
        this.dataVencimento = dataVencimento;
        this.status = StatusParcela.ABERTA;
    }

    // O método polimórfico. A "mágica" do OCP acontece aqui.
    // A Carteira chamará este método sem saber como ele é implementado.
    public abstract BigDecimal getValorAtualizado(LocalDate dataCalculo);

    public void pagar(BigDecimal valorPago) {
        // Lógica de pagamento...
        this.status = StatusParcela.PAGA;
    }

    // Getters...
    public int getNumero() { return numero; }
    public LocalDate getDataVencimento() { return dataVencimento; }
    public StatusParcela getStatus() { return status; }
}

enum StatusParcela { ABERTA, PAGA, VENCIDA; }

```

### 2. As Implementações Concretas
Agora, criamos as subclasses. Cada uma contém apenas os atributos e a lógica relevantes para seu contexto.

#### a) ParcelaPreFixada

Possui um valor fixo, definido no momento do contrato.

```java

// ParcelaPreFixada.java
import java.math.BigDecimal;
import java.time.LocalDate;

public class ParcelaPreFixada extends Parcela {
    private final BigDecimal valorFixo; // Atributo específico!

    public ParcelaPreFixada(int numero, LocalDate dataVencimento, BigDecimal valorFixo) {
        super(numero, dataVencimento);
        this.valorFixo = valorFixo;
    }

    @Override
    public BigDecimal getValorAtualizado(LocalDate dataCalculo) {
        // A lógica é simples: retorna o valor fixo.
        // Poderíamos adicionar juros de mora se dataCalculo > dataVencimento.
        return this.valorFixo;
    }
}
```

##### b) ParcelaPosFixada

Seu valor depende de um índice econômico e é calculado "a posteriori".

```java

    // ParcelaPosFixada.java
    import java.math.BigDecimal;
    import java.time.LocalDate;

    public class ParcelaPosFixada extends Parcela {
        private final BigDecimal valorBase; // Atributos específicos!
        private final IndiceCorrecao indice;
        
        // Simulação de um serviço externo que busca o valor do índice
        private final ServicoIndiceEconomico servicoIndice;

        public ParcelaPosFixada(int numero, LocalDate dataVencimento, BigDecimal valorBase, IndiceCorrecao indice, ServicoIndiceEconomico servicoIndice) {
            super(numero, dataVencimento);
            this.valorBase = valorBase;
            this.indice = indice;
            this.servicoIndice = servicoIndice;
        }

        @Override
        public BigDecimal getValorAtualizado(LocalDate dataCalculo) {
            // Lógica de cálculo específica para pós-fixado.
            BigDecimal fatorCorrecao = servicoIndice.buscarFator(this.indice, dataVencimento);
            return this.valorBase.multiply(fatorCorrecao);
        }
    }

    // Classes de suporte para o exemplo
    enum IndiceCorrecao { CDI, IPCA; }
    interface ServicoIndiceEconomico {
        BigDecimal buscarFator(IndiceCorrecao indice, LocalDate data);
    }

```
### 3. A Raiz do Agregado: Carteira

A Carteira é a nossa Raiz de Agregado. Ela gerencia o ciclo de vida de suas parcelas e expõe os comportamentos de negócio. Notem como ela opera sobre a abstração Parcela, em vez das implementações concretas.

```java

    // Carteira.java
    import java.math.BigDecimal;
    import java.time.LocalDate;
    import java.util.ArrayList;
    import java.util.List;
    import java.util.stream.Collectors;

    public class Carteira {
        private final Guid id;
        private final List<Parcela> parcelas;

        public Carteira() {
            this.id = Guid.randomUUID();
            this.parcelas = new ArrayList<>();
        }

        // MÉTODO DE FÁBRICA: Encapsula a criação de parcelas pré-fixadas
        public void contratarOperacaoPreFixada(BigDecimal valorTotal, int totalParcelas, LocalDate dataPrimeiroVencimento) {
            if (!this.parcelas.isEmpty()) {
                throw new IllegalStateException("Carteira já possui uma operação contratada.");
            }
            BigDecimal valorParcela = valorTotal.divide(new BigDecimal(totalParcelas), 2, RoundingMode.HALF_UP);
            for (int i = 1; i <= totalParcelas; i++) {
                LocalDate vencimento = dataPrimeiroVencimento.plusMonths(i - 1);
                this.parcelas.add(new ParcelaPreFixada(i, vencimento, valorParcela));
            }
        }
        
        // MÉTODO DE FÁBRICA: Encapsula a criação de parcelas pós-fixadas
        public void contratarOperacaoPosFixada(BigDecimal valorBase, int totalParcelas, IndiceCorrecao indice, LocalDate dataPrimeiroVencimento, ServicoIndiceEconomico servicoIndice) {
            if (!this.parcelas.isEmpty()) {
                throw new IllegalStateException("Carteira já possui uma operação contratada.");
            }
            for (int i = 1; i <= totalParcelas; i++) {
                LocalDate vencimento = dataPrimeiroVencimento.plusMonths(i - 1);
                this.parcelas.add(new ParcelaPosFixada(i, vencimento, valorBase, indice, servicoIndice));
            }
        }

        // *** O CORAÇÃO DO OCP NA PRÁTICA ***
        // Este método funciona para QUALQUER tipo de parcela, hoje e no futuro.
        // Ele está FECHADO para modificação. Não precisamos alterá-lo se um
        // novo tipo de parcela (ex: ParcelaHibrida) for criado.
        public BigDecimal getSaldoDevedorTotal(LocalDate dataCalculo) {
            return this.parcelas.stream()
                    .filter(p -> p.getStatus() == StatusParcela.ABERTA)
                    .map(p -> p.getValorAtualizado(dataCalculo)) // Polimorfismo em ação!
                    .reduce(BigDecimal.ZERO, BigDecimal::add);
        }
        
        public List<Parcela> getParcelas() {
            return List.copyOf(this.parcelas); // Retorna cópia imutável
        }
    }

```

Análise do Design e Conformidade com o OCP
Fechada para Modificação: A classe Carteira é estável. Seu método principal, getSaldoDevedorTotal, não precisa ser alterado para suportar novos tipos de parcelamento. Ele delega o cálculo para os objetos Parcela, que são os especialistas.
Aberta para Extensão: Se o negócio criar um "Parcelamento Híbrido", nós simplesmente criamos uma nova classe ParcelaHibrida extends Parcela, implementamos sua lógica de cálculo e adicionamos um novo método de fábrica na Carteira (contratarOperacaoHibrida(...)). O código existente e testado permanece intocado.
Este design encapsula a complexidade onde ela pertence, promove alta coesão em cada classe e mantém um baixo acoplamento entre a Carteira e os detalhes de suas parcelas. É um modelo de domínio rico, resiliente e alinhado aos princípios que prezamos.


## Assunto: EVOLUÇÃO DO DESIGN - Estratégias de Amortização para Componentes da Parcela
  Para: Time de Desenvolvimento
  De: Engenheiro Linus
  Data: 21 de junho de 2025
  
  Colegas,
  
  Dando continuidade ao nosso modelo de Carteira e Parcela, enfrentamos um novo desafio de design: a regra de amortização de um pagamento dentro de uma única parcela pode variar. Um contrato pode estipular que o pagamento quite primeiro os juros, enquanto outro pode priorizar o principal.
  
  Inserir essa lógica com if/else dentro do método pagar() da Parcela seria um erro grave de design. A Parcela não deve ser responsável por conhecer todos os algoritmos de amortização possíveis. Ela deve, isso sim, delegar esse comportamento.
  
  A solução aqui é o Padrão Strategy. Usaremos o Strategy para encapsular o algoritmo de amortização.
  
  O Design Proposto: Composição de Componentes e o Padrão Strategy
  Evoluir a Parcela: Uma parcela não é mais um valor monolítico. Ela é uma composição de diferentes componentes financeiros.
  Criar IAmortizacaoStrategy: Uma interface que define o contrato para um algoritmo de amortização.
  Implementar Estratégias Concretas: Criaremos classes que implementam a interface para cada regra de negócio (JurosPrimeiroStrategy, PrincipalPrimeiroStrategy, etc.).
  Orquestrar na Carteira: A Carteira, como raiz do agregado, decidirá qual estratégia usar ao aplicar um pagamento.

 ## 1. Evoluindo a Parcela para ter Componentes

  Primeiro, tornamos explícito que uma parcela é composta por partes. Criaremos uma classe ComponenteFinanceiro.
  
 ```java
  
  // ComponenteFinanceiro.java
  import java.math.BigDecimal;
  
  public class ComponenteFinanceiro {
      private final TipoComponente tipo;
      private final BigDecimal valorOriginal;
      private BigDecimal saldoDevedor;
  
      public ComponenteFinanceiro(TipoComponente tipo, BigDecimal valorOriginal) {
          this.tipo = tipo;
          this.valorOriginal = valorOriginal;
          this.saldoDevedor = valorOriginal;
      }
  
      public BigDecimal amortizar(BigDecimal valor) {
          BigDecimal valorAmortizado = valor.min(this.saldoDevedor);
          this.saldoDevedor = this.saldoDevedor.subtract(valorAmortizado);
          return valor.subtract(valorAmortizado); // Retorna o valor restante do pagamento
      }
      
      // Getters
      public TipoComponente getTipo() { return tipo; }
      public BigDecimal getSaldoDevedor() { return saldoDevedor; }
  }
  
  enum TipoComponente { PRINCIPAL, JUROS, MULTA, TAXA; }

  ```

  Agora, a classe Parcela (e suas filhas) conterá uma lista desses componentes. O método getValorAtualizado irá somar o saldo devedor de seus componentes.
  
 ```java
  
  // Parcela.java (Abstrata, agora com componentes)
  import java.math.BigDecimal;
  import java.time.LocalDate;
  import java.util.List;
  import java.util.stream.Collectors;
  
  public abstract class Parcela {
      // ... numero, dataVencimento, status ...
      protected final List<ComponenteFinanceiro> componentes;
  
      public Parcela(int numero, LocalDate dataVencimento, List<ComponenteFinanceiro> componentes) {
          // ... construtor ...
          this.componentes = componentes;
      }
  
      // O método de cálculo agora soma os saldos dos componentes
      public BigDecimal getValorAtualizado() {
          return componentes.stream()
                  .map(ComponenteFinanceiro::getSaldoDevedor)
                  .reduce(BigDecimal.ZERO, BigDecimal::add);
      }
  
      // O método de negócio para pagar/amortizar agora DELEGA para uma estratégia
      public void aplicarPagamento(BigDecimal valorPago, IAmortizacaoStrategy estrategia) {
          if (valorPago.compareTo(BigDecimal.ZERO) <= 0) {
              return; // Não faz nada com pagamento zerado ou negativo
          }
          
          // A Parcela não sabe COMO amortizar, ela confia na estratégia injetada.
          estrategia.aplicar(this.componentes, valorPago);
  
          // Atualiza o status geral da parcela se o saldo for zerado
          if (getValorAtualizado().compareTo(BigDecimal.ZERO) == 0) {
              this.status = StatusParcela.PAGA;
          }
      }
  
      // Getters...
  }

  ```

  ## 2. Definindo a Interface da Estratégia

  Esta interface define o contrato para qualquer algoritmo de amortização.
  
```java  
    // IAmortizacaoStrategy.java
    import java.math.BigDecimal;
    import java.util.List;
    
    public interface IAmortizacaoStrategy {
        void aplicar(List<ComponenteFinanceiro> componentes, BigDecimal valorPago);
    }
  ```
  ## 3. Implementando as Estratégias Concretas
  Aqui está a beleza do padrão. Cada regra complexa é isolada em sua própria classe.
  
  ### a) ParcialStrategy
  
  Primeiro quita juros e multa, depois o principal.
  
   
```java 
  
    // ParcialStrategy.java
    import java.math.BigDecimal;
    import java.util.Comparator;
    import java.util.List;
    import java.util.Map;
    import java.util.function.Function;
    import java.util.stream.Collectors;
    
    public class ParcialStrategy implements IAmortizacaoStrategy {
        // Define a ordem de prioridade para pagamento
        private static final List<TipoComponente> ORDEM_PAGAMENTO = 
            List.of(TipoComponente.MULTA, TipoComponente.JUROS, TipoComponente.TAXA, TipoComponente.PRINCIPAL);
    
        @Override
        public void aplicar(List<ComponenteFinanceiro> componentes, BigDecimal valorPago) {
            BigDecimal valorRestante = valorPago;
    
            // Ordena os componentes pela prioridade definida
            Map<TipoComponente, ComponenteFinanceiro> mapaComponentes = componentes.stream()
                .collect(Collectors.toMap(ComponenteFinanceiro::getTipo, Function.identity()));
    
            for (TipoComponente tipo : ORDEM_PAGAMENTO) {
                if (valorRestante.compareTo(BigDecimal.ZERO) <= 0) break;
                
                ComponenteFinanceiro componente = mapaComponentes.get(tipo);
                if (componente != null) {
                    valorRestante = componente.amortizar(valorRestante);
                }
            }
        }
    }
  ```
  ### b) IntegralStrategy
  
  Implementação inversa.
  
  ```java 
  
    // IntegralStrategy.java
    public class IntegralStrategy implements IAmortizacaoStrategy {
        private static final List<TipoComponente> ORDEM_PAGAMENTO = 
            List.of(TipoComponente.PRINCIPAL, TipoComponente.JUROS, TipoComponente.MULTA, TipoComponente.TAXA);
    
        // A implementação do método aplicar() seria idêntica à anterior,
        // apenas usando esta nova ORDEM_PAGAMENTO.
        @Override
        public void aplicar(List<ComponenteFinanceiro> componentes, BigDecimal valorPago) {
            // ... lógica idêntica à JurosPrimeiroStrategy, mas com a ordem invertida ...
        }
    }

 ```
 
 ## 4. Orquestrando na Carteira
  A Carteira agora tem a responsabilidade de conhecer a política do contrato e aplicar a estratégia correta.
  
   ```java 
  
    // Carteira.java (trecho modificado)
    public class Carteira {
        // ... id, lista de parcelas ...
        private final IAmortizacaoStrategy estrategiaDeAmortizacao; // A política do contrato!
    
        // O construtor agora define a política para toda a carteira
        public Carteira(IAmortizacaoStrategy estrategiaDeAmortizacao) {
            this.id = Guid.randomUUID();
            this.parcelas = new ArrayList<>();
            this.estrategiaDeAmortizacao = estrategiaDeAmortizacao;
        }
    
        // Método de negócio que recebe um pagamento para uma parcela específica
        public void receberPagamento(int numeroParcela, BigDecimal valorPago) {
            Parcela parcelaAlvo = this.parcelas.stream()
                .filter(p -> p.getNumero() == numeroParcela && p.getStatus() == StatusParcela.ABERTA)
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("Parcela não encontrada ou já paga."));
    
            // A Carteira injeta a estratégia de amortização correta na Parcela.
            parcelaAlvo.aplicarPagamento(valorPago, this.estrategiaDeAmortizacao);
        }
        
        // Os métodos de fábrica (contratarOperacao...) agora criariam as parcelas
        // com seus respectivos componentes de principal e juros.
        // ...
    }

  ```

  Conclusão e Próximos Passos
  Com este design, alcançamos um novo nível de flexibilidade:
  
  Herança para "O que é?": ParcelaPreFixada e ParcelaPosFixada ainda modelam as diferenças de natureza e cálculo de juros de cada tipo de contrato.
  Strategy para "Como se comporta?": IAmortizacaoStrategy modela as diferenças nos algoritmos de pagamento, independentemente do tipo de parcela.
  Nosso modelo de domínio está mais rico e muito mais resiliente. Se amanhã o negócio inventar uma "Amortização Proporcional", nós criamos a classe ProporcionalStrategy, instanciamos a Carteira com ela e nenhuma outra parte do nosso código precisa ser modificada.
  
  Este é o poder de combinar padrões para resolver problemas complexos, mantendo nosso código limpo, coeso e fiel ao Princípio Aberto/Fechado.

## Assunto: IMPLEMENTAÇÃO - Validação de Componentes com o Padrão Resultado/Erro
  Para: Time de Desenvolvimento
  De: Engenheiro Linus
  Data: 21 de junho de 2025
  
  Colegas,
  
  Identificamos um requisito crítico: nenhuma amortização deve ser realizada se qualquer um dos componentes da parcela for inválido. A questão é: como modelar essa validação de forma limpa, segura e que evite o uso de exceções para controle de fluxo?
  
  A resposta é adotar o padrão Resultado/Erro. Este padrão funcional nos força a lidar tanto com o caminho de sucesso quanto com o de falha, tornando nosso código mais explícito e seguro.
  
  O Design Proposto: Railway-Oriented Programming
  A ideia é que qualquer operação que possa falhar devido a uma regra de negócio não retorne o valor diretamente nem lance uma exceção. Em vez disso, ela retorna um objeto Resultado que encapsula um dos dois estados possíveis: Sucesso (contendo o valor) ou Erro (contendo os detalhes da falha).
  
  1. A Ferramenta Fundamental: A Classe Resultado
  Primeiro, definimos nossa ferramenta genérica. Esta classe é o pilar do padrão.
  
  ```java 
  
    // Resultado.java
    import java.util.List;
    import java.util.Optional;
    import java.util.function.Consumer;
    import java.util.function.Function;
    
    public final class Resultado<S, E> {
        private final S sucesso;
        private final E erro;
    
        private Resultado(S sucesso, E erro) {
            this.sucesso = sucesso;
            this.erro = erro;
        }
    
        public static <S, E> Resultado<S, E> sucesso(S valor) {
            return new Resultado<>(valor, null);
        }
    
        public static <S, E> Resultado<S, E> erro(E erroInfo) {
            return new Resultado<>(null, erroInfo);
        }
    
        public boolean isSucesso() {
            return sucesso != null;
        }
    
        public boolean isErro() {
            return erro != null;
        }
    
        public Optional<S> getValor() {
            return Optional.ofNullable(sucesso);
        }
    
        public Optional<E> getErro() {
            return Optional.ofNullable(erro);
        }
    
        // Permite encadear operações no "trilho" do sucesso
        public <R> Resultado<R, E> map(Function<S, R> mapper) {
            if (isSucesso()) {
                return Resultado.sucesso(mapper.apply(sucesso));
            }
            return Resultado.erro(erro);
        }
        
        // Consumidores para tratar os dois casos, evitando if/else no código cliente
        public Resultado<S, E> ifSucesso(Consumer<S> acao) {
            if (isSucesso()) {
                acao.accept(sucesso);
            }
            return this;
        }
    
        public Resultado<S, E> ifErro(Consumer<E> acao) {
            if (isErro()) {
                acao.accept(erro);
            }
            return this;
        }
    }
  ```
```java 
  
  // Classe para carregar os detalhes do erro
  public record ErroDeValidacao(String mensagem, String campo) {}

  ```
 ##  2. Adicionando Validação aos Domínios

  Agora, integramos o Resultado em nossas classes de domínio.
  
 ### a) ComponenteFinanceiro
  
  Cada componente agora sabe como se validar.
  
  ```java
  
    // ComponenteFinanceiro.java (evoluído)
    public class ComponenteFinanceiro {
        // ... atributos ...
    
        public Resultado<ComponenteFinanceiro, ErroDeValidacao> validar() {
            // Exemplo de regra de negócio: o valor original não pode ser negativo
            if (this.valorOriginal.compareTo(BigDecimal.ZERO) < 0) {
                return Resultado.erro(
                    new ErroDeValidacao("Valor original não pode ser negativo.", "valorOriginal")
                );
            }
            // Poderiam existir outras regras aqui...
    
            return Resultado.sucesso(this); // Sucesso, retorna a própria instância
        }
        // ... resto da classe ...
    }
  
  ```

 ### b) Parcela
  
  A parcela é responsável por orquestrar a validação de todos os seus componentes e agregar os erros.
  
  ```java
  
    // Parcela.java (evoluído)
    public abstract class Parcela {
        // ... atributos e métodos anteriores ...
    
        public Resultado<Parcela, List<ErroDeValidacao>> validarParaAmortizacao() {
            List<ErroDeValidacao> erros = this.componentes.stream()
                    .map(ComponenteFinanceiro::validar) // Valida cada componente
                    .filter(Resultado::isErro)          // Filtra apenas os resultados de erro
                    .flatMap(r -> r.getErro().stream())  // Extrai o ErroDeValidacao de dentro do Optional
                    .collect(Collectors.toList());
    
            if (!erros.isEmpty()) {
                // Se houver qualquer erro, retorna um resultado de erro com a lista completa
                return Resultado.erro(erros);
            }
    
            // Se tudo estiver OK, retorna um resultado de sucesso com a própria instância
            return Resultado.sucesso(this);
        }
    }

 ```
 ###  3. Orquestração Final na Carteira

  A Carteira agora usa este mecanismo para proteger a operação de amortização. O método receberPagamento se torna um exemplo claro de "Railway-Oriented Programming".
  
```java
  
        // Carteira.java (método receberPagamento final)
        public class Carteira {
            // ... atributos e construtor ...
        
            public void receberPagamento(int numeroParcela, BigDecimal valorPago) {
                Parcela parcelaAlvo = this.parcelas.stream()
                    // ... lógica para encontrar a parcela ...
                    .findFirst()
                    .orElseThrow(() -> new IllegalStateException("Parcela não encontrada. Este é um erro de aplicação, não de negócio."));
        
                // 1. CHAMA A VALIDAÇÃO: A operação retorna um Resultado
                Resultado<Parcela, List<ErroDeValidacao>> resultadoValidacao = parcelaAlvo.validarParaAmortizacao();
        
                // 2. PROCESSA O RESULTADO: Sem if/else, usando os métodos do objeto Resultado
                System.out.println("Tentando processar pagamento de R$" + valorPago + " para a parcela " + numeroParcela);
                
                resultadoValidacao
                    .ifSucesso(parcelaValida -> {
                        // --- TRILHO DO SUCESSO ---
                        // Este código só executa se a validação passar.
                        // A lógica de negócio principal fica isolada e limpa.
                        System.out.println("Validação OK. Aplicando estratégia de amortização...");
                        parcelaValida.aplicarPagamento(valorPago, this.estrategiaDeAmortizacao);
                        System.out.println("Pagamento aplicado com sucesso. Novo saldo da parcela: " + parcelaValida.getValorAtualizado());
                    })
                    .ifErro(erros -> {
                        // --- TRILHO DO ERRO ---
                        // Este código só executa se a validação falhar.
                        // O tratamento de erro fica isolado.
                        System.err.println("ERRO: O pagamento não pôde ser processado. A parcela está em um estado inválido.");
                        erros.forEach(erro -> 
                            System.err.println("- Campo '" + erro.campo() + "': " + erro.mensagem())
                        );
                    });
            }
        }
  
 ```

  Conclusão: Um Domínio mais Seguro e Explícito
  Ao adotar o padrão Resultado/Erro:
  
  Eliminamos Exceções para Validação: Reservamos exceções para falhas de sistema (ex: IllegalStateException se a parcela não existe), não para regras de negócio previsíveis.
  Tornamos a Falha Explícita: O tipo de retorno Resultado<S, E> força o desenvolvedor a lidar com a possibilidade de erro. É impossível esquecer.
  Limpamos o Código Cliente: A lógica de receberPagamento na Carteira ficou mais limpa e declarativa, separando claramente o "o quê" (validar, depois aplicar) do "como".
  Agregamos Erros: Fornecemos um feedback completo ao usuário ou sistema cliente, listando todos os problemas de uma vez, em vez de falhar no primeiro erro encontrado.
  Este é o padrão de um domínio que não apenas executa lógica, mas ativamente se protege, garantindo sua consistência a cada passo. É um pilar fundamental para um software confiável e de fácil manutenção.


## Assunto: REFINAMENTO FINAL - Validação Declarativa com Predicados e Regras Compostas
  Para: Time de Desenvolvimento
  De: Engenheiro Linus
  Data: 21 de junho de 2025
  
  Colegas,
  
  O design atual de validação, embora funcional, nos forçaria a modificar o método validar() da Parcela para cada nova regra de negócio. Isso é um "code smell" e uma violação do OCP.
  
  A solução é externalizar as regras de validação, tratando-as como "dados" (uma lista de regras) em vez de "código" (uma sequência de ifs). Usaremos a interface funcional Predicate como base para construir um validador genérico, declarativo e reutilizável.
  
  O Design Proposto: Um Framework de Validação Reutilizável
  A simples java.util.function.Predicate<T> retorna apenas true ou false. Precisamos de mais: em caso de falha, precisamos do ErroDeValidacao correspondente. Portanto, criaremos nossa própria abstração que combina a condição (o predicado) com o erro.
  
  ## 1. A Abstração Central: RegraDeValidacao
  Esta classe encapsula uma única regra de negócio: a condição a ser testada e o erro a ser retornado em caso de falha.
  
  ```java
  
  // RegraDeValidacao.java
  import java.util.Optional;
  import java.util.function.Predicate;
  
  public record RegraDeValidacao<T>(
      Predicate<T> condicao, 
      ErroDeValidacao erroSeFalhar
  ) {
      /**
       * Aplica a regra ao objeto.
       * @return Um Optional contendo o erro se a validação falhar, ou Optional.empty() se for bem-sucedida.
       */
      public Optional<ErroDeValidacao> aplicar(T objeto) {
          if (!condicao.test(objeto)) {
              return Optional.of(erroSeFalhar);
          }
          return Optional.empty();
      }
  }

 ```
  
 ## 2. O Motor de Validação: A Classe Validador
  Esta classe genérica será nosso motor. Ela recebe uma lista de RegraDeValidacao e as aplica a um objeto, coletando todas as falhas.
  
 ```java
  
  // Validador.java
  import java.util.List;
  import java.util.stream.Collectors;
  
  public class Validador<T> {
      private final List<RegraDeValidacao<T>> regras;
  
      public Validador(List<RegraDeValidacao<T>> regras) {
          this.regras = regras;
      }
  
      public Resultado<T, List<ErroDeValidacao>> validar(T objeto) {
          List<ErroDeValidacao> erros = this.regras.stream()
                  .map(regra -> regra.aplicar(objeto)) // Aplica cada regra
                  .filter(Optional::isPresent)        // Filtra apenas os resultados com erro
                  .map(Optional::get)                 // Extrai o erro do Optional
                  .collect(Collectors.toList());
  
          if (!erros.isEmpty()) {
              return Resultado.erro(erros);
          }
  
          return Resultado.sucesso(objeto);
      }
  }
  
 ```
 ## 3. Refatorando o Domínio para Usar o Framework

  Agora, a parte mais elegante. Removemos a lógica de validação de dentro das nossas classes de domínio e passamos a definir as regras de forma declarativa.
  
###  a) Fábrica de Validadores para ComponenteFinanceiro
  
  Criamos uma classe separada cuja única responsabilidade é construir o validador para ComponenteFinanceiro. Aqui é onde as regras de negócio vivem agora.
  
```java
  
  // ComponenteFinanceiroValidadorFactory.java
  import java.math.BigDecimal;
  import java.util.List;
  
  public class ComponenteFinanceiroValidadorFactory {
      
      // As regras são definidas declarativamente como uma lista de objetos.
      private static final List<RegraDeValidacao<ComponenteFinanceiro>> REGRAS = List.of(
          new RegraDeValidacao<>(
              comp -> comp.getValorOriginal().compareTo(BigDecimal.ZERO) >= 0,
              new ErroDeValidacao("Valor original não pode ser negativo.", "valorOriginal")
          ),
          new RegraDeValidacao<>(
              comp -> comp.getSaldoDevedor().compareTo(BigDecimal.ZERO) >= 0,
              new ErroDeValidacao("Saldo devedor não pode ser negativo.", "saldoDevedor")
          ),
          new RegraDeValidacao<>(
              comp -> comp.getSaldoDevedor().compareTo(comp.getValorOriginal()) <= 0,
              new ErroDeValidacao("Saldo devedor não pode exceder o valor original.", "saldoDevedor")
          )
          // *** NOVA REGRA PODE SER ADICIONADA AQUI SEM ALTERAR NENHUM OUTRO CÓDIGO ***
          // , new RegraDeValidacao<>( ... )
      );
  
      private static final Validador<ComponenteFinanceiro> validador = new Validador<>(REGRAS);
  
      public static Validador<ComponenteFinanceiro> getValidador() {
          return validador;
      }
  }
  
 ```
###  b) A Parcela Agora Delega a Validação
  
  O método validarParaAmortizacao() na Parcela fica drasticamente mais simples e limpo. Ele não conhece mais os detalhes das regras; ele apenas orquestra a validação usando o motor que criamos.
  
```java
  
  // Parcela.java (método de validação refatorado)
  public abstract class Parcela {
      // ...
  
      public Resultado<Parcela, List<ErroDeValidacao>> validarParaAmortizacao() {
          // Pega o validador da fábrica
          Validador<ComponenteFinanceiro> validadorDeComponente = ComponenteFinanceiroValidadorFactory.getValidador();
  
          // Itera nos componentes, valida cada um e coleta TODOS os erros
          List<ErroDeValidacao> errosAgregados = this.componentes.stream()
                  .map(validadorDeComponente::validar) // Usa o validador para cada componente
                  .filter(Resultado::isErro)
                  .flatMap(r -> r.getErro().orElse(List.of()).stream())
                  .collect(Collectors.toList());
  
          if (!errosAgregados.isEmpty()) {
              return Resultado.erro(errosAgregados);
          }
  
          return Resultado.sucesso(this);
      }
      
      // ...
  }

 ```
  Análise da Evolução e Conclusão
  Notem o que alcançamos:
  
  Conformidade Total com OCP: Para adicionar uma nova regra de validação em ComponenteFinanceiro, agora simplesmente adicionamos um novo objeto RegraDeValidacao à lista na ComponenteFinanceiroValidadorFactory. Nenhuma classe de domínio (ComponenteFinanceiro, Parcela, Carteira) precisa ser modificada. O sistema está aberto para extensão (novas regras) e fechado para modificação.
  Alta Coesão e Baixo Acoplamento: As regras de validação estão agora em um único lugar (Factory), altamente coesas. As classes de domínio (Parcela) não estão mais acopladas aos detalhes dessas regras, apenas ao Validador.
  Declarativo vs. Imperativo: Trocamos código imperativo (if/else) por uma definição declarativa de regras. Isso torna a intenção do código muito mais clara e fácil de entender.
  Reutilização: O Validador é uma classe genérica que pode ser usada para validar qualquer objeto em nosso sistema, bastando criar uma Factory com suas regras específicas.
  O Validador se torna uma ferramenta de primeira classe em nosso design de domínio. Este padrão nos permite construir sistemas complexos cuja lógica de negócio pode evoluir de forma segura e previsível.
  
  O design da Carteira e do serviço de aplicação que a consome permanece exatamente o mesmo. Eles continuam a chamar parcela.validarParaAmortizacao() e a tratar o Resultado, alheios a esta elegante refatoração interna. Isso demonstra a força de uma arquitetura bem encapsulada.

## Assunto: DESIGN FINAL - Geração de Memorial de Amortização Auditável
Para: Time de Desenvolvimento
De: Engenheiro Linus
Data: 21 de junho de 2025

Colegas,

O requisito final para o nosso fluxo de amortização é a geração de um "Memorial de Amortização" detalhado para cada pagamento. Este memorial deve servir como um registro imutável e auditável do evento. A questão central de design é: qual objeto é o especialista que possui toda a informação para criar este memorial?

A resposta não é a Carteira (que não conhece os detalhes internos da parcela) nem a Parcela (que não deveria ter a responsabilidade de formatação de relatórios). O verdadeiro especialista é a IAmortizacaoStrategy. É a estratégia que sabe exatamente como o valor foi distribuído entre os componentes.

Portanto, a estratégia não irá mais simplesmente modificar os componentes; ela irá executar a modificação e relatar em detalhes o que foi feito.

O Design Proposto: Eventos de Domínio como Resultado
Faremos com que a operação de amortização retorne um objeto de valor imutável, nosso MemorialDeAmortizacao, que representa o resultado detalhado do evento.

## 1. Os Objetos de Valor do Memorial (Os Dados)

Primeiro, definimos as estruturas de dados que irão compor nosso memorial. São objetos imutáveis, perfeitos para DTOs ou Value Objects.

```java
// DetalheAplicacaoComponente.java
import java.math.BigDecimal;

/**
 * Registra o efeito da amortização em um único componente financeiro.
 */
public record DetalheAplicacaoComponente(
    TipoComponente tipo,
    BigDecimal saldoAnterior,
    BigDecimal valorAplicado,
    BigDecimal saldoNovo
) {}

// MemorialDeAmortizacao.java
import java.math.BigDecimal;
import java.time.OffsetDateTime;
import java.util.List;
import java.util.UUID;

/**
 * Um registro imutável e detalhado de uma operação de amortização.
 * Este é o nosso evento de domínio materializado.
 */
public record MemorialDeAmortizacao(
    UUID idTransacao,
    OffsetDateTime dataProcessamento,
    BigDecimal valorPago,
    String nomeEstrategiaUsada,
    List<DetalheAplicacaoComponente> detalhesPorComponente,
    BigDecimal totalAmortizado,
    BigDecimal valorNaoUtilizado // Troco
) {}

```
## 2. Evoluindo a IAmortizacaoStrategy para Relatar o que Fez

A mudança principal é no contrato da estratégia. Em vez de retornar void, ela retornará o MemorialDeAmortizacao.

```java

    // IAmortizacaoStrategy.java (interface modificada)
    public interface IAmortizacaoStrategy {
        /**
         * Aplica o pagamento aos componentes e retorna um memorial detalhado da operação.
         */
        MemorialDeAmortizacao aplicar(List<ComponenteFinanceiro> componentes, BigDecimal valorPago);
    }

```
Agora, a implementação da estratégia se torna responsável por construir este relatório.

```java

    // JurosPrimeiroStrategy.java (implementação modificada)
    public class JurosPrimeiroStrategy implements IAmortizacaoStrategy {
        private static final List<TipoComponente> ORDEM_PAGAMENTO = /* ... */;

        @Override
        public MemorialDeAmortizacao aplicar(List<ComponenteFinanceiro> componentes, BigDecimal valorPago) {
            BigDecimal valorRestante = valorPago;
            List<DetalheAplicacaoComponente> detalhes = new ArrayList<>();
            
            // ... Lógica para ordenar os componentes ...

            for (TipoComponente tipo : ORDEM_PAGAMENTO) {
                ComponenteFinanceiro componente = //... encontra o componente
                if (componente == null) continue;

                BigDecimal saldoAnterior = componente.getSaldoDevedor();
                if (saldoAnterior.compareTo(BigDecimal.ZERO) == 0) continue; // Pula se já está zerado

                // Amortiza e captura o valor que não foi usado
                BigDecimal valorNaoUtilizadoNaEtapa = componente.amortizar(valorRestante);
                
                BigDecimal saldoNovo = componente.getSaldoDevedor();
                BigDecimal valorAplicado = saldoAnterior.subtract(saldoNovo);

                if (valorAplicado.compareTo(BigDecimal.ZERO) > 0) {
                    detalhes.add(new DetalheAplicacaoComponente(tipo, saldoAnterior, valorAplicado, saldoNovo));
                }
                valorRestante = valorNaoUtilizadoNaEtapa;
            }

            BigDecimal totalAmortizado = valorPago.subtract(valorRestante);

            return new MemorialDeAmortizacao(
                UUID.randomUUID(),
                OffsetDateTime.now(),
                valorPago,
                this.getClass().getSimpleName(), // Nome da estratégia usada
                detalhes,
                totalAmortizado,
                valorRestante // O troco
            );
        }
    }
```

## 3. Propagando o Memorial pela Pilha de Chamadas

As classes de domínio agora simplesmente repassam o memorial para o chamador.

```java
    // Parcela.java (método aplicarPagamento modificado)
    public abstract class Parcela {
        // ...
        public MemorialDeAmortizacao aplicarPagamento(BigDecimal valorPago, IAmortizacaoStrategy estrategia) {
            MemorialDeAmortizacao memorial = estrategia.aplicar(this.componentes, valorPago);
            
            if (getValorAtualizado().compareTo(BigDecimal.ZERO) == 0) {
                this.status = StatusParcela.PAGA;
            }
            return memorial; // Retorna o memorial para o chamador
        }
        // ...
    }
```
## 4. O Resultado Final na Carteira (A Combinação Perfeita)
A Carteira agora tem um método receberPagamento que retorna o resultado final da operação: ou uma lista de erros de validação, ou o memorial de sucesso. O nosso objeto Resultado brilha aqui.

```java

    // Carteira.java (método receberPagamento modificado)
    public class Carteira {
        // ...
        public Resultado<MemorialDeAmortizacao, List<ErroDeValidacao>> receberPagamento(int numeroParcela, BigDecimal valorPago) {
            Parcela parcelaAlvo = //... encontra a parcela ou lança exceção

            // A mágica da programação funcional e do nosso design:
            // 1. Valida a parcela.
            // 2. Se a validação for bem-sucedida, o `.map()` aplica a função seguinte.
            // 3. A função interna aplica o pagamento, que retorna o Memorial.
            // 4. O `.map()` encapsula o Memorial em um `Resultado.sucesso()`.
            // 5. Se a validação falhar, o `.map()` é ignorado e o resultado de erro original é retornado.
            return parcelaAlvo.validarParaAmortizacao()
                .map(parcelaValida -> parcelaValida.aplicarPagamento(valorPago, this.estrategiaDeAmortizacao));
        }
    } 

```
Como o Serviço de Aplicação usaria isso:

```java

    // Exemplo no Application Service
    Carteira minhaCarteira = //...
    Resultado<MemorialDeAmortizacao, List<ErroDeValidacao>> resultado = 
        minhaCarteira.receberPagamento(1, new BigDecimal("150.00"));

    resultado.ifSucesso(memorial -> {
        System.out.println("=== Memorial de Amortização ===");
        System.out.println("ID Transação: " + memorial.idTransacao());
        System.out.println("Estratégia: " + memorial.nomeEstrategiaUsada());
        memorial.detalhesPorComponente().forEach(detalhe -> {
            System.out.println(
                "  - Componente: " + detalhe.tipo() +
                " | Saldo Anterior: " + detalhe.saldoAnterior() +
                " | Valor Aplicado: " + detalhe.valorAplicado() +
                " | Saldo Novo: " + detalhe.saldoNovo()
            );
        });
        System.out.println("Total Amortizado: " + memorial.totalAmortizado());
        // Aqui, o memorial poderia ser salvo no banco, enviado para uma fila, etc.

    }).ifErro(erros -> {
        System.err.println("Pagamento falhou na validação.");
        // ... loga os erros
    });

```
Conclusão
Este design final alcança todos os nossos objetivos. A responsabilidade de criar o memorial está precisamente onde deveria estar: na Strategy. Nossas entidades de domínio (Parcela, Carteira) permanecem limpas, focadas em suas responsabilidades principais. O fluxo de dados é explícito, imutável e seguro.

O resultado é um sistema que não apenas funciona corretamente, mas também conta a história de como ele funciona, garantindo a rastreabilidade e a confiança que são indispensáveis em nosso domínio de negócio.   

## Assunto: Análise de Design: Amortização Antecipada e o Efeito Cascata no Agregado

Para: Time de Desenvolvimento
De: Engenheiro Linus
Data: 21 de junho de 2025

Colegas,

A nova exigência de que uma amortização antecipada recalcule todo o cronograma futuro segundo a Curva Price nos apresenta um desafio de design de ordem superior. O padrão de interação dentro do nosso domínio muda fundamentalmente. Devemos analisar este cenário com extremo cuidado para não comprometer a integridade do modelo que construímos.

Análise do Cenário: O Efeito Cascata e o Especialista da Informação
Até agora, nossas operações (pagar uma parcela) eram locais. A ação de pagar a Parcela N afetava apenas o estado da Parcela N. O restante do agregado permanecia inalterado.

O novo requisito introduz um efeito cascata (ripple effect). Uma ação na Parcela N (amortizar o principal) agora dispara uma alteração em cadeia nas Parcela N+1, Parcela N+2, e assim por diante. A integridade do cronograma como um todo precisa ser recalculada e mantida.

A pergunta central de design que devemos responder é: Qual objeto é o especialista da informação e do comportamento para governar esta operação complexa?

Vamos avaliar os candidatos:

## 1. A Entidade Parcela?
Análise: Poderíamos dar à Parcela um método amortizarPrincipalAntecipadamente que, de alguma forma, acessasse e modificasse suas parcelas "irmãs"?
Veredito: Inaceitável. Este seria um erro grave de design. Uma entidade Parcela não deve ter conhecimento sobre a coleção à qual pertence. Forçar essa consciência criaria um acoplamento cíclico e altíssimo entre as entidades, tornando o modelo frágil e impossível de raciocinar sobre. A Parcela é especialista em seu próprio estado, não no estado do contrato inteiro.

## 2. A IAmortizacaoStrategy?
Análise: A estratégia de amortização já lida com o pagamento. Poderíamos expandi-la para também recalcular o resto do cronograma?
Veredito: Incorreto. Isso violaria grosseiramente o Princípio da Responsabilidade Única (SRP). A responsabilidade da IAmortizacaoStrategy é clara e focada: aplicar um valor pago aos componentes internos de uma única parcela. A responsabilidade de recalcular um cronograma financeiro inteiro é um domínio de negócio completamente distinto. Misturá-los tornaria a estratégia inflada, complexa e com duas razões para mudar.

## 3. A Carteira (Raiz do Agregado)?

Análise: A Carteira é a Raiz do nosso Agregado. Por definição em DDD, a raiz é a única que pode ser referenciada diretamente de fora do agregado e é responsável por manter a consistência e as invariantes de todos os objetos dentro de seus limites. A lista de parcelas é o coração do estado da Carteira.
Veredito: Correto. A Carteira é a orquestradora. Ela é o único objeto que possui a visão completa do cronograma e a autoridade para realizar uma mudança em massa em seus objetos filhos (as Parcelas) para garantir a consistência do todo. A Carteira sabe quando um recálculo é necessário.
O Refinamento do Design: Orquestração vs. Algoritmo
Identificamos a Carteira como a orquestradora. Agora, a segunda pergunta: devemos colocar a lógica matemática complexa da Curva Price dentro da classe Carteira?

A resposta é não. Fazer isso tornaria a Carteira uma classe massiva, misturando a responsabilidade de orquestração de estado com a de cálculo financeiro algorítmico.

Para manter a coesão e, crucialmente, aderir ao Princípio Aberto/Fechado (OCP), vamos delegar o "como fazer" o cálculo para uma nova abstração:

Criação de um novo Papel: Definiremos um novo conceito no nosso domínio: Estratégia de Recálculo de Cronograma (IRecalculoDeCronogramaStrategy). Este será um Serviço de Domínio (stateless) ou uma Estratégia.
Implementações Concretas: A lógica específica da Curva Price será encapsulada em uma classe PriceRecalculoStrategy. Se no futuro precisarmos do método SAC, criaremos uma SacRecalculoStrategy.
Injeção de Dependência: A Carteira será configurada no momento de sua criação com a estratégia de recálculo apropriada para aquele tipo de contrato.
Conclusão da Análise
O design final para este requisito complexo é o seguinte:

Um cliente (Serviço de Aplicação) chama um novo método na Carteira, por exemplo, amortizarAntecipadamente().
A Carteira, como orquestradora, executa os seguintes passos:
Valida a operação.
Invoca a IAmortizacaoStrategy para aplicar o pagamento à parcela-alvo.
Analisa o resultado (o Memorial) para ver se o principal foi amortizado.
Se sim, invoca sua IRecalculoDeCronogramaStrategy injetada, passando os dados necessários (saldo devedor remanescente, taxa, etc.).
Recebe da estratégia uma nova lista de parcelas futuras.
Como guardiã do estado, ela substitui as antigas parcelas futuras pelas novas, garantindo a consistência do agregado.
Este design separa as responsabilidades de forma limpa e nos dá total flexibilidade para estender o sistema com novos modelos de cálculo financeiro sem nunca modificar a classe Carteira, cumprindo com excelência o Princípio Aberto/Fechado.
