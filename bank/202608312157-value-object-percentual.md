# Value object `Percentual` — aplicar taxa sobre dinheiro sem criar nem sumir centavo

> Continuação de [value-object-dinheiro](202608312136-value-object-dinheiro.md). Use quando
> uma taxa chega de fora (API, configuração, banco) e multiplica um valor monetário.
>
> Duas coisas precisam ser resolvidas **antes** do código, e nenhuma é de programação.

## Pergunta 1 — a unidade

`"10.0"` é **10 por cento** ou **0,1 por cento**? A diferença é de 100×.

A convenção desta receita: `"10.0"` = 10% = fração `0.10`. Se fosse fração, o campo viria
como `"0.1"`.

🔴 **Confirme com quem publica o campo e trave a resposta num teste.** "Quase certamente é
percentual" é exatamente o raciocínio que produz incidente. O teste vira a documentação
executável dessa conversa — se alguém mudar a interpretação depois, ele quebra com a
mensagem certa.

📌 Valor terminado em `.0` (`"10.0"`, `"42.0"`) é assinatura de `Double.toString()` na origem.
Não muda o que você faz aqui, mas veja a receita do `Dinheiro` — do outro lado da fronteira
isso pode significar precisão já perdida na escrita.

## Pergunta 2 — o arredondamento

Percentual sobre dinheiro **sempre** gera fração de centavo. Alguém tem que decidir o que
fazer com ela, e não existe default honesto:

| Modo | Quando |
|---|---|
| `HALF_UP` | mais comum em contexto financeiro BR; é o que auditoria costuma esperar |
| `HALF_EVEN` | evita viés acumulado quando a operação se repete muitas vezes |

Enquanto a regra não vier do negócio, deixe **explícita e greppável** — nunca escondida num
default de parâmetro:

```kotlin
// domain — a regra, nomeada e auditável
val ARREDONDAMENTO_MONETARIO = RoundingMode.HALF_UP   // ⚠️ confirmar com negócio
```

A diferença entre uma decisão registrada e um default esquecido é essa linha.

---

## A classe

```kotlin
package <SEU_PACOTE>.domain

import java.math.BigDecimal
import java.math.RoundingMode

data class Percentual(val fracao: BigDecimal) {

    /** Arredondamento é decisão de negócio: entra como parâmetro, não como default. */
    fun aplicarSobre(valor: Dinheiro, modo: RoundingMode): Dinheiro =
        Dinheiro(
            BigDecimal.valueOf(valor.centavos)
                .multiply(fracao)
                .setScale(0, modo)
                .toLong()
        )

    override fun toString() = fracao.movePointRight(2).toPlainString() + "%"

    companion object {
        /** "10.0" -> 10% */
        fun dePercentual(valor: String) = dePercentual(BigDecimal(valor.trim()))
        fun dePercentual(valor: BigDecimal) = Percentual(valor.movePointLeft(2).stripTrailingZeros())
    }
}
```

```kotlin
val total = Dinheiro.deReais(valorRecebido)
val taxa  = Percentual.dePercentual(percentualRecebido)
val parte = taxa.aplicarSobre(total, ARREDONDAMENTO_MONETARIO)
```

### As três escolhas que não são acidentais

**`aplicarSobre` mora no `Percentual`, não no `Dinheiro`.** Assim o `Dinheiro` continua sem
conhecer arredondamento nenhum. Se você colocar um `vezes(BigDecimal)` no `Dinheiro`, ele
precisa de um `RoundingMode` default — e esse default vira "a regra" no dia em que alguém usar
sem pensar. Deixe a decisão do lado de quem tem contexto para tomá-la.

**Uma fábrica só, não duas.** A tentação é oferecer `dePercentual("10.0")` **e**
`deFracao("0.1")` para cobrir os dois casos. Não faça, se só uma notação atravessa a sua
fronteira. Com uma fábrica só, **não existe fábrica errada para escolher** — a ambiguidade de
100× deixa de ser possível na superfície da API, em vez de depender de alguém lembrar da
convenção. Se um dia a segunda notação aparecer de verdade, aí ela ganha fábrica própria.

**`stripTrailingZeros` nas fábricas.** `equals` de `BigDecimal` compara escala: sem isso,
`Percentual(0.100) != Percentual(0.10)`. Contanto que tudo passe pelas fábricas, a comparação
se comporta.

---

## ⚠️ A armadilha que vale mais que a classe inteira

Se existir a **parte complementar** — o que sobrou, o restante, a outra fatia — calcule por
subtração, **nunca** por um segundo percentual:

```kotlin
val parte       = taxa.aplicarSobre(total, ARREDONDAMENTO_MONETARIO)
val complemento = total - parte        // ✅ conserva
```

O motivo é concreto. `100,05` com 50%:

| Caminho | Parte | Complemento | Soma |
|---|---|---|---|
| dois percentuais | 50,03 | 50,03 | **100,06** ❌ |
| percentual + subtração | 50,03 | 50,02 | 100,05 ✅ |

Meio centavo arredondado para cima nos dois lados **cria dinheiro do nada**. Repetido ao longo
de um ciclo, vira divergência de conciliação que ninguém consegue explicar depois.

A mesma lógica vale para N fatias: aplique o percentual em N−1 e obtenha a última por
subtração. Ou, se as fatias são iguais, use `repartir` (na receita do `Dinheiro`), que já
distribui o resto.

---

## O teste

```kotlin
package <SEU_PACOTE>.domain

import org.junit.jupiter.api.Test
import java.math.BigDecimal
import java.math.RoundingMode
import kotlin.test.assertEquals

class PercentualTest {

    private val modo = RoundingMode.HALF_UP

    @Test
    fun `dez virgula zero e dez por cento e nao um decimo por cento`() {
        assertEquals(
            Dinheiro.deReais("10.00"),
            Percentual.dePercentual("10.0").aplicarSobre(Dinheiro.deReais("100.00"), modo),
        )
    }

    @Test
    fun `resultado e sempre centavo inteiro`() {
        // 10% de 100,02 = 10,002 -> 10,00
        assertEquals(
            Dinheiro.deReais("10.00"),
            Percentual.dePercentual("10.0").aplicarSobre(Dinheiro.deReais("100.02"), modo),
        )
    }

    @Test
    fun `cem por cento devolve o valor intacto`() {
        val total = Dinheiro.deReais("100.02")
        assertEquals(total, Percentual.dePercentual("100.0").aplicarSobre(total, modo))
    }

    @Test
    fun `zero por cento devolve zero`() {
        assertEquals(
            Dinheiro.ZERO,
            Percentual.dePercentual("0.0").aplicarSobre(Dinheiro.deReais("100.02"), modo),
        )
    }

    @Test
    fun `complemento por subtracao conserva o total`() {
        val total = Dinheiro.deReais("100.05")
        val parte = Percentual.dePercentual("50.0").aplicarSobre(total, modo)
        val complemento = total - parte
        assertEquals(Dinheiro.deReais("50.03"), parte)
        assertEquals(Dinheiro.deReais("50.02"), complemento)
        assertEquals(total, parte + complemento)   // exige `plus` no Dinheiro
    }

    @Test
    fun `dois percentuais independentes NAO conservam`() {
        // documenta por que o complemento é subtração, e não outro percentual
        val total = Dinheiro.deReais("100.05")
        val metade = Percentual.dePercentual("50.0")
        val a = metade.aplicarSobre(total, modo)
        val b = metade.aplicarSobre(total, modo)
        assertEquals(Dinheiro.deReais("100.06"), a + b)   // um centavo do nada
    }
}
```

O primeiro teste é a documentação executável da Pergunta 1. Os dois últimos são a da armadilha
— o teste que afirma o comportamento **errado** existe de propósito: ele impede que alguém
"conserte" o código para calcular o complemento por percentual.

📌 Os dois testes de conservação usam `plus` no `Dinheiro`. Se o seu fluxo nunca soma dinheiro,
não reintroduza o operador só para o teste — troque a asserção por comparação de `centavos`.

---

## Quando **não** criar esta classe

Se a taxa é constante no código (`val TAXA = BigDecimal("0.05")`), ela não atravessa fronteira
nenhuma: o valor está à vista, a unidade é inequívoca na declaração, e um `BigDecimal` com nome
bom já resolve. Classe aí é cerimônia.

O critério é o mesmo do `Dinheiro`: **o valor atravessa uma fronteira com formato que você não
controla?** Se sim, a classe se paga — ela dá endereço para a ambiguidade de unidade e para a
decisão de arredondamento. Se não, não se paga.
