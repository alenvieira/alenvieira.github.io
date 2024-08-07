---
title: "O tal do Isolamento no A.C.I.D."
date: 2024-07-11T09:38:53-03:00
description: "Vamos aprofundar um pouco sobre o isolamento?"
draft: false
---
>A solidão é estar sozinho; o isolamento é achar-se só no mundo quando se está só. Diane de Beausacq

Isolamento no A.C.I.D. é a garantia responsável por gerenciar a concorrência entre as transações e verificar como as transações simultâneas podem afetar umas às outras. Como tratei [no artigo passado](/posts/explicando-acid/).  

Vamos utilizar Python com algumas bibliotecas para trabalhar com PostgreSQL em um container, como abordado no artigo anterior. Realizaremos diversos testes para demonstrar diferentes cenários de isolamento. Os detalhes podem ser encontrados no [repositório](https://github.com/alenvieira/acidwithtests/).  

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

- **Read uncommitted**(Leitura não commitada): Nível menos restritivo que permite ler dados de outras transações que ainda não foram commitados ou confirmados, podendo levar a dirty read(leitura suja), non-repeatable read(leitura não repetível), lost update(atualizações perdida), read skew(leitura distorcida), write skew(escrita distorção) e phantom read(leitura fantasma).
- **Read committed**(Leitura commitada): Nível onde a transação pode ler apenas dados de transações que foram commitadas ou confirmadas, evitando dirty read, mas ainda permitindo non-repeatable read, lost update, read skew, write skew e phantom read.
- **Repeatable read**(Leitura repetida): Nível que garante que os dados lidos durante a transação não serão alterados até que a transação seja concluída, evitando leitura non-repeatable read, lost update, read skew e write skew, mas permitindo o fenômeno phantom read. 
- **Serializable**(Serializável): Nível de isolamento mais restritivo, que garante que as transações sejam completamente isoladas umas das outras, evitando qualquer interferência e fenômenos como dirty read, non-repeatable read, lost update, read skew, write skew e phantom read.

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

![image](/images/dirty_read.svg)

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

![image](/images/nonrepeatable_read.svg)
- Em T2, seleciona o funcionário;
- Em T1, atualiza o salário do funcionário;
- Por fim, seleciona o funcionário novamente.

Para exercitar a situação, fizemos dois testes. Um teste com o nível de isolamento **Read Committed** para ver o fenômeno e outro com o nível de isolamento **Repeatable Read** onde não ocorre.

#### Teste 01: Leitura não repetível com o nível de isolamento leitura commitada
```python
def test_norepeatable_read_with_read_committed_isolation_level(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Beth Lee", 3500))
            id_employee, = cur.fetchone()

    def transaction1():
        with psycopg.connect(self.connection_uri()) as conn:
            global norepeatable_read_first_salary, norepeatable_read_second_salary
            with conn.cursor() as cur:
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                norepeatable_read_first_salary, = cur.fetchone()
                time.sleep(2)
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                norepeatable_read_second_salary, = cur.fetchone()

    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                time.sleep(1)
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (4500, id_employee))                  
    
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

No teste 01, observamos o fenômeno da leitura não repetível. O valor do salário do funcionário na primeira consulta difere do valor na segunda consulta do mesmo funcionário, pois o salário foi modificado entre as duas consultas.

#### Teste 02: Evitando leitura não repetível com o nível de isolamento leitura repetível
```python
def test_without_norepeatable_read_with_repeatable_read_isolation_level(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Dep Tunner", 3000))
            id_employee, = cur.fetchone()

    def transaction1():
        with psycopg.connect(self.connection_uri()) as conn:
            global without_norepeatable_read_first_salary, without_norepeatable_read_second_salary
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                without_norepeatable_read_first_salary, = cur.fetchone()
                time.sleep(2)
                cur.execute("SELECT salary FROM employee WHERE id = %s", (id_employee,))
                without_norepeatable_read_second_salary, = cur.fetchone()
    
    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
                time.sleep(1)
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (4000, id_employee))
    
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

Já no teste 02, não ocorre o fenômeno porque o nível de isolamento **Repeatable Read** impede isso e contexto dessa transação as leituras são repetíveis. O banco de dados resolveu ignorar que o valor foi atualizado.

Perceba que isso não é considerado um erro. Dependendo do contexto, ter dados mais atualizados pode ser fundamental para a regra de negócio. Em outros casos, a consistência dos dados é mais importante. **É essencial identificar em quais situações esse fenômeno pode ser tolerado.**

### Lost Update

Lost Update, ou Atualização Perdida, é um fenômeno que ocorre quando duas transações tentam atualizar o mesmo registro e, devido à sobreescrita do dado por uma transação concomitante, uma delas acaba perdendo as informações. Esse fenômeno pode ocorrer no PostgreSQL quando o nível de isolamento está configurado como **Read Committed**, mas não ocorre no nível **Repeatable Read**. A imagem a seguir ilustra um exemplo desse fenômeno:

![image](/images/lost_update.svg)
- Em T1, seleciona o funcionário;
- Em T2, seleciona o mesmo funcionário e, em seguida, atualiza o seu salário;
- Por fim, em T1, atualiza o salário do funcionário, sobrescrevendo o dado.

Novamente, realizamos dois testes: um com o nível de isolamento **Read Committed** para observar o fenômeno e outro com o nível de isolamento **Repeatable Read**, onde ele não ocorre.

#### Teste 01: Atualização perdida com o nível de isolamento leitura commitada
```python
def test_lost_update_with_read_committed_isolation_level(self):
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

No teste 01, observamos o fenômeno da atualização perdida. A primeira transação não refletiu as mudanças no salário feitas pela segunda transação, resultando na sobreescrita das informações da segunda transação.

#### Teste 02: Evitando atualização perdida com o nível de isolamento leitura não repetível
```python
def test_without_lost_update_with_repeatable_read_isolation_level(self):
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

No cenário do teste 02, o banco de dados com o nível de isolamento **Repeatable Read** identifica o problema e retorna um erro de serialização, impedindo que o fenômeno de atualização perdida ocorra.

Para evitar comprometer a integridade dos dados com a atualização perdida, é preferível obter um erro ou identificar a situação e tentar novamente a transação com dados mais consistentes. Existem outras maneiras de tratar esse fenômeno, que serão discutidas em um próximo post.

### Read Skew

Read Skew ou Leitura Distorcida é um fenômeno onde as transações executam consultas sobre dados que podem sofrer alterações e retornam dados inconsistentes. Ele pode ocorrer no PostgreSQL quando está no nível de isolamento **Read Committed**, mas não ocorre no **Repeatable Read**. 

![image](/images/read_skew.svg)
- T1 deseja selecionar os funcionários 1 e 2.
- Durante a seleção dos dois funcionários, T2 atualiza os salários deles.
- Como resultado, T1 obtém uma versão inconsistente dos dados: a informação do funcionário 1 desatualizada e a do funcionário 2 atualizada.

Para simular o fenômeno, realizamos dois testes: um com o nível de isolamento Read Committed, onde ele ocorre, e outro com Repeatable Read, onde ele não ocorre.

#### Teste 01: Leitura distorcida com o nível de isolamento leitura commitada 
```python
def test_read_skew_with_read_committed_isolation_level(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Amanda Lang", 4000))
            id_employee1, = cur.fetchone()
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("August Morse", 4000))
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
    self.assertEqual(read_skew_salary_employee2, 5000)
    
    self.assertEqual(new_salary_employee1, 5000)
    self.assertEqual(new_salary_employee2, 5000)
```

No teste acima, no primeiro conjunto de validações, observamos que o salário do primeiro funcionário ainda é o valor inicial, enquanto o salário do segundo funcionário já está atualizado. Isso demonstra o fenômeno da leitura distorcida. No entanto, ao verificarmos os valores dos salários após a conclusão das transações, confirmamos que ambos foram atualizados corretamente.

#### Teste 02: Evitando leitura distorcida com o nível de isolamento leitura repetível
```python
def test_without_read_skew_with_repeatable_read_isolation_level(self):
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
No teste com o nível de isolamento Repeatable Read, o banco de dados intervém e mantém ambos os valores antigos, sem atualizações durante a transação de consulta dos funcionários no banco de dados. Isso pode ser necessário quando a funcionalidade exige dados consistentes no instante da consulta.

Esse fenômeno também ocorre quando há consultas em mais de uma tabela. Ao consultar outras tabelas, pode ocorrer uma alteração nos dados da tabela onde ocorreu a primeira consulta, resultando em inconsistências. Por simplicidade, mostramos o exemplo usando apenas uma tabela.

É importante lembrar que manter a sincronia dos dados de forma consistente é um desafio, especialmente quando há vários relacionamentos entre tabelas. Avalie cuidadosamente as possibilidades de vir a ocorrer o fenômeno da leitura distorcida ao consultar dados vitais para o seu negócio.

### Write Skew
Write Skew, ou Escrita Distorcida, é um fenômeno que ocorre quando transações modificam um dado baseado em uma leitura de dados que já não é mais a mesma. De acordo com o padrão SQL, isso não ocorre no nível de isolamento **Repeatable Read**, mas ocorre no PostgreSQL. Esse fenômeno é evitado no nível **Serializable**.

![image](/images/write_skew.svg)
- Em T1, ocorre a seleção do somatório dos salários dos funcionários.
- Em T2, também é selecionado o somatório dos salários dos funcionários e, em seguida, o salário do funcionário 2 é atualizado com base nessa informação.
- Por fim, em T1, o salário do funcionário 1 é atualizado com base no valor do primeiro somatório obtido, gerando o fenômeno de escrita distorcida.

Ambas as transações consultam o mesmo dado (somatório de salários) e atualizam os salários de funcionários distintos. O fenômeno acontece porque, durante a atualização do salário do funcionário 1 em T1, o somatório dos salários foi alterado por T2.

No teste 01, replicamos o fenômeno de escrita distorcida, enquanto no teste 02 impedimos sua ocorrência ao aplicarmos o nível de isolamento **Serializable**. 

#### Teste 01: Escrita distorcida com o nível de isolamento leitura repetível
```python
def test_write_skew_with_repeatable_read_isolation_level(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Selma Bates", 5000))
            id_employee1, = cur.fetchone()
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Samuel Bowen", 9000))
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
    
    self.assertEqual(new_salary_employee1, 6400)
    self.assertEqual(new_salary_employee2, 10400)
```

No teste 01, os salários dos funcionários são incrementados em 10% com base no somatório de salários. Se as transações fossem executadas sequencialmente, o segundo funcionário se beneficiaria do aumento de salário do primeiro, recebendo um aumento maior. No entanto, o fenômeno de escrita distorcida ocorre, resultando em ambos os funcionários recebendo o mesmo incremento. 
 
#### Teste 02: Evitando escrita distorcida com o nível de isolamento serializável
```python
def test_without_write_skew_with_serializable_isolation_level(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Dean Knox", 5000))
            id_employee1, = cur.fetchone()
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s) RETURNING id", ("Madison Frey", 9000))
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
    self.assertEqual(new_salary_employee2, 10400)
```


No teste 2, o banco de dados com o nível de isolamento mais agressivo detecta o caso de escrita distorcida e impede que um dos funcionários receba o aumento. Em vez de incrementar o salário, é lançado um erro de serialização. Isso faz sentido, já que a regra do incremento é aplicar 10% sobre o somatório dos salários atualizados. Se a transação que gerou o erro fosse refeita, o funcionário 1 receberia o aumento corretamente.

Esse fenômeno, assim como a leitura distorcida, também ocorre quando há consultas em mais de uma tabela. É necessário estar atento às possíveis ocorrências e verificar se são toleráveis dentro das regras de negócio.

### Phanton Read

Phantom Read, ou Leitura Fantasma, ocorre quando uma transação depende de dados que podem ser modificados por outras transações. De acordo com o padrão SQL, esse fenômeno pode ocorrer no nível **Repeatable Read**, mas no PostgreSQL isso não acontece. No entanto, ele pode ser observado no nível **Read Committed**. A figura abaixo demonstra um exemplo desse fenômeno:

![image](/images/phantom_read.svg)
- Em T1, seleciona o somatório dos salários dos funcionários;
- Em T2, insere um funcionário;
- Por fim, em T1, seleciona novamente o somatório dos salários dos funcionários, que é diferente do valor selecionado anteriormente.

Dois testes foram criados para verificar esse fenômeno em diferentes nível de isolamento: um teste com o nível de isolamento com **Read Committed** e outro com **Repeatable Read**.

#### Teste 01: Leitura Fantasma com o nível de isolamento leitura commitada
```python
def test_phantom_read_with_read_committed_isolation_level(self):
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

No teste 01 acontece o fenômeno da leitura fantasma, pois entre a primeira e a segunda consulta do somatório dos salários ocorre a inserção de um novo funcionário.

#### Teste 02: Evitando leitura fantasma com nível de isolamento leitura repetível 
```python
def test_without_phanton_read_with_repeatable_read_isolation_level(self):
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

No PostgreSQL, o nível de isolamento **Repeatable Read** impede o fenômeno da leitura fantasma, como podemos verificar no teste 02. É importante destacar que isso ocorre quando estamos realizando uma consulta de leitura numa base de dados enquanto outra transação está realizando inserções ou deleções de registros.

Esse caso é semelhante ao da leitura não repetível, com a diferença de que a leitura fantasma trabalha com um **conjunto de dados**, enquanto a leitura não repetível trabalha com o **mesmo registro**. Devemos avaliar se é aceitável, em termos de regras de negócio, não possuir a informação mais atualizada e como isso impacta as funcionalidades da sua aplicação.

## Algumas observações

Dentre os níveis de isolamento analisados, fica claro que o nível **Serializable** é o mais restritivo e impede a ocorrência de fenômenos que podem gerar inconsistências nos dados. No entanto, adotar o nível Serializable como padrão pode ser custoso e afetar o desempenho do banco de dados e do sistema. O isolamento é uma escolha entre **maior consistência** e **maior concorrência**. 

Existem algumas abordagens para evitar ou detectar diversos desses fenômenos. Os métodos mais comuns são:

- [Lock-based concurrency](https://en.wikipedia.org/wiki/Record_locking) ou Controle de Concorrência com Bloqueio, utiliza travas ou bloqueios de recursos para impedir que transações acessem ou modifiquem dados simultaneamente, principalmente através de bloqueio pessimista. Isso significa que enquanto um recurso está bloqueado, outras transações precisam esperar até que o bloqueio seja liberado.

- [Non-lock concurrency control](https://en.wikipedia.org/wiki/Non-lock_concurrency_control) ou Controle de Concorrência sem Bloqueio, utiliza métodos que evitam o uso de bloqueios. Geralmente, emprega controle por timestamp ou bloqueios otimistas para manter a consistência do banco de dados. Isso permite que várias transações possam acessar e modificar dados simultaneamente, desde que não ocorram conflitos.

A escolha entre essas abordagens depende de vários fatores que devem ser considerados:

- Tratamento e frequência de erros ou conflitos na base de dados, e a estratégia para lidar com essas situações, como a retentativa de operações.
- Restrições de tempo da aplicação ou do negócio, e requisitos de desempenho que podem influenciar na escolha do método mais adequado.
- Nível de consistência dos dados necessário para cada funcionalidade do sistema.
- Complexidade de implementação e manutenção de cada mecanismo na aplicação.
- Consulta detalhada da documentação sobre o isolamento do banco de dados para garantir que os requisitos sejam atendidos.

No próximo post, vamos explorar esses métodos com mais detalhes.

## E por hoje é só, pessoal!

Hoje aprofundamos o conceito de isolamento no A.C.I.D., seus níveis e os fenômenos que podem ocorrer, focando nossa análise em como garantir que nossa aplicação não tenha dados inconsistentes. Assim como no artigo anterior, abaixo estão alguns links interessantes que complementam esta leitura.

Agradecimento especial ao meu amigo [Marcelo Gomes](https://gomesmr.substack.com/) pela revisão e melhorias deste artigo. Até o próximo artigo!

[Deeply understand Isolation levels and Read phenomena in MySQL & PostgreSQL](https://dev.to/techschoolguru/understand-isolation-levels-read-phenomena-in-mysql-postgres-c2e)  
[MVCC in PostgreSQL — 1. Isolation : Postgres Professional](https://postgrespro.com/blog/pgsql/5967856)  
[SQL Phenomena for Developers](https://dzone.com/articles/sql-phenomena-for-developers)  
[Transaction Isolation Levels With PostgreSQL as an example](https://mkdev.me/posts/transaction-isolation-levels-with-postgresql-as-an-example)