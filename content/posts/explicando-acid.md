---
title: "Explicando o A.C.I.D. com testes"
date: 2024-04-15T12:18:29-03:00
description: "Quais são as garantias do A.C.I.D.?"
draft: false
---
>Os dados são preciosos e duram mais que os próprios sistemas. Tim Berners-Lee

O A.C.I.D. é uma sopa de letrinhas onde cada letra corresponde uma garantia para transações em diferentes bancos de dados. No entanto, essas propriedades são geralmente explicadas de forma teórica e nem sempre são demonstradas na prática com código. Eu prefiro ver como os conceitos funcionam através de testes, pois isso me ajuda a entendê-los melhor.

Os testes foram realizados com a linguagem de programação [Python](https://www.python.org/) e o banco de dados relacional [PostgreSQL](https://www.postgresql.org/). Utilizamos a biblioteca Python [testcontainers-python](https://testcontainers-python.readthedocs.io/en/latest/) para rodar o PostgreSQL em um container e a biblioteca Python [Psycopg 3](https://www.psycopg.org/psycopg3/) para conectar-se ao PostgreSQL e executar comandos. Mais detalhes estão disponíveis neste [repositório](https://github.com/alenvieira/acidwithtests/), onde você pode ver que os [testes foram executados](https://github.com/alenvieira/acidwithtests/actions/runs/8672804316/job/23783502175) com sucesso no GitHub Actions, comprovando que funcionam fora da minha máquina. :D

Um detalhe muito importante a observar é o uso do contexto de conexão no Psycopg 3. Quando você cria uma conexão usando o **with**, o comportamento é o seguinte:
```python
# Fonte: https://www.psycopg.org/psycopg3/docs/basic/usage.html#connection-context
conn = psycopg.connect()
try:
    ... # use the connection
except BaseException:
    conn.rollback()
else:
    conn.commit()
finally:
    conn.close()
```
Ou seja, se todas as operações na conexão forem bem-sucedidas, elas serão confirmadas (commit). Caso contrário, todas serão revertidas (rollback). Em ambos os casos, a conexão será encerrada automaticamente.

Para preparar o ambiente dos testes unitários, implementei o seguinte:

- o método setUpClass que inicia o PostgreSQL e cria uma tabela antes da execução dos testes;
- o método tearDownClass para parar o PostgreSQL após todos os testes;
- o método connection_uri permite obter facilmente um URI do Postgresql e usá-lo para criar uma conexão;
- o método tearDown que realiza a limpeza da tabela após cada teste.
```python
class AcidWithPostgresTestCase(unittest.TestCase):
  
    @classmethod
    def setUpClass(cls):
        cls.postgres = PostgresContainer("postgres:16.2-alpine")
        cls.postgres.start()
      
        with psycopg.connect(cls.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("CREATE TABLE employee (id serial PRIMARY KEY, name text, salary double precision)")

    @classmethod
    def tearDownClass(cls):
        cls.postgres.stop()
  
    @classmethod
    def connection_uri(cls):
        return cls.postgres.get_connection_url().replace("+psycopg2", "")
  
    def tearDown(self):
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("DELETE FROM employee")
```
Todos os conceitos apresentados referem-se a transações. Mas o que é uma transação? Uma transação é um processo que envolve uma ou mais operações de leitura e gravação, geralmente associadas a uma regra de negócio. Agora, vamos testar cada uma das propriedades do acrônimo:

O A de Atomicity(Atomicidade) é a propriedade responsável por gerenciar as transações. As transações são indivisíveis/atômicas, devem confirmar tudo com um commit ou desfazer tudo com um rollback.
```python
def test_atomicity(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s)", ("John Smith", 2500))
  
    with self.assertRaises(Exception):
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s)", ("Beth Lee", 3500))
                raise RuntimeError
  
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT name, salary FROM employee")
            db_data = cur.fetchall()
  
    self.assertEqual(len(db_data), 1)
    self.assertEqual(db_data[0], ("John Smith", 2500))
```
No teste de atomicidade acima, a primeira parte realiza uma transação para adicionar o funcionário John Smith. A segunda parte tenta adicionar a funcionária Beth Lee, mas uma exceção ocorre imediatamente após, que pode ser causada por diversos problemas, como um tipo de dado incorreto para o salário ou uma regra de negócio inválida. Para simplificar, neste exemplo, a exceção é uma RuntimeError. A terceira parte do teste verifica quais funcionários foram realmente salvos na base de dados.

Na parte final, a primeira afirmação pode causar confusão, pois só um registro foi salvo no banco. Vamos esclarecer: na primeira etapa, o registro de John foi salvo com sucesso, totalizando um registro. Na segunda etapa, apesar de Beth ter sido salva corretamente, uma exceção fez com que toda a transação fosse revertida. Portanto, apenas o registro de John permaneceu. Os testes confirmaram que as transações são indivisíveis, prevenindo a introdução de dados inconsistentes.

Porém, entretanto, todavia, é crucial avaliar quais operações realmente precisam estar agrupadas em uma transação. Por exemplo, uma transferência de valores entre duas contas — de X para Y — deve ser tratada como uma única transação, pois envolve a alteração dos valores de ambas as contas. Neste caso, garantir que ambas as operações (débito e crédito) ocorram juntas é essencial para manter a consistência.

Por outro lado, a inserção de funcionários no sistema pode não ser o melhor exemplo de uma transação indivisível, visto que cada cadastro de funcionário pode ser independente. Portanto, ao projetar suas transações, é importante analisar cuidadosamente seu cenário e determinar quais operações podem ser agrupadas.

O C de Consistency(Consistência) é uma propriedade associada às regras de integridade definidas. Cada transação sempre deixa o estado consistente. Isso significa que todas as regras de integridade são aplicadas como validações de existência da coluna, de tipo de dado da coluna, de chave primária, de chave estrangeira, de unicidade da coluna e outras restrições que o banco de dados suporte.
```python
def test_consistency(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s)", ("Dep Tunner", 3000))
  
    with self.assertRaises(Exception):
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s)", ("Mary Castle", datetime.date(2024, 12, 20)))

    with self.assertRaises(Exception):
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s)", ("Bob Fox", 2750))
                cur.execute("INSERT INTO employee (name, date) VALUES (%s, %s)", ("Alan Rock", datetime.date(2024, 12, 20)))
  
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT name, salary FROM employee")
            db_data = cur.fetchall()
  
    self.assertEqual(len(db_data), 1)
    self.assertEqual(db_data[0], ("Dep Tunner", 3000))
```
No teste de consistência acima, vemos 3 transações para inserir funcionários:
- A primeira tem uma operação para inserir o funcionário Dep Tunner;
- Na segunda tem uma operação para inserir a funcionária Mary Castle, porém no campo do salário dessa funcionária passa se uma data invés de um número;
- Por fim, a terceira operação tem uma transação para inserir o funcionário Bob Fox e o funcionário Alan Rock que tenta inserir em um campo chamado date uma data.

Em resumo, apenas o funcionário Dep Tunner foi inserido porque a segunda transação foi revertida devido a um tipo de dado incorreto no campo salário, o que causou o rollback da transação. Da mesma forma, a terceira transação falhou ao tentar inserir dados em um campo que não existe na tabela. Esses mecanismos garantem a consistência dos dados ao impedir a inserção de tipos de dados incompatíveis ou a inserção em campos inexistentes na base de dados.

Devido à garantia de integridade, nossa tabela pode se tornar menos flexível, e restrições excessivas podem afetar a performance. Lembre-se de que todas as regras de integridade são aplicadas a cada operação. Portanto, é crucial escolher as regras com cuidado e, se necessário, medir o impacto delas no desempenho do sistema.

O I de Isolation(Isolamento) é a propriedade que gerencia a concorrência entre as transações. Cada transação opera independente das outras que estão rodando em paralelo, como se executasse um de cada vez. Há níveis de isolamento entre as transações que podem ser ativadas.
```python
def test_isolation(self):
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s)", ("Jess Tex", 4000))

    def transaction1():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT id, salary FROM employee WHERE name = %s", ("Jess Tex",))
                db_data = cur.fetchone()
                time.sleep(4)
                new_salary = db_data[1] * 1.1
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (new_salary, db_data[0]))
  
    def transaction2():
        with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                time.sleep(2)
                cur.execute("SELECT id, salary FROM employee WHERE name = %s", ("Jess Tex",))
                db_data = cur.fetchone()
                new_salary = db_data[1] * 1.2
                cur.execute("UPDATE employee SET salary=%s WHERE id=%s", (new_salary, db_data[0]))
  
    threads = [Thread(target=transaction1), Thread(target=transaction2)]
    for thread in threads:
        thread.start()
    for thread in threads:
        thread.join()
  
    with psycopg.connect(self.connection_uri()) as conn:
            with conn.cursor() as cur:
                cur.execute("SELECT salary FROM employee WHERE name = %s", ("Jess Tex",))
                db_data = cur.fetchone()
  
    self.assertEqual(db_data[0], 4400)
```
O teste começa com a inserção da funcionária Jess Tex, e possui dois métodos que rodam em threads diferentes para uma concorrência de transações:
- O primeiro método seleciona a funcionária Jess, "pensa" por um tempo por 4 segundos e resolve realizar um aumento de 10% em seu salário. 
- No segundo método começa "pensando" por 2 segundos o que irá fazer, já seleciona a funcionária Jess e aplica um aumento de 20% em seu salário. 

No entanto, ao verificar, notamos que o salário da funcionária foi aumentado em apenas 10%, e não em 20% como esperado. O que aconteceu? A primeira transação não incorporou as alterações da segunda transação e acabou sobrescrevendo-as, resultando em uma inconsistência no banco de dados. Isso ocorreu porque as transações foram executadas de forma isolada, sem um tratamento adequado para garantir que as alterações de uma não interferissem na outra.

Existem diversas abordagens e configurações para controlar o isolamento de transações, e é melhor explorar esses detalhes em um [próximo post](/posts/o-tal-do-isolamento/). No entanto, é importante notar que, ao acessar e escrever as mesmas informações, devemos prestar atenção ao nível de isolamento para equilibrar a consistência com a concorrência. Isso garantirá que as transações sejam tratadas da melhor maneira possível, considerando tanto a integridade dos dados quanto a eficiência do sistema.

O D de Durability(Durabilidade) é a propriedade que gerencia a recuperação do banco de dados em caso de falhas. Em caso de sucesso de uma transação, manter o estado de forma permanente.
```python
def test_durability(self):
    container = self.postgres.get_wrapped_container()
    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("INSERT INTO employee (name, salary) VALUES (%s, %s)", ("Paul Port", 3000))
            conn.commit()
            container.stop()
            container.wait()

    with self.assertRaises(Exception) as cm:
        psycopg.connect(self.connection_uri())
  
    container.start()
    self.postgres._connect()

    with psycopg.connect(self.connection_uri()) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT name, salary FROM employee")
            db_data = cur.fetchall()
  
    self.assertEqual(len(db_data), 1)
    self.assertEqual(db_data[0], ("Paul Port", 3000))
```
Podemos observar que, na primeira conexão aberta, após o comando para inserir um novo funcionário, forcei um commit e, em seguida, derrubei o container. Para demonstrar que o banco de dados estava inacessível, tentei acessá-lo e não obtive sucesso. Em seguida, iniciei novamente o banco de dados para verificar se os dados realmente foram salvos após o desligamento forçado do container. Dessa forma, conseguimos validar a propriedade de durabilidade.

Após uma transação ser confirmada por meio de um commit, todas as operações devem ser refletidas no banco de dados. A durabilidade garante que essas alterações sejam armazenadas em uma unidade de persistência não volátil. Assim, mesmo que o banco de dados sofra uma falha, as alterações realizadas durante a transação não serão perdidas.

Abordei todos os conceitos do ACID e deixei um gancho para discutir o isolamento com mais detalhes em outro post. O código apresentado foi criado para fins didáticos; em um ambiente real, você geralmente possui um conjunto de conexões abertas com o banco de dados e solicita conexões conforme necessário para as transações, ao contrário dos testes onde abri e fechei diversas conexões.

Espero que tenha aproveitado a jornada! Se quiser explorar novas abordagens para entender o ACID, confira os três links que seguem. Até mais!

Referências:

[ACID Databases – Atomicity, Consistency, Isolation & Durability Explained](https://www.freecodecamp.org/news/acid-databases-explained/)

[ACID Properties](https://medium.com/@mesfandiari77/acid-properties-dc233074050c)

[Building Robust Systems with ACID and Constraints](https://brandur.org/acid)