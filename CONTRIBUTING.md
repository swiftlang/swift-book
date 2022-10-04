# Contribuindo

Esse projeto tem como objetivo traduzir e disponibilizar gratuitamente em português o livro _The Swift Programming Language_.

Cada tópico do livro tem uma **issue**. Antes de iniciar a tradução de um tópico verifique na respectiva **issue** se alguém já não está traduzindo. Quando alguém pegar uma dessas `sessao/topico` para traduzir, terá o nome atribuído a Issue.

## Convenções

Para facilitar o processo de revisão e diminuir as inconsistências entre traduções, quem contribuir deverá seguir algumas convenções:

- Não traduzir palavras reservadas.
- Na dúvida entre traduzir ou não determinado termo, deixar o termo em inglês.
- Criar Branches no padrão `sessao/topico` e traduzir por _Sessão_ e _Tópico_ (textos mais internos dentro da _Sessão_).
- _Pull Requests_ devem ser feitos para `master`, marcando a Issue que resolvem.

## Criação de Branches

A tradução deverá ser feita através de Forks, com branches nomeadas por sessão e tópico, com letras minúsculas, seguindo o seguinte padrão:

`nome-da-sessao/nome-do-topico`

Exemplo:
`a-swift-tour/control-flow`

Caso não existam tópicos para determinada sessão, a branch deve ter somente o nome da sessão:
`nome-da-sessao`

## Pull Requests

Os Pull Requests devem ser feitos diretamente para a branch `master`, e utilizar o template do repositório específico para PRs. Após terminar uma tradução, lembre-se de marcá-la como traduzida no README. A revisão será feita avaliando a tradução em si, a coerência do texto da tradução, e a coerência com as traduções anteriores.

## Motivação

Existe pouco ou nenhum material gratuito em português de Swift. Contribuindo com a tradução desse livro, você pode estar ajudando alguém sem dinheiro para comprar material a estudar. Além disso, você mesmo estará estudando, revisando, e aprendendo recursos da linguagem que você talvez não conheça.

Com esforço coletivo, conseguiremos finalizar a tradução e facilitar o aprendizado de Swift para quem não sabe inglês.


## Guia de Tradução

Na tabela abaixo estão listadas algumas palavras importantes que aparecem com recorrência no livro e suas respectivas traduções para manter consistência. Se você encontrar uma palavra que acredite que deve ser incluída na tabela, submeta um _pull request_ com a adição.

| Original | Tradução |
| ---------| ---------|
| Tuple | Tupla |
| Array | _Array_ |
| Set | _Set_ |
| Dictionary | Dicionário |
| Class | Classe |
| Struct | Estrutura |
| Enumeration | Enumeração |
| Closure | - |
| Getter | - |
| Setter | - |
| Lazy Properties | Propriedades _Lazy_ |
| In-Out Parameters | Parâmetros _In-Out_ |


**Importante**: Palavras que não são traduzidas devem ser marcadas no texto em _itálico_.

```diff
- Closures are self-contained blocks of functionality that can be passed around and used in your code. Closures in Swift are similar to blocks in C and Objective-C and to lambdas in other programming languages.
+ _Closures_ são blocos autocontidos de funcionalidade que podem ser passados e usados em seu código. _Closures_ em Swift são semelhantes a blocos em C e Objective-C e _lambdas_ em outras linguagens de programação.
```

### Blocos de Código

Nos blocos de código somente os **comentários** devem ser traduzidos. Menções a palavras do código, como tipos, variáveis, e funções, devem ser marcadas com aspas simples.

```diff
let string1 = "hello"
let string2 = " there"
var welcome = string1 + string2
- // welcome now equals "hello there"
+ // 'welcome' agora é igual a "hello there"
```

### Projeto de Tradução Antigo

Há um [repositório arquivado](https://github.com/AcademyIFCE/swift-book-markdown) com um projeto de tradução antigo que está incompleto e foi descontinuado em favor desse projeto novo que é um _fork_ do [repositório oficial](https://github.com/apple/swift-book) com suporte ao [DocC](https://developer.apple.com/documentation/docc). O projeto antigo pode ser utilizado como base para a tradução mas é importante que o texto seja revisado porque pode haver diferenças no conteúdo ou a tradução pode não estar adequada às regras definidas nesse guia.
