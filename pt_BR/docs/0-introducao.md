# 0 - Introdução

Quando falamos em dar o próximo passo como programador, precisamos entender que a qualidade do código é sempre o foco principal.

Eu gosto de pensar que o que diferencia o Júnior do Pleno é a preocupação com a qualidade da entrega. O Júnior tem a missão de **FAZER FUNCIONAR**, enquanto o Pleno faz funcionar mirando a **QUALIDADE DA ENTREGA**.

Existem várias ideias para melhorar o seu código, como o KISS (Keep It Simple, Stupid), o DRY (Don't Repeat Yourself) e o que vamos abordar neste tutorial: o SOLID.

O SOLID é um acrônimo de cinco princípios da Programação Orientada a Objetos que facilitam a vida de quem vai LER o código na próxima manutenção. Eles funcionam quase como leis que você segue para ter um código mais legível e fácil de manter. Esses princípios foram reunidos por Robert C. Martin, também conhecido como Uncle Bob, e o nome SOLID veio depois, criado por Michael Feathers.

O acrônimo é formado por:

* **S** — Single Responsibility Principle (Princípio da Responsabilidade Única)
* **O** — Open-Closed Principle (Princípio Aberto-Fechado)
* **L** — Liskov Substitution Principle (Princípio da Substituição de Liskov)
* **I** — Interface Segregation Principle (Princípio da Segregação de Interfaces)
* **D** — Dependency Inversion Principle (Princípio da Inversão de Dependência)

Vamos usar um projeto bem simples, um CHAT no estilo da Twitch, para explicar todos os princípios.

> Os exemplos usam **PHP 8.3+** e **Laravel 11+**. Se você usa outra linguagem, relaxa: as ideias são as mesmas.

---

## Navegação

[1 – Single Responsibility Principle](1-srp.md) • [2 – Open-Closed Principle](2-ocp.md) • [3 – Liskov Substitution Principle](3-lsp.md) • [4 – Interface Segregation Principle](4-isp.md) • [5 – Dependency Inversion Principle](5-dip.md)
