

# Compatibilidade de versões

Este livro descreve o Swift 5.7,
a versão padrão do Swift que está incluída no Xcode 14.
Você pode usar o Xcode 14 para criar destinos
que são escritos em Swift 5.7, Swift 4.2 ou Swift 4.

@Comment {
  - test: `swift-version`
  
  ```swifttest
  >> #if swift(>=5.7.1)
  >>     print("Too new")
  >> #elseif swift(>=5.7)
  >>     print("Just right")
  >> #else
  >>     print("Too old")
  >> #endif
  << Just right
  ```
}

Quando você usa o Xcode 14 para compilar o código Swift 4 e Swift 4.2,
a maioria das funcionalidades do Swift 5.7 está disponível.
Dito isto,
as seguintes alterações estão disponíveis apenas para código que usa Swift 5.7 ou posterior:

- As funções que retornam um tipo opaco requerem o tempo de execução do Swift 5.1.
- A expressão `try?` não introduz um nível extra de opcionalidade
  para expressões que já retornam opcionais.
- Expressões de inicialização literais de inteiros grandes são inferidas
  para ser do tipo inteiro correto.
  Por exemplo, `UInt64(0xffff_ffff_ffff_ffff)` avalia o valor correto
  ao invés de transbordar.

A simultaneidade requer Swift 5.7 ou posterior,
e uma versão da biblioteca padrão Swift
que fornece os tipos de simultaneidade correspondentes.
Em plataformas Apple, defina um destino de implantação
de pelo menos iOS 13, macOS 10.15, tvOS 13 ou watchOS 6.

Um destino escrito em Swift 5.7 pode depender
de um destino escrito em Swift 4.2 ou Swift 4,
e vice versa.
Isso significa que, se você tiver um grande projeto
que é dividido em várias _frameworks_,
você pode migrar seu código do Swift 4 para o Swift 5.7
um _framework_ de cada vez.


@Comente {
Este arquivo de origem faz parte do projeto de código aberto Swift.org

Copyright (c) 2014 - 2022 Apple Inc. e os autores do projeto Swift
Licenciado sob Apache License v2.0 com exceção de biblioteca de tempo de execução

Consulte https://swift.org/LICENSE.txt para obter informações sobre a licença
Veja https://swift.org/CONTRIBUTORS.txt para a lista de autores de projetos Swift
}
