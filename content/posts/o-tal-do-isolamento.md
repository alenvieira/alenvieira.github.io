---
title: "O tal do Isolamento no A.C.I.D."
date: 2024-06-02T09:38:53-03:00
description: "Vamos aprofundar um pouco sobre o isolamento?"
draft: false
---
>A solidão é estar sozinho; o isolamento é achar-se só no mundo quando se está só. Diane de Beausacq

Isolamento no A.C.I.D. é a garantia responsável por gerenciar a concorrência entre as transações e verificar como as transações simultâneas podem afetar umas às outras. Como tratei [no artigo passado](/posts/explicando-acid/).  

Vamos utilizar Python com algumas bibliotecas para trabalhar com PostgreSQL em um container, como abordado no artigo anterior. Realizaremos diversos testes para demonstrar diferentes cenários de isolamento. Os detalhes podem ser encontrados no repositório.  

Vale lembrar que, ao utilizar conexões criadas com a estrutura "with", todas as operações serão confirmadas (commit) se concluídas com sucesso durante a conexão; caso contrário, tudo será revertido (rollback). Em ambos os casos, a conexão será encerrada automaticamente.  

    "Ok! Cara, está tudo muito confuso! Pode me dar um exemplo? Como podemos identificara importância do isolamento?"  

Certo, vamos lá: imagine que seu gerente está considerando te dar um aumento ao mesmo tempo em que a empresa está realizando um reajuste salarial global. Se ambas as operações forem executadas ao mesmo tempo, diferentes cenários podem afetar seu salário:  

- No melhor cenário, você recebe os aumentos na ordem que mais te beneficie.
- Um dos aumentos pode ser perdido se um sobrepuser o outro.
- Um dos aumentos pode gerar um erro.  

Isso depende de como está configurada a visibilidade de uma transação em relação à outra.  

Vamos aprender um pouco mais sobre isso começando pelos níveis de isolamento existentes.  

## Níveis de isolamento

O padrão SQL define quatro níveis de isolamento:

- **Read uncommitted**(Leitura não commitada): Nível menos restritivo que permite ler dados de outras transações que ainda não foram commitados ou confirmados, podendo levar a dirty reads (leituras sujas), non-repeatable reads (leituras não repetíveis), lost updates (atualizações perdidas), read skew (distorções de leitura), write skew (distorções de escrita) e phantom reads (leituras fantasmas).
- **Read committed**(Leitura commitada): Nível onde a transação pode ler apenas dados de transações que foram commitadas ou confirmadas, evitando leituras sujas, mas ainda permitindo non-repeatable reads, lost updates, read skew, write skew e phantom reads.
- **Repeatable read**(Leitura repetida): Nível que garante que os dados lidos durante a transação não serão alterados até que a transação seja concluída, evitando leituras non-repeatable reads, mas permitindo fenômenos de lost updates, read skew, write skew e phantom reads. 
- **Serializable**(Serializável): Nível de isolamento mais restritivo, que garante que as transações sejam completamente isoladas umas das outras, evitando qualquer interferência e fenômenos como dirty reads, non-repeatable reads, lost updates, read skew, write skew e phantom reads.



## Principais Fenômenos

Alguns fenômenos podem ser causados por interferência das transações executadas simultaneamente, dependendo da configuração do nível do isolamento. Abaixo, vemos alguns dos fenômenos em que níveis de isolamento podem ocorrer:

<table>
    <thead>
        <tr>
            <th rowspan="2" style="border:1px solid #000; text-align:center; vertical-align:middle;">Nível de isolamento
            </th>
            <th colspan="6" style="border:1px solid #000; text-align:center; vertical-align:middle;">Fenômenos</th>
        </tr>
        <tr>
            <th style="border:1px solid #000; text-align:center; vertical-align:middle;">Dirty Read</th>
            <th style="border:1px solid #000; text-align:center; vertical-align:middle;">Nonrepeatable Read</th>
            <th style="border:1px solid #000; text-align:center; vertical-align:middle;">Lost Update</th>
            <th style="border:1px solid #000; text-align:center; vertical-align:middle;">Read Skew</th>
            <th style="border:1px solid #000; text-align:center; vertical-align:middle;">Write Skew</th>
            <th style="border:1px solid #000; text-align:center; vertical-align:middle;">Phantom Read</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;"><b>Read uncommitted</b></td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Pode ocorrer, mas <b>não ocorre</b> no PostgreSQL*</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Pode ocorrer</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Pode ocorrer</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Pode ocorrer</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Pode ocorrer</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Pode ocorrer</td>
        </tr>
        <tr>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;"><b>Read committed</b></td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Não ocorre</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Pode ocorrer</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Pode ocorrer</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Pode ocorrer</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Pode ocorrer</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Pode ocorrer</td>
        </tr>
        <tr>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;"><b>Repeatable read</b></td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Não ocorre</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Não ocorre</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Não ocorre</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Não ocorre</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Não ocorre, mas <b>pode ocorrer</b> no PostgreSQL*</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Pode ocorrer, mas <b>não ocorre</b> no PostgreSQL*</td>
        </tr>
        <tr>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;"><b>Serializable</b></td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Não ocorre</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Não ocorre</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Não ocorre</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Não ocorre</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Não ocorre</td>
            <td style="border:1px solid #000; text-align:center; vertical-align:middle;">Não ocorre</td>
        </tr>
    </tbody>
</table>

**Tabela dos principais fenômenos nos diferentes níveis de isolamento.** Fonte: Compilação do autor a partir de [_Tabela de Níveis de isolamento de transação da Documentação do PostgreSQL_](https://www.postgresql.org/docs/16/transaction-iso.html#MVCC-ISOLEVEL-TABLE), do artigo [_A Critique of ANSI SQL Isolation Levels_](https://arxiv.org/pdf/cs/0701157.pdf) e do post [_A beginner’s guide to Read and Write Skew phenomena_](https://vladmihalcea.com/a-beginners-guide-to-read-and-write-skew-phenomena/).

**\*** Dependendo do banco de dados utilizado, suas implementações sobre as garantias do isolamento podem ser ligeiramente diferentes. Como vamos utilizar o PostgreSQL, resolvi incorporar as informações específicas desse banco de dados que diferem em três pontos com relação ao fenômeno e nível de isolamento que não acompanham o padrão SQL.

### Dirty Read

Dirty Read ou Leitura Suja é o fenômeno em que a transação pode ler dados que ainda não foram commitados ou confirmados. No PostgreSQL, onde irei executar os testes, o nível de isolamento **Read Uncommitted** não é realmente implementado. Ele está presente apenas por compatibilidade, e não é possível reproduzir esse fenômeno no nesse banco de dados porque o nível de isolamento real é **Read Committed**.

```python
def test_without_dirty_read(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("SHOW TRANSACTION ISOLATION LEVEL")
            default_transaction_isolation, = cur.fetchone()
            cur.execute("SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED")
            cur.execute("SHOW TRANSACTION ISOLATION LEVEL")
            new_transaction_isolation, = cur.fetchone()
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("John Smith", 2500))
            id_employee, = cur.fetchone()
            
    def transaction1():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (3500, id_employee))
                time.sleep(2)
    
    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            global dirty_read_salary
            with conn.cursor() as cur:
                time.sleep(1)
                cur.execute("SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED")
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                dirty_read_salary, = cur.fetchone()
    
    threads = [Thread(target=transaction1), Thread(target=transaction2)]
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()         

    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
            new_salary, = cur.fetchone()            
    
    self.assertEqual(default_transaction_isolation, "read committed")
    self.assertEqual(new_transaction_isolation, "read uncommitted")
    self.assertEqual(dirty_read_salary, 2500)
    self.assertEqual(new_salary, 3500)
```

Nesse teste, primeiramente verificamos qual é o nível de isolamento padrão no PostgreSQL e fizemos uma operação para mudar o nível de isolamento. Após isso, executamos um cenário de dirty read com os seguintes passos entre as transações:

![A imagem mostra um diagrama de sequência com três colunas rotuladas como T1, Database e T2. As interações são as seguintes: Na coluna T1: Uma ação rotulada como "Atualiza salário do funcionário" envia uma seta para a coluna Database. Outra ação rotulada como "Confirma transação" envia uma seta para a coluna Database. Na coluna T2: Uma ação rotulada como "Seleciona funcionário" envia uma seta para a coluna Database.
O diagrama ilustra uma sequência onde T1 atualiza e confirma a transação do salário de um funcionário no banco de dados, enquanto T2 seleciona um funcionário do banco de dados. Existe uma relação temporal explicitada na imagem na qual T2 inicia-se após T1, mas é finalizada antes de T1.](/static/images/dirty_read.svg)

- Primeira transação(T1) começa realizando uma operação de atualização do salário de um funcionário.
- A segunda transação(T2) seleciona o salário do mesmo funcionário. Ela tem seu inicio e é confirmação antes do término de T1.
- Por fim, é confirmado/commitado a transação T1 após a finalização de T2.

No nível de isolamento **Read Uncommitted**, a segunda transação deveria ler o valor não confirmado/commitado da primeira transação, resultando no fenômeno de dirty read. No entanto, isso não ocorre no PostgreSQL, como podemos confirmar nas asserções. As asserções verificam as seguintes questões:

- O nível de isolamento padrão no PostgreSQL é **Read Committed**;
- Verifica-se a mudança no nível de isolamento da transação;
- Há confirmação de que não houve a leitura suja;
- E que a alteração do salário realmente ocorreu.

Por fim, verificamos nos testes que o nível de isolamento padrão é **Read Committed** e que o fenômeno Dirty Read não acontece no PostgreSQL. Sendo uma questão específica de cada banco de dados.  

### Nonrepeatable Read

Nonrepeatable Read ou Leitura Não Repetível, também conhecido por Fuzzy Read, é um fenômeno em que uma transação lê um registro duas vezes e obtém resultados diferentes. Pode ocorrer no PostgreSQL quando está no nível de isolamento **Read Committed**, mas não ocorre no **Repeatable Read**. Na imagem abaixo, mostramos o funcionamento desse fenômeno:

![image](/static/images/nonrepeatable_read.svg)
- Em T2, seleciona o funcionário;
- Em T1, atualiza o salário do funcionário;
- Por fim, seleciona o funcionário novamente.

Para exercitar a situação, fizemos dois testes. Um teste com o nível de isolamento **Read Committed** para ver o fenômeno e outro com o nível de isolamento **Repeatable Read** onde não ocorre.

```python
def test_norepeatable_read_with_isolation_level_read_committed(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Beth Lee", 3500))
            id_employee, = cur.fetchone()

    def transaction1():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                time.sleep(1)
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (4500, id_employee))                  
    
    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            global norepeatable_read_first_salary, norepeatable_read_second_salary
            with conn.cursor() as cur:
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                norepeatable_read_first_salary, = cur.fetchone()
                time.sleep(2)
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                norepeatable_read_second_salary, = cur.fetchone()

    threads = [Thread(target=transaction1), Thread(target=transaction2)]
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()         

    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
            new_salary, = cur.fetchone()
            
    self.assertNotEqual(norepeatable_read_first_salary, norepeatable_read_second_salary)
    self.assertEqual(new_salary, 4500)
```

Nesse teste acima, vemos o fenômeno da leitura não repetível acontecendo. O valor do salário do funcionário da primeira consulta difere do salário da segunda consulta do mesmo funcionário porque entre as duas consultas houve modificação do salário.

```python
def test_without_norepeatable_read_with_isolation_level_repeatable_read(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Dep Tunner", 3000))
            id_employee, = cur.fetchone()

    def transaction1():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
                time.sleep(1)
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (4000, id_employee))
    
    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            global without_norepeatable_read_first_salary, without_norepeatable_read_second_salary
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                without_norepeatable_read_first_salary, = cur.fetchone()
                time.sleep(2)
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                without_norepeatable_read_second_salary, = cur.fetchone()
    
    threads = [Thread(target=transaction1), Thread(target=transaction2)]
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()         

    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
            new_salary, = cur.fetchone()
            
    self.assertEqual(without_norepeatable_read_first_salary, without_norepeatable_read_second_salary)
    self.assertEqual(without_norepeatable_read_first_salary, 3000)
    self.assertEqual(new_salary, 4000)
```

Já nesse outro teste, não temos o fenômeno ocorrendo porque o nível de isolamento **Repeatable Read** impede isso e dentro dessa transação as leituras são repetíveis. O banco de dados resolveu ignorar que o valor foi atualizado.

Perceba que isso não foi considerado um erro. Dependendo do contexto, ter o dado mais atualizado é fundamental para a regra de negócio. Em outros casos, é melhor a consistência de dados mais fiel possível. Observe onde esse fenômeno pode ser tolerável.

### Lost Update

Lost Update ou Atualização Perdida, é um fenômeno quando duas transações tentam atualizar o mesmo registro e um transação acaba perdendo as informações da outra. Ele pode ocorrer no PostgreSQL quando está no nível de isolamento **Read Committed**, mas não ocorre no **Repeatable Read**. Na imagem a seguir vemos um exemplo desse fenômeno:

![image](/images/lost_update.svg)
- Em T1, seleciona o funcionário;
- Em T2, seleciona o mesmo funcionário e logo depois atualiza o salário do funcionário;
- Por fim, em T1, atualiza o salário do funcionário.

Novamente criamos dois testes: um teste com o nível de isolamento **Read Committed** para ver o fenômeno e outro com o nível de isolamento **Repeatable Read** onde não ocorre.

```python
def test_lost_update_with_isolation_level_read_committed(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Mary Castle", 4000))
            id_employee, = cur.fetchone()

    def transaction1():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                salary, = cur.fetchone()
                time.sleep(3)
                new_salary = salary * 1.1
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (new_salary, id_employee))
    
    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                time.sleep(2)
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                salary, = cur.fetchone()
                new_salary = salary * 1.2
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (new_salary, id_employee))
    
    threads = [Thread(target=transaction1), Thread(target=transaction2)]
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()
    
    with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                salary, = cur.fetchone()
    
    self.assertEqual(salary, 4400)
```

O fenômeno da atualização perdida acontece no teste acima. A primeira transação não obteve as mudanças do salário da segunda transação e sobrescreveu a informação da segunda.

```python
def test_without_lost_update_with_isolation_level_repeatable_read(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Bob Fox", 4000))
            id_employee, = cur.fetchone()

    def transaction1():
        global cm_lost_update
        with self.assertRaises(Exception) as cm_lost_update, psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                salary, = cur.fetchone()
                time.sleep(3)
                new_salary = salary * 1.1
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (new_salary, id_employee))
    
    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
                time.sleep(2)
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                salary, = cur.fetchone()
                new_salary = salary * 1.2
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (new_salary, id_employee))
    
    threads = [Thread(target=transaction1), Thread(target=transaction2)]
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()
    
    with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                salary, = cur.fetchone()
    
    self.assertEqual(salary, 4800)
    self.assertEqual(type(cm_lost_update.exception), psycopg.errors.SerializationFailure)
```

Já nesse segundo teste, o banco de dados com o nível de isolamento **Repeatable Read** identifica o problema e retorna um erro de serialização. Assim, impedindo do fenômeno ocorrer.

Em vez de causar um problema na integridade dos dados, é preferível obter um erro ou idenficar a situação e tentar novamente a transação com os dados mais consistentes. Deixando a base de dados íntegra. Há outras maneiras de tratar esse fenômeno, a ser discurtido num próximo post.

### Read Skew

Read Skew ou Leitura Distorcida é um fenômeno onde as transações executam consultas sobre dados que podem sofrer alterações e retornam dados inconsistentes. Ele pode ocorrer no PostgreSQL quando está no nível de isolamento Read Committed, mas não ocorre no Repeatable Read. 

![image](/images/read_skew.svg)
- T1 quer selecionar o funcionário 1 e 2;
- No meio da seleção dos dois funcionários, T2 atualiza os salários deles;
- Por fim, T1 tem um versão distorcida dos dados: a informação do funcionário 1 desatualizada e a informação do funcionário 2 atualizada.

Para simular o fenômeno, temos dois testes: um com o nível de isolamento **Read Committed** que acontece e outro com **Repeatable Read** onde não ocorre.

```python
def test_read_skew_with_isolation_level_read_committed(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Selma Bates", 4000))
            id_employee1, = cur.fetchone()
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Samuel Bowen", 4000))
            id_employee2, = cur.fetchone()

    def transaction1():
        global read_skew_salary_employee1, read_skew_salary_employee2
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee1,))
                read_skew_salary_employee1, = cur.fetchone()
                time.sleep(2)
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee2,))
                read_skew_salary_employee2, = cur.fetchone()

    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                time.sleep(1)
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (5000, id_employee1))
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (5000, id_employee2))
    
    threads = [Thread(target=transaction1), Thread(target=transaction2)]
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()
    
    with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee1,))
                new_salary_employee1, = cur.fetchone()
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee2,))
                new_salary_employee2, = cur.fetchone()
    
    self.assertEqual(read_skew_salary_employee1, 4000)
    self.assertNotEqual(read_skew_salary_employee2, 4000)
    self.assertEqual(new_salary_employee1, 5000)
    self.assertEqual(new_salary_employee2, 5000)
```

Vemos no teste acima que o salário do primeiro funcionário ainda é o mesmo do inicial, mas o do segundo já é atualizado. Sendo demonstrado assim, o fenômeno e confirmando que ambos os salários foram atualizados.

```python
def test_without_read_skew_with_isolation_level_repeatable_read(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Eric Wilson", 4000))
            id_employee1, = cur.fetchone()
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Roxy Clark", 4000))
            id_employee2, = cur.fetchone()

    def transaction1():
        global without_read_skew_salary_employee1, without_read_skew_salary_employee2
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee1,))
                without_read_skew_salary_employee1, = cur.fetchone()
                time.sleep(2)
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee2,))
                without_read_skew_salary_employee2, = cur.fetchone()

    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
                time.sleep(1)
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (5000, id_employee1))
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (5000, id_employee2))
    
    threads = [Thread(target=transaction1), Thread(target=transaction2)]
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()
    
    with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee1,))
                new_salary_employee1, = cur.fetchone()
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee2,))
                new_salary_employee2, = cur.fetchone()
    
    self.assertEqual(without_read_skew_salary_employee1, 4000)
    self.assertEqual(without_read_skew_salary_employee2, 4000)
    self.assertEqual(new_salary_employee1, 5000)
    self.assertEqual(new_salary_employee2, 5000)
```
No teste com o nível de isolamento **Repeatable Read**, o banco de dados intervém e mantém ambos os valores antigos e sem atualização. Isso pode ser necessário caso a funcionalidade realmente precise dos dados consistentes.

Esse fenômeno também ocorre quando há consultas em mais de uma tabela. No caso, ao consultar na segunda tabela ocorre alteração no dado da primeira tabela, assim ficando inconsistentes os dados. Por questão de simplicidade resolvemos mostrar apenas usando uma tabela.

Tenha em mente que é difícil manter a sincronia dos dados de forma consistente ao ter vários relacionamentos entre tabelas. Sempre avaliar ao máximo, as possibilidades ao se consultar algo vital ao negócio.

### Write Shew

Write Skew ou Escrita Distorcida, também chamado de Serialization Anomaly ou Anomalia na serialização, é um fenômeno que ocorre quando transações modificam um dado baseado em uma leitura de dados que já não é mais a mesma. Esse é um caso particular do PostgreSQL, no padrão SQL diz que não deveria ocorrer quando está no nível de isolamento **Repeatable Read**, mas ocorre. Somente no nível **Serializable** não ocorre.

![image](/static/images/write_skew.svg)
- Em T1, seleciona o total dos salários dos funcionários;
- Em T2, seleciona o total dos salários dos funcionários e já atualiza o salário do funcionário 1 com base nisso;
- Por fim, em T1, atualiza o salário do funcionário 2 com base na consulta que realizou que já não é consitente porque houve uma atualização no salário do funcionário 2;

Ambas transações consultam o mesmo dado, mas atualizam os salários de funcionários diferentes. O fenômeno acontece porque durante a atualização do salário do funcionário 1 em T1, o valor da consulta do total dos salários já foi alterado por T2. Nos testes abaixo, vemos um caso do fenômeno acontecendo e no outro o nível de isolamento impedindo de acontecer.   

```python
def test_write_skew_with_isolation_level_repeatable_read(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Selma Bates", 5000))
            id_employee1, = cur.fetchone()
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Samuel Bowen", 5000))
            id_employee2, = cur.fetchone()

    def transaction1():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
                cur.execute("SELECT SUM(salary) FROM employee")
                total_salary, = cur.fetchone()
                increment_salary = 0.1 * total_salary
                time.sleep(2)
                cur.execute("UPDATE employee SET salary=salary+%s WHERE id=%s", (increment_salary, id_employee1))

    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
                time.sleep(1)
                cur.execute("SELECT SUM(salary) FROM employee")
                total_salary, = cur.fetchone()
                increment_salary = 0.1 * total_salary
                cur.execute("UPDATE employee SET salary=salary+%s WHERE id=%s", (increment_salary, id_employee2))
    
    threads = [Thread(target=transaction1), Thread(target=transaction2)]
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()
    
    with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee1,))
                new_salary_employee1, = cur.fetchone()
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee2,))
                new_salary_employee2, = cur.fetchone()
    
    self.assertEqual(new_salary_employee1, 6000)
    self.assertEqual(new_salary_employee2, 6000)
```

No teste acima, os salários dos funcionários são incrementados com 10% do total dos salários deles. Se fossem executados sequencialmente, o segundo funcionário iria se beneficiar do aumento do salário do primeiro e teria um aumento maior. O fenômeno aconteceu e ambos ficam com o mesmo incremento.
 
```python
def test_without_write_skew_with_isolation_level_serializable(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Dean Knox", 5000))
            id_employee1, = cur.fetchone()
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Madison Frey", 5000))
            id_employee2, = cur.fetchone()

    def transaction1():
        global cm_write_skew
        with self.assertRaises(Exception) as cm_write_skew, psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
                cur.execute("SELECT SUM(salary) FROM employee")
                total_salary, = cur.fetchone()
                increment_salary = 0.1 * total_salary
                time.sleep(2)
                cur.execute("UPDATE employee SET salary=salary+%s WHERE id=%s", (increment_salary, id_employee1))
                
    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
                time.sleep(1)
                cur.execute("SELECT SUM(salary) FROM employee")
                total_salary, = cur.fetchone()
                increment_salary = 0.1 * total_salary
                cur.execute("UPDATE employee SET salary=salary+%s WHERE id=%s", (increment_salary, id_employee2))
    
    threads = [Thread(target=transaction1), Thread(target=transaction2)]
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()
    
    with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee1,))
                new_salary_employee1, = cur.fetchone()
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee2,))
                new_salary_employee2, = cur.fetchone()

    self.assertEqual(type(cm_write_skew.exception), psycopg.errors.SerializationFailure)        
    self.assertEqual(new_salary_employee1, 5000)
    self.assertEqual(new_salary_employee2, 6000)
```

Nesse teste acima, o banco de dados com o nível de isolamento mais agressivo percebe o caso de escrita distorcida e impede um dos funcionários de ter o aumento. Um erro de serialização é lançado ao invés de ser incrementado o salário. O que faz sentido, a regra do incremento é os 10% do total dos salários atualizados. Pode tentar fazer a transação e concluir os aumentos de salários.

Esse fenômeno, assim como a leitura distorcida, também ocorre quando há consultas em mais de uma tabela. Sendo necessário atentar as possíveis possibilidades e ver se são toleráveis nas regras de negócio.

### Phanton Read

Phantom Read ou Leitura fantasma, ocorre quando uma transação depende de dados que podem ser modificados por outras transações. Segundo o Padrão SQL, no nível Repeatable Read esse fenômeno poderia ocorrer, mas no PostgreSQL não ocorre. Podemos verificar esse fenômeno no nível Read committed. Um exemplo desse fenômeno está demonstrado na figura abaixo:

![image](/images/phantom_read.svg)
- Em T1, seleciona o total dos salários dos funcionários;
- Em T2, insere um funcionário;
- Por fim, em T1, seleciona novamente o total dos salários dos funcionários.

Dois testes foram criados para verificar esse fenômeno em diferentes nível de isolamento: um teste com o nível de isolamento com **Read Committed** e outro com **Repeatable Read**.

```python
def test_phantom_read_with_isolation_level_read_committed(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s)", ("Alan Rock", 2500))
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s)", ("Jess Tex", 3000))

    def transaction1():
        global phantom_read_first_sum_salary, phantom_read_second_sum_salary
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT SUM(salary) FROM employee")
                phantom_read_first_sum_salary, = cur.fetchone()
                time.sleep(2)
                cur.execute("SELECT SUM(salary) FROM employee")
                phantom_read_second_sum_salary, = cur.fetchone()
    
    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                time.sleep(1)
                cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s)", ("Emma Crow", 3500))
    
    threads = [Thread(target=transaction1), Thread(target=transaction2)]
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()
    
    with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT SUM(salary) FROM employee")
                sum_salary,  = cur.fetchone()
    
    self.assertNotEqual(phantom_read_first_sum_salary, phantom_read_second_sum_salary)
    self.assertEqual(sum_salary, 9000)
```

O teste acima acontece o fenômeno da leitura fantasma, pois entre a primeira consulta da soma dos salários e a segunda consulta da soma dos salários ocorre uma inserção de um novo funcionário.

```python
def test_without_phanton_read_with_isolation_level_repeatable_read(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s)", ("Rosie Cole", 2500))
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s)", ("Iggy Bell", 3000))

    def transaction1():
        global without_phanton_read_first_sum_salary, without_phanton_read_second_sum_salary
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
                cur.execute("SELECT SUM(salary) FROM employee")
                without_phanton_read_first_sum_salary, = cur.fetchone()
                time.sleep(2)
                cur.execute("SELECT SUM(salary) FROM employee")
                without_phanton_read_second_sum_salary, = cur.fetchone()
    
    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
                time.sleep(1)
                cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s)", ("Hugo Cash", 3500))

    
    threads = [Thread(target=transaction1), Thread(target=transaction2)]
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()
    
    with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT SUM(salary) FROM employee")
                sum_salary,  = cur.fetchone()
    
    self.assertEqual(without_phanton_read_first_sum_salary, without_phanton_read_second_sum_salary)
    self.assertEqual(sum_salary, 9000)
```

O PostgreSQL com o nível de isolamento **Repeatable Read** impede o fenômeno da leitura fantasma ocorrer, como podemos verificar no teste acima. Uma questão importante a se ressaltar é que isso acontece na inserção e deleção de registros na qual trabalhamos.

Esse caso é bem semelhante ao da leitura não repetível, com a diferença da leitura não repetível, com a diferença que nesse fenômeno trabalha sobre um conjunto de dados e na leitura não repetível trabalha com o mesmo registro. Novamente, devemos avaliar que é aceitável não ter a informação mais atualizada, se isso impacta nas funcionalidades da sua aplicação.

## Algumas observações

Não é porque o nível de isolamento **Serializable** impede todos esses fenômenos de ocorrer que você deveria adotar isso como padrão nas suas transações, isso pode ser muito custoso para o banco de dados e afetar o desempenho de seu sistema. Lembre-se de que o isolamento é uma escolha entre uma maior consistência ou uma maior concorrência. Devemos observar diversas questões:

- Tratamento dos erros ou conflitos se houver;
- Algum limite de tempo da aplicação ou de negócio;
- Retentativa de alguma operação;
- Observar a documentação sobre o isolamento de seu banco de dados;

Existem algumas abordagens que evitam ou detectam alguns desses casos. Os protocolos mais comuns são:
- [Two-phase locking (P2L)](https://en.wikipedia.org/wiki/Two-phase_locking) mais conhecido como bloqueio pessimista se utilizando de alguma trava ou bloqueio de algum recurso;
- [Optimistic concurrency control (OOC)](https://en.wikipedia.org/wiki/Optimistic_concurrency_control) mais connhecido como bloqueio otimista que deixam a operação acontecer até que que detecta um conflito e transação é revertida.

Ambos protocolos tratam diversos esses fenômenos.

Eles são eficazes em cenários de conflito intenso, pois evitam perda de tempo abortando e repetindo transações.

Protocolos otimistas são melhores em cenários de leitura intensa, pois evitam a carga de manipulação de bloqueios e resolução de possíveis impasses..

## E por hoje é só, pessoal!

Vimos mais sobre as garantias do isolamento, seus níveis e fenômenos que podem ocorrer e é bom estar atento para que sua aplicação/programa não fique com os dados inconsistentes. Assim como no artigo passado, abaixo tem links bem interessantes que possam complementar sua leitura. Até o próximo artigo.

[Deeply understand Isolation levels and Read phenomena in MySQL & PostgreSQL](https://dev.to/techschoolguru/understand-isolation-levels-read-phenomena-in-mysql-postgres-c2e)  
[MVCC in PostgreSQL — 1. Isolation : Postgres Professional](https://postgrespro.com/blog/pgsql/5967856)
[SQL Phenomena for Developers](https://dzone.com/articles/sql-phenomena-for-developers)  
[Transaction Isolation Levels With PostgreSQL as an example](https://mkdev.me/posts/transaction-isolation-levels-with-postgresql-as-an-example)