# Value object `Dinheiro` — quando o valor chega em dois formatos diferentes

> Use quando o mesmo valor monetário entra no serviço por **duas fontes com formatos
> distintos** — uma manda centavos (`"10002"`), outra manda reais (`"100.02"`) — e você
> precisa comparar e subtrair sem perder centavo.
>
> A classe guarda **centavos inteiros** (`Long`). Todo o resto é tradução de fronteira.

## Por que não `Double`, `Float` nem `String`

`Double` não representa `0.1` exatamente: subtrações encadeadas divergem, e o erro só
aparece depois de N iterações — em produção, não no teste feliz.

E tem um efeito colateral que morde antes disso: **`Double.toString()` vira notação
científica a partir de 10⁷**. Um valor de dez milhões é gravado como `"1.0E7"`. Se o valor
foi persistido como texto, o dado no banco fica assim para sempre.

Se você encontrar valores terminados em `.0` (`"42.0"`, `"50000.0"`) num campo de texto,
essa é a assinatura de `Double.toString()` no caminho de escrita. Procure antes de culpar
a leitura:

```bash
grep -rnE "Double|Float|toDouble\(\)|BigDecimal\(\s*[a-z]" --include="*.kt" .
```

📌 `BigDecimal(double)` é tão ruim quanto o `Double` cru: ele fotografa o erro do binário
(`BigDecimal(0.1)` → `0.1000000000000000055511151231257827…`). Só o construtor que recebe
`String` é confiável.

---

## A classe

```kotlin
package <SEU_PACOTE>.domain

import java.math.BigDecimal

/**
 * Valor monetário, guardado como centavos inteiros.
 *
 * Fronteiras:
 *  - fonte A -> centavos, sem separador ("10002")
 *  - fonte B -> reais, com ponto ("100.02")
 */
data class Dinheiro(val centavos: Long) : Comparable<Dinheiro> {

    operator fun minus(outro: Dinheiro) = Dinheiro(centavos - outro.centavos)

    fun paraReais(): String = BigDecimal.valueOf(centavos, 2).toPlainString()

    override fun compareTo(other: Dinheiro) = centavos.compareTo(other.centavos)
    override fun toString() = paraReais()

    companion object {
        val ZERO = Dinheiro(0)

        /** "10002" -> 100,02 */
        fun deCentavos(valor: String) = Dinheiro(valor.trim().toLong())
        fun deCentavos(valor: Long) = Dinheiro(valor)

        /** "100.02", "50000.0", "100", "1.0E7" -> valor exato */
        fun deReais(valor: String) = deReais(BigDecimal(valor.trim()))
        fun deReais(valor: BigDecimal) = Dinheiro(valor.movePointRight(2).setScale(0).toLong())
    }
}
```

### As cinco escolhas que não são acidentais

**`data class`, não `@JvmInline value class`.** Value class sofre *name mangling* no getter
(`getValor-x1y2z3()`). O Enhanced Client do DynamoDB mapeia por reflexão sobre convenção de
bean e simplesmente **não enxerga a propriedade**. Jackson e alguns frameworks de mock também
tropeçam. A otimização não paga o custo.

**`BigDecimal.valueOf(centavos, 2)`** é a conversão exata para 2 casas — sem parse de string,
sem risco de escala errada.

**`toPlainString()`, nunca `toString()`.** `toString()` pode emitir notação científica e
recolocar no banco o problema que você acabou de resolver.

**`setScale(0)` sem `RoundingMode` em `deReais`** é proposital: se chegar `"10.025"`, lança
`ArithmeticException` em vez de descartar meio centavo em silêncio. Se o domínio realmente
recebe fração de centavo, aí sim passe um modo — mas isso é decisão de negócio, não default.

**Sem `require(centavos >= 0)`.** Se a condição de parada do seu fluxo é `<= 0`, negativo é
estado legítimo. Só proíba se houver regra dizendo isso.

### O que ficou de fora, de propósito

`plus`, `unaryMinus`, multiplicação por taxa, divisão em parcelas. Nenhum é difícil de
escrever — o problema é outro: **método não usado congela uma decisão sem regra que a
sustente**. Uma multiplicação precisa de `RoundingMode`, e o default que você escolher hoje
sem caso de uso vira "a regra" no dia em que alguém usar. Value object seu, com teste:
adicionar depois custa minutos.

📌 Quando a taxa aparecer de verdade, ela **não** vira um `vezes` aqui dentro: vira classe
própria, com o arredondamento como parâmetro. Ver
[value-object-percentual](202608312157-value-object-percentual.md).

📌 Se você **for** precisar dividir em parcelas, não use divisão simples — o centavo residual
some e a soma das parcelas deixa de bater com o total. A operação certa distribui o resto:

```kotlin
fun repartir(partes: Int): List<Dinheiro> {
    require(partes > 0) { "partes deve ser maior que zero, veio $partes" }
    val base = centavos / partes
    val resto = centavos % partes
    val sinal = if (resto < 0) -1L else 1L
    val sobras = kotlin.math.abs(resto).toInt()
    return List(partes) { i -> Dinheiro(base + if (i < sobras) sinal else 0L) }
}
```

O `sinal` existe porque `%` em Kotlin devolve resto **negativo** para dividendo negativo.
Sem ele, o parcelamento de valor negativo não conserva o total.

---

## O teste

```kotlin
package <SEU_PACOTE>.domain

import org.junit.jupiter.api.Test
import org.junit.jupiter.api.assertThrows
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class DinheiroTest {

    @Test
    fun `le os formatos irregulares que ja existem gravados`() {
        assertEquals(Dinheiro(5_000_000), Dinheiro.deReais("50000.0"))
        assertEquals(Dinheiro(5_000_000), Dinheiro.deReais("50000.00"))
        assertEquals(Dinheiro(5_000_000), Dinheiro.deReais("50000"))
        assertEquals(Dinheiro(10_002), Dinheiro.deReais("100.02"))
    }

    @Test
    fun `le notacao cientifica gravada por Double`() {
        assertEquals(Dinheiro(1_000_000_000), Dinheiro.deReais("1.0E7"))
    }

    @Test
    fun `as duas fontes produzem o mesmo valor`() {
        assertEquals(Dinheiro.deReais("100.02"), Dinheiro.deCentavos("10002"))
    }

    @Test
    fun `normaliza o formato ao gravar`() {
        assertEquals("100.02", Dinheiro.deCentavos(10_002).paraReais())
        assertEquals("50000.00", Dinheiro.deReais("50000.0").paraReais())
    }

    @Test
    fun `rejeita fracao de centavo em vez de arredondar calado`() {
        assertThrows<ArithmeticException> { Dinheiro.deReais("10.025") }
    }

    @Test
    fun `subtrai sem perder centavo`() {
        assertEquals(Dinheiro(5_002), Dinheiro.deReais("100.02") - Dinheiro.deReais("50.00"))
    }

    @Test
    fun `permite saldo negativo`() {
        val saldo = Dinheiro.deReais("10.00") - Dinheiro.deReais("30.00")
        assertEquals(Dinheiro(-2_000), saldo)
        assertTrue(saldo <= Dinheiro.ZERO)
    }

    @Test
    fun `compara numericamente e nao lexicograficamente`() {
        assertTrue(Dinheiro.deReais("100.02") > Dinheiro.deReais("9.00"))
        assertTrue(Dinheiro.deReais("9.00") <= Dinheiro.deReais("100.02"))
    }

    @Test
    fun `zero tem representacao unica`() {
        assertEquals(Dinheiro.ZERO, Dinheiro.deReais("0.00"))
        assertEquals(Dinheiro.ZERO, Dinheiro.deReais("0.0"))
        assertEquals(Dinheiro.ZERO, Dinheiro.deReais("0"))
    }

    @Test
    fun `subtracoes sucessivas nao acumulam erro`() {
        var saldo = Dinheiro.deReais("50000.0")
        val fatia = Dinheiro.deReais("1666.67")
        repeat(30) { saldo -= fatia }
        assertEquals(Dinheiro.deReais("49.90"), saldo)
    }
}
```

Os dois que justificam a classe inteira:

**`subtracoes sucessivas nao acumulam erro`** — em `Double` isso diverge; em centavos
inteiros é exato por construção.

**`zero tem representacao unica`** — comparar com zero via `equals` só é seguro porque cada
valor tem **uma** representação. Com `BigDecimal` seria armadilha: `BigDecimal("0.0")` e
`BigDecimal("0.00")` são diferentes no `equals` (iguais só no `compareTo`).

---

## Onde a classe mora

| Camada | O que fica lá |
|---|---|
| **domínio** | `Dinheiro` — value object puro |
| **adaptadores** | o `AttributeConverter`, os mappers de request/response |
| **serviços** | orquestra; só manipula `Dinheiro`, nunca `String` de dinheiro |

O teste decisivo: se `Dinheiro.kt` precisar de `import software.amazon.awssdk...` ou de
Jackson, **está na camada errada**. A tradução de fronteira é responsabilidade do adaptador.

Se aparecer `String` de dinheiro ou `BigDecimal` solto na camada de serviço, é sinal de que
a conversão vazou da borda.

---

## Ligando no DynamoDB (Enhanced Client)

```kotlin
class DinheiroReaisConverter : AttributeConverter<Dinheiro> {
    override fun transformFrom(input: Dinheiro): AttributeValue =
        AttributeValue.builder().s(input.paraReais()).build()

    override fun transformTo(input: AttributeValue): Dinheiro = when {
        input.s() != null -> Dinheiro.deReais(input.s())
        input.nul() == true -> Dinheiro.ZERO
        else -> error("valor em formato inesperado: $input")
    }

    override fun type() = EnhancedType.of(Dinheiro::class.java)
    override fun attributeValueType() = AttributeValueType.S
}
```

```kotlin
@DynamoDbBean
class MinhaEntity {
    @get:DynamoDbPartitionKey
    var id: String = ""

    @get:DynamoDbConvertedBy(DinheiroReaisConverter::class)
    var valor: Dinheiro = Dinheiro.ZERO

    @get:DynamoDbVersionAttribute
    var versao: Long? = null
}
```

O ganho de existir converter, mesmo mantendo `S`: nenhuma regra de negócio sabe que existem
dois formatos, e a troca de tipo um dia custa **um converter**, não uma varredura pelo código.

### ⚠️ Três consequências de guardar dinheiro como `S`

Atributo fora das chaves no DynamoDB **não tem tipo declarado** — o tipo é decidido item a
item, por quem escreve. Se está `S` hoje, foi o código que escolheu. Isso significa:

| Consequência | Detalhe |
|---|---|
| **Comparação é lexicográfica** | `"9.00" > "100.02"` é `true` como string. Todo `>`, `<` e `BETWEEN` nesse atributo está errado |
| **Não existe update atômico** | `SET x = x + :v` só funciona em `N`. Com `S` é ler-modificar-escrever, que perde atualização sob concorrência |
| **Formato vira identidade** | `"100.0"` e `"100.00"` são valores distintos numa `ConditionExpression` de igualdade |

📌 `N` no DynamoDB é decimal exato de até 38 dígitos, não é float — guardar número em `N`
não tem o problema que as pessoas temem.

**Quando migrar `S` → `N` vale a pena:** tabela de vida longa, ou mais de um consumidor.
Nesse caso **não troque o tipo do atributo no lugar** — atributo com `S` em uns itens e `N`
em outros quebra todo consumidor que você não controla. Escreva **atributo novo em paralelo**
(dual-write), migre os leitores, faça o backfill, só então apague o antigo.

**Quando não vale:** tabela transitória, um único leitor, vida curta. Aí normalize a escrita
e siga — dual-write e backfill não se pagam numa tabela que vai ser descartada.

### ⚠️ Se o atributo for chave de índice

`S` como chave só ordena certo com **largura fixa e zero-padding**:

```kotlin
fun paraChave(): String = "%019d".format(centavos)   // "0000000000000010002"
```

E isso só vale para valores não-negativos — negativo quebra a ordenação lexicográfica e
exige deslocar tudo para o positivo com um offset.

### Concorrência

O `@DynamoDbVersionAttribute` acima não é enfeite: como `S` obriga a ler-modificar-escrever,
é ele que gera a condição de versão e faz o update **falhar** em vez de sobrescrever. Trate a
`ConditionalCheckFailedException` refazendo a leitura. Sem isso, duas requisições simultâneas
no mesmo item e uma delas some.

Se o fluxo tem retry e cada escrita é uma etapa de um processo mais longo, versão não basta —
garanta idempotência por etapa (id da etapa na condição de escrita) em vez de confiar que o
processo não repete.
