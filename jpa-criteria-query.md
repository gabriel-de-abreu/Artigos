# CriteriaQuery no JPA: mais controle sobre as queries sem precisar recorrer ao SQL nativo

O Spring Data JPA resolve muito bem a maioria das consultas de uma aplicação.

Para uma busca simples, algo como:

```java
List<Order> findByCustomerIdAndStatus(
    Long customerId,
    Status status
);
```

é praticamente tudo que precisamos.

O problema começa quando a consulta deixa de ser simples. É comum termos telas de pesquisa ou endpoints com vários filtros opcionais: cliente, status, vendedor, período, valores, região, produto, ordenação e paginação.

Nesse ponto, começam a aparecer duas alternativas.

A primeira é criar vários métodos no repository para cobrir as diferentes combinações:

```java
findByCustomerId(...)
findByCustomerIdAndStatus(...)
findByCustomerIdAndStatusAndSellerId(...)
findByCustomerIdAndStatusAndSellerIdAndRegion(...)
```

Obviamente, isso não escala muito bem.

A segunda alternativa costuma ser partir para uma `native query`, o que resolve o problema de flexibilidade, mas cria outro: passamos a trabalhar diretamente com a estrutura do banco.

É justamente nesse cenário que a Criteria API pode ser uma alternativa interessante.

## CriteriaQuery como uma camada intermediária

Com CriteriaQuery continuamos trabalhando dentro do JPA, mas ganhamos bastante controle sobre a consulta.

Uma query simples poderia ser construída assim:

```java
CriteriaBuilder cb = entityManager.getCriteriaBuilder();

CriteriaQuery<Order> query =
    cb.createQuery(Order.class);

Root<Order> order =
    query.from(Order.class);

query.select(order)
     .where(
         cb.equal(
             order.get("customerId"),
             customerId
         ),
         cb.equal(
             order.get("status"),
             status
         )
     );
```

É mais código do que um método do Spring Data. Para uma consulta simples, não vejo vantagem em usar CriteriaQuery.

A diferença começa a aparecer quando os parâmetros são dinâmicos.

Podemos adicionar os filtros conforme eles realmente existirem:

```java
List<Predicate> predicates = new ArrayList<>();

if (customerId != null) {
    predicates.add(
        cb.equal(
            order.get("customerId"),
            customerId
        )
    );
}

if (status != null) {
    predicates.add(
        cb.equal(
            order.get("status"),
            status
        )
    );
}

if (sellerId != null) {
    predicates.add(
        cb.equal(
            order.get("sellerId"),
            sellerId
        )
    );
}

query.where(
    cb.and(
        predicates.toArray(new Predicate[0])
    )
);
```

Nesse caso, a query final contém somente os filtros utilizados.

Isso é diferente de montar uma consulta genérica com várias condições desse tipo:

```sql
WHERE
    (:customerId IS NULL OR customer_id = :customerId)
AND (:status IS NULL OR status = :status)
AND (:sellerId IS NULL OR seller_id = :sellerId)
```

Além de deixar a consulta mais difícil de ler conforme ela cresce, esse tipo de abordagem pode dificultar a otimização do plano de execução em determinados bancos e cenários.

Com CriteriaQuery, podemos simplesmente não adicionar uma condição que não é necessária.

## O mesmo vale para joins

O controle não fica limitado aos `WHERE`.

Imagine que `Order` possua relacionamentos com `Customer`, `Seller` e `Product`.

Se o filtro por vendedor for opcional, podemos criar o join somente quando ele for necessário:

```java
if (sellerId != null) {
    Join<Order, Seller> seller =
        order.join("seller");

    predicates.add(
        cb.equal(
            seller.get("id"),
            sellerId
        )
    );
}
```

Se não existe filtro por vendedor, não precisamos necessariamente colocar esse relacionamento na consulta.

Isso pode ser relevante em consultas que possuem vários relacionamentos e tabelas com grande volume de dados.

Novamente, não significa que qualquer join adicional será ruim ou que removê-lo sempre deixará a query mais rápida. O plano de execução precisa ser analisado.

A vantagem aqui é ter controle para montar a consulta de acordo com o caso de uso.

## Fetch e o problema do N+1

Outro ponto bastante relevante no uso de CriteriaQuery é o controle sobre o carregamento dos relacionamentos.

Considere uma situação como:

```java
List<Order> orders = repository.findAll();

for (Order order : orders) {
    System.out.println(
        order.getCustomer().getName()
    );
}
```

Dependendo do mapeamento e da estratégia de carregamento, podemos ter uma query para buscar os pedidos e depois várias outras para buscar os clientes.

É o conhecido problema de N+1.

Com CriteriaQuery podemos utilizar `fetch` para indicar que determinado relacionamento deve ser carregado junto com a entidade principal:

```java
Root<Order> order =
    query.from(Order.class);

order.fetch(
    "customer",
    JoinType.LEFT
);
```

Nesse cenário, o Hibernate pode gerar uma consulta utilizando `JOIN`, trazendo `Order` e `Customer` na mesma operação.

Em vez de algo próximo de:

```sql
SELECT ...
FROM orders;

SELECT ...
FROM customer
WHERE id = ?;

SELECT ...
FROM customer
WHERE id = ?;

SELECT ...
FROM customer
WHERE id = ?;
```

podemos ter:

```sql
SELECT ...
FROM orders o
LEFT JOIN customer c
    ON c.id = o.customer_id;
```

A principal vantagem aqui é evitar as consultas adicionais necessárias para carregar o relacionamento.

Isso reduz round trips entre aplicação e banco e pode fazer bastante diferença quando estamos retornando uma lista com muitos registros.

Também é importante diferenciar `join` de `fetch`.

Um `join` pode ser utilizado simplesmente para filtrar:

```java
Join<Order, Customer> customer =
    order.join("customer");

predicates.add(
    cb.equal(
        customer.get("status"),
        CustomerStatus.ACTIVE
    )
);
```

Nesse caso, precisamos do relacionamento para construir a condição da consulta.

Já o `fetch` indica que queremos também carregar os dados desse relacionamento.

Na prática:

```text
JOIN
→ preciso do relacionamento para montar a consulta

FETCH
→ preciso carregar o relacionamento junto com o resultado
```

Essa diferença permite controlar melhor o que será carregado em cada caso de uso.

### Fetch também precisa ser utilizado com cuidado

É importante não transformar `fetch` em uma regra de "quanto mais, melhor".

Se fizermos `fetch` de uma coleção `@OneToMany`, por exemplo, podemos acabar multiplicando as linhas retornadas.

Imagine:

```text
Order 1 → Item 1
Order 1 → Item 2
Order 1 → Item 3
Order 2 → Item 4
Order 2 → Item 5
```

A consulta pode precisar retornar várias linhas para representar os relacionamentos.

Isso pode aumentar bastante o volume de dados e também criar problemas em determinadas consultas paginadas.

Por isso, vejo o `fetch` como uma ferramenta para resolver uma necessidade específica: quando sabemos que aquele relacionamento será utilizado e queremos evitar que ele seja carregado posteriormente através de novas queries.

A ideia não é fazer `fetch` de todos os relacionamentos da entidade.

## Projeções também fazem diferença

Outro ponto que muitas vezes é ignorado quando falamos de performance em JPA é a quantidade de dados que estamos trazendo.

Imagine uma entidade `Order` com diversas propriedades, mas uma API que precisa retornar somente:

```text
id
customerName
total
createdAt
```

Não necessariamente faz sentido carregar a entidade inteira.

Com CriteriaQuery podemos criar uma projeção diretamente para um DTO:

```java
CriteriaQuery<OrderSummary> query =
    cb.createQuery(OrderSummary.class);

Root<Order> order =
    query.from(Order.class);

query.select(
    cb.construct(
        OrderSummary.class,
        order.get("id"),
        order.get("customerName"),
        order.get("total"),
        order.get("createdAt")
    )
);
```

Nesse caso, o SQL pode selecionar somente as colunas necessárias.

Em consultas que retornam muitos registros, isso pode reduzir o volume de dados transferidos entre banco e aplicação, além do custo de criação e gerenciamento das entidades pelo Hibernate.

Em muitos casos, esse tipo de otimização é mais relevante do que tentar otimizar a própria construção da CriteriaQuery.

## CriteriaQuery não é automaticamente mais rápida

Aqui existe um ponto que vale deixar claro.

CriteriaQuery não é uma forma "mais rápida" de executar uma query.

O Hibernate ainda precisa transformar a CriteriaQuery em SQL:

```text
CriteriaQuery
      ↓
Hibernate
      ↓
SQL
      ↓
Database
```

O banco não sabe se aquela consulta veio de CriteriaQuery, JPQL ou de uma native query.

Portanto, não existe uma regra dizendo que CriteriaQuery será mais rápida que uma native query.

Uma native query bem construída pode ser mais eficiente.

Uma CriteriaQuery mal construída pode ser pior.

O ganho está principalmente na possibilidade de construir uma consulta mais específica para cada situação.

Se o SQL gerado for ruim, precisamos olhar para o SQL.

E aí entram `EXPLAIN`, `EXPLAIN ANALYZE`, índices, cardinalidade, joins, quantidade de dados retornados e o plano de execução escolhido pelo banco.

## Onde CriteriaQuery começa a fazer sentido

Para mim, existe uma divisão relativamente simples.

### Spring Data para consultas simples

Se a consulta cabe naturalmente em:

```java
findByCustomerIdAndStatus(...)
```

eu usaria o método do Spring Data.

É mais simples e mais fácil de manter.

### JPQL quando a consulta é relativamente estática

Se existe uma consulta um pouco mais complexa, mas que não precisa ser montada dinamicamente, `@Query` com JPQL pode ser suficiente.

Não existe necessidade de utilizar CriteriaQuery só porque a consulta tem alguns joins.

### CriteriaQuery para consultas dinâmicas

Aqui é onde vejo mais valor.

Principalmente quando temos:

* muitos filtros opcionais;
* joins condicionais;
* ordenação dinâmica;
* projeções;
* paginação;
* diferentes combinações de filtros;
* necessidade de controlar quais relacionamentos serão carregados.

Nesses casos, construir a query de forma programática começa a ser mais interessante do que tentar representar todas as possibilidades através de métodos do repository.

### Native Query quando precisamos realmente do SQL

Também não vejo problema em utilizar native query quando ela é a melhor solução.

Existem consultas que simplesmente não se encaixam bem no modelo do JPA.

CTEs, window functions, recursos específicos do banco e algumas consultas analíticas são exemplos em que o SQL nativo pode ser a opção mais adequada.

O problema não é usar native query.

__O problema é começar a utilizar SQL nativo para qualquer consulta que ficou um pouco mais complexa__.

## E o código da CriteriaQuery?

Essa é provavelmente a principal desvantagem.

Uma CriteriaQuery grande pode ficar bastante verbosa.

Por isso, em aplicações Spring Data, uma abordagem com `Specification` pode ajudar bastante.

Podemos separar cada filtro:

```java
public static Specification<Order> hasStatus(
    Status status
) {
    return (root, query, cb) ->
        status == null
            ? null
            : cb.equal(
                root.get("status"),
                status
            );
}
```

E depois combinar:

```java
Specification<Order> specification =
    Specification
        .where(hasCustomer(customerId))
        .and(hasStatus(status))
        .and(hasSeller(sellerId));
```

Dessa forma, a complexidade da consulta deixa de ficar concentrada em um único método.

Cada filtro pode ser implementado e testado separadamente.

Para sistemas que possuem várias consultas com filtros semelhantes, isso tende a ser mais fácil de manter.

## No final, o que importa é o SQL

Quando o assunto é performance em JPA, é fácil cair na discussão de qual API é "mais rápida".

Não acho que essa seja a melhor forma de analisar o problema.

A pergunta mais importante é:

> Qual SQL está chegando ao banco?

E, depois:

> Qual plano de execução o banco está utilizando?

CriteriaQuery ajuda justamente a ter mais controle sobre a primeira parte.

Podemos decidir quais filtros entram, quais joins são necessários, quais relacionamentos devem ser carregados com `fetch` e quais campos realmente precisam ser retornados.

Isso não garante performance.

Mas evita algumas situações bastante comuns em aplicações JPA: queries genéricas demais, joins desnecessários, carregamento excessivo de entidades e principalmente consultas adicionais causadas por relacionamentos que não foram planejados.

No fim, vejo CriteriaQuery menos como uma alternativa "mais rápida" ao Spring Data ou ao SQL nativo e mais como uma ferramenta para quando o nível de abstração padrão começa a limitar o controle que precisamos ter sobre a consulta.

Para consultas simples, continue usando o Spring Data.

Quando a consulta começa a ficar dinâmica, CriteriaQuery e `Specification` são boas opções.

E quando o problema exige controle específico do banco, use SQL nativo.

O importante é não tratar nenhuma dessas abordagens como solução universal. O melhor resultado normalmente vem de escolher a ferramenta de acordo com a complexidade e o comportamento que aquela consulta realmente precisa ter.
