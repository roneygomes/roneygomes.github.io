---
title: "Polimorfismo em Rust: Traits"
date: 2021-11-08T19:34:22-03:00
draft: true
---

Talvez você tenha alguma familiaridade com o conceito de polimorfismo por já tê-lo explorado em outras linguagens de
programação, mas ainda assim, antes de começarmos a ver o lado Rust das coisas acho que é uma boa ideia estabelecermos
uma definição simples do que é polimorfismo.

Num entendimento bem rasteiro da etimologia da palavra, polimorfismo descreve uma coisa que tem várias formas. Uma certa
coisa é polimórfica se ela consegue se manifestar de várias formas diferentes sem perder a sua essência. Vamos
considerar a ação de escrever como um exemplo. A escrita é uma operação polimórfica porque pode ser realizada de várias
maneiras: a gente pode escrever num papel, num computador, numa lousa, na areia etc.

Traduzindo esse conceito pro universo das linguagens de programação, a gente pode entender que o polimorfismo descreve a
capacidade que algumas operações tem de serem aplicadas a tipos diferentes. Se definirmos um método `write` pra
representar a nossa operação de escrita, poderíamos criar uma versão sua pra cada tipo em que pode ser aplicado:

```rust
// escrevendo em papel
fn write_on_paper(media: &mut Paper, text: &str);

// escrevendo num computador
fn write_on_computer(media: &mut Computer, text: &str);

// escrevendo na areia
fn write_on_sand(media: &mut Sand, text: &str);
```

O problema dessa abordagem, além de não ser muito prático ter que escrever uma nova função sempre que precisarmos
escrever em uma mídia nova, é que não temos nada que garanta que a implementação dessas funções de fato preserva o
propósito original da operação: escrever em alguma coisa. É como se faltasse algo que unificasse essas funções pra que
possíveis clientes possam utilizá-las de forma intercambiável sabendo que a funcionalidade de escrita se mantém.

Como então é que Rust poderia nos habilitar a escrever uma única função `write` que pudesse ser aplicada a tudo quanto é
tipo de mídia sem que a gente precise ficar se repetindo?

```rust
fn write(media: &mut ???, text: &str);
```

Conforme veremos, existem duas maneiras de se fazer isso: **traits** e **generics**. Nesse artigo vamos falar
exclusivamente sobre traits, exploraremos funções genéricas mais à frente numa parte dois, ou três, ainda a ser
publicada.

## Traits

Quem tem alguma familiaridade com Java pode entender traits como se fossem interfaces, uma espécie de protocolo que
podemos definir pra que métodos operem sob uma expectativa comum. Essa espécie de contrato serve pra desacoplar os
usuários dessas funções das estruturas de dados que as implementam, permitindo que operem apenas no nível da interface
sem realmente se importar com características de implementação. Coleções costumam ser um exemplo bem frequente pois,
geralmente, clientes estão apenas interessados nas operações que podem realizar sobre elas: adicionar um elemento,
remover um elemento, percorrer a coleção etc.

Vamos tomar um exemplo real pra entender melhor como traits são declaradas. O código a seguir é um trecho da definição
da trait `Write` disponível no módulo `std::io` da biblioteca padrão.

```rust
pub trait Write {
    // escreve um buffer nesse escritor retornando
    // quantos bytes foram escritos em caso de sucesso
    fn write(&mut self, buf: &[u8]) -> Result<usize>;

    // limpa o escritor, garantindo que qualquer conteúdo
    // armazenado em buffers intermediários seja persistido
    // no armazenamento final
    fn flush(&mut self) -> Result<()>;

    // ...
}
```

Dá pra ver que definir uma trait é uma operação bem simples. Basta pensar em um nome e depois declarar uma lista de
métodos.

Implementar uma trait também não é coisa de outro mundo. No exemplo a seguir vamos implementar `Write` pra um tipo bem
simples que armazena o conteúdo sendo escrito num array interno.

```rust
use std::io::Write;
use std::io::Result;

#[derive(Debug)]
struct ArrayWrapper {
    pub buffer: Vec<u8>,
}

impl Write for ArrayWrapper {
    fn write(&mut self, buf: &[u8]) -> Result<usize> {
        for byte in buf {
            self.buffer.push(byte.clone())
        }

        Ok(buf.len())
    }

    fn flush(&mut self) -> Result<()> {
        Ok(())
    }
}

fn main() {
    let bytes = vec![128, 255, 144];
    let mut array_wrapper = ArrayWrapper {
        buffer: vec![]
    };

    let _ = array_wrapper.write(&bytes);

    println!("{:?}", array_wrapper);
}
```

Massa, conseguimos implementar uma trait sem muita dificuldade, mas e a parte do desacoplamento, como funciona? Isto é,
como que fazemos pra escrever uma função que consiga trabalhar com qualquer struct que seja `Write` sem se importar como
ela é construída?

## Trait Objects

No começo do texto definimos que o nosso objetivo é criar uma função que pudesse escrever um pedaço de texto, uma
string, em qualquer mídia.

```rust
fn write(media: &mut ???, text: &str);
```

Agora que sabemos que traits podem nos ajudar com isso e também se valendo do nosso conhecimento prévio de outras
linguagens, podemos tranquilamente construir nossa função `write` sem se preocupar com o tipo de `media`, basta passar
qualquer coisa que implemente uma trait compatível com as nossas expectativas, nesse caso `Write`.

```rust
// ...

fn write(media: &mut Write, text: &str) {
    let text_bytes = text.as_bytes();
    let _ = media.write(text_bytes);
}

fn main() {
    let mut array_wrapper = ArrayWrapper {
        buffer: vec![]
    };

    write(&mut array_wrapper, "alô mundo!");

    println!("{:?}", array_wrapper);
}
```

Tudo parece fazer sentido, até tentarmos compilar o código.

```rust
Compiling playground v0.0.1 (/playground)
error[E0782]: trait objects must include the `dyn` keyword
  --> src/main.rs:22:22
   |
22 | fn write(media: &mut Write, text: &str) {
   |                      ^^^^^
   |
help: add `dyn` keyword before this trait
   |
22 | fn write(media: &mut dyn Write, text: &str) {
   |                      +++

For more information about this error, try `rustc --explain E0782`.
error: could not compile `playground` due to previous error
```

O compilador não está muito feliz com o fato de não termos usado a palavra reservada `dyn` na declaração dos parâmetros
da função. Vamos aplicar a correção e executar o código novamente pra ver o que acontece.

```rust
fn write(media: &mut dyn Write, text: &str) {
    // ...
}
```

```rust
Compiling playground v0.0.1 (/playground)
    Finished dev [unoptimized + debuginfo] target(s) in 1.21s
     Running `target/debug/playground`

ArrayWrapper { buffer: [97, 108, 195, 180, 32, 109, 117, 110, 100, 111, 33] }
```

Funcionou, mas o que está acontecendo aqui? Pra que serve essa palavra reservada no fim das contas?

### Dynamic Dispatch

É bom termos em mente que Rust é uma linguagem muitas vezes classificada como sendo
uma [systems language](https://en.wikipedia.org/wiki/System_programming_language), ou seja, uma linguagem que nos
permite interagir mais proximamente do hardware e do sistema operacional pra escrever código de alta performance. Em
contextos como esse é importante deixar explícitas quaisquer características da linguagem envolvendo tradeoffs que
afetem de alguma maneira a performance do programa. Ao tornar obrigatório o uso de `dyn` a linguagem está nos informando
que quando lidamos com trait objects estamos optando pelo uso
de [dynamic dispatch](https://en.wikipedia.org/wiki/Dynamic_dispatch) pra nossa operação polimórfica.

Antes de entender melhor que [tradeoffs](https://en.wikipedia.org/wiki/Trade-off) são esses, vamos dar um passo pra trás
e explicar o que realmente são trait objects e qual a sua relação com dynamic dispatch. Melhor ainda, vamos dar dois
passos pra trás e entender como funciona o mecanismo por trás de uma chamada de função. Não é muito complexo, pode
acreditar.

Quando nosso código é compilado pra linguagem de máquina todas as instruções que compõem um função são armazenadas numa
região especial da memória do programa. Em alto nível seria algo mais ou menos assim.

<p align="center">
    <img width="400" height="400" src="/img/functions-memory-layout.png">
</p>
<p class="caption">Imagine que os valores em hexadecimal são os bytes dos opcodes de cada um das funções.</p>

Quando invocamos uma função não-polimórfica o compilador sabe exatamente em que região de memória essa função estará
disponível, portanto pra executar esse código basta atualizar
o [contador de instruções](https://en.wikipedia.org/wiki/Program_counter) do programa pra esse endereço e deixar a
função ditar os rumos da execução da aplicação. Em resumo: uma chamada de função é um procedimento que podemos realizar
em um **único passo**.

Agora vamos dar um passo à frente e explicar, finalmente, o que são trait objects. Também não é muito complexo: trait
objects são referências pra traits. Só isso? Sim, essencialmente é só isso.

```rust
let mut array_wrapper = ArrayWrapper {
    buffer: vec![]
};

let media: &mut dyn Write = &mut array_wrapper;

write(media, "alô mundo!");
```

Mas o que é que isso tem de especial e qual a sua relação com dynamic dispatch? Bem, no caso de chamadas de funções
polimórficas que utilizam trait objects o compilador não tem como inferir o endereço das funções sendo executadas
através do trait object.

```rust
let _ = media.write(text_bytes);
```

Do ponto de vista da nossa função, o parâmetro `media` pode ser qualquer coisa que implemente `Write` portanto não temos
como saber em que lugar da memória `Write::write` se encontra.

Não estamos interessados em quem implementa a trait e por isso não temos como saber em tempo de compilação o endereço de
seus métodos pois não sabemos qual das muitas implementações possíveis estamos utilizando. O mecanismo de dynamic
dispatch serve pra resolver justamente isso. Através de dynamic dispatch o programa passa a manter uma tabela interna,
também conhecida como [vtable](https://en.wikipedia.org/wiki/Virtual_method_table), contendo os endereços de memória de
todas as funções da implementação de uma trait.

Um trait object é portanto um tipo especial de referência, um ponteiro mágico, que contem internamente um ponteiro pra
estrutura de dados que implementa a trait e também outro ponteiro pra vtable da implementação da trait correspondente
àquele tipo.

<img width="680" height="440" src="/img/rust-vtables.png">

É aí que o tradeoff se encontra. Pois se antes era possível executar uma função em apenas um único passo, agora
precisamos de **pelo menos dois**: acessar a vtable através do trait object e atualizar o contador de instruções pro
endereço definido na vtable. Além de um custo extra em memória também, pois agora cada implementação da nossa
trait `Write` terá uma tabela de endereços gerada em tempo de compilação, aumentando o tamanho do binário e
consequentemente o espaço que ocupa em memória.

Contudo isso não é o fim do mundo. Pra maioria das aplicações que escrevemos essa diferença é negligenciável. Acontece
que Rust tem exigências que algumas linguagens não tem e por isso, se você precisa otimizar cada ciclo de CPU, é
importante que a linguagem deixe evidente o custo que cada coisa tem. Se tomarmos, mais uma vez, Java como um exemplo,
dynamic dispatch é algo bastante prevalente e mesmo assim existem exemplos fartos de aplicações altamente performáticas
desenvolvidas nessa linguagem. Geralmente os gargalos estão em outras partes da aplicação.

## Conclusão

Tirando o aspecto mais baixo nível envolvendo dynamic dispatch e vtables, traits não são um conceito muito complexo e a
sintaxe é bem acessível pra quem já teve algum contato com linguagens mais imperativas e orientadas a objeto.

Existe uma série de outras coisas que ainda poderíamos abordar mas esses temas não são inerentes ao estudo de
polimorfismo e podem ser acessados sem muita dificuldade por uma leitura complementar, vou listá-los aqui:

* [Traits com implementações default](https://doc.rust-lang.org/book/ch10-02-traits.html#default-implementations).
* [Extendendo tipos usando traits](https://cmcenroe.me/2016/08/22/rust-extending-type.html).
* [Trait objects em um vector](https://dev.to/magnusstrale/rust-trait-objects-in-a-vector-non-trivial-4co5).

Como sempre, a [documentação oficial](https://doc.rust-lang.org/book/ch10-02-traits.html) é muito útil caso você queira
se aprofundar mais ainda no tema.

Espero que esse texto lhe tenha sido útil de alguma forma. Agradeço qualquer feedback que você queira compartilhar
comigo no twitter [@umbravirtual](http://twitter.com/umbravirtual).

:-)
