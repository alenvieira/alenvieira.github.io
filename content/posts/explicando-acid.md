---
title: "Explicando o A.C.I.D. com testes"
date: 2024-04-15T08:13:29-03:00
description: "Quais são as garantias do A.C.I.D.?"
draft: true
---
>Os dados são preciosos e duram mais que os próprios sistemas. Tim Berners-Lee

O A.C.I.D. é uma sopa de letrinhas que oferece diversas garantias para transações em diferentes bancos de dados. Certo, mas que história é essa de explicar isso com testes? Essas propriedades são difundidas na forma teórica, porém nem sempre vemos na prática por meio de código. Gosto de ver como um conceito funciona por meio do código mais especificamente com testes unitários e assim consigo assimilar melhor os conceitos.

Os testes usaram a linguagem de programação [Python](https://www.python.org/), [PostgreSQL](https://www.postgresql.org/) como banco de dados relacional, usamos  a biblioteca Python [testcontainers-python](https://testcontainers-python.readthedocs.io/en/latest/) para rodar o PostgreSQL em um container e a biblioteca Python [Psycopg 3](https://www.psycopg.org/psycopg3/) para conectar-se ao PostgreSQL e executar comandos. Mais informações podem ser encontradas neste [repositório](https://github.com/alenvieira/acidwithtests/) no qual até [rodei os testes](https://github.com/alenvieira/acidwithtests/actions/runs/8672804316/job/23783502175) através do Github Actions e provar que não rodam somente na minha máquina :D.

Um detalhe muito importante ao qual prestar atenção é o uso de contexto de conexão. As conexões criadas usando with no Psycopg 3 apresentam o seguinte comportamento:
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
Ou seja, se tudo for concluído com sucesso durante a conexão, todas as operações serão confirmadas(commit), caso contrário tudo será revertido(rollback) e em ambos os casos a conexão será encerrada.

Para preparar o ambiente dos testes unitários, precisei criar o seguinte:

- o método setUpClass que inicia o PostgreSQL e cria uma tabela nele na criação da classe, ou seja, antes de executar os testes;
- o método tearDownClass para parar o PostgreSQL após todos os testes;
- o método connection_uri permite obter facilmente um URI do Postgresql e usá-lo para criar uma conexão;
- o método tearDown para fazer a limpeza na tabela após cada teste.
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
Todos os conceitos apresentados são sobre transações. Mas o que é uma transação? Transação é um processo que envolve uma ou mais operações de leitura e gravação e está normalmente associada a uma regra de negócio. Ok! Vamos aos testes de cada letrinha do acrônimo:

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
No teste de atomicidade acima, a primeira parte mostra uma transação adicionando o funcionário John Smith e a segunda parte mostra uma transação adicionando a funcionária Beth Lee, mas imediatamente após ocorre uma exceção. Que poderia ser inúmeras situações como, por exemplo, o tipo do dado do salário errado, ou uma regra de negócio incorreta. Neste exemplo, simplifiquei retornando uma exceção RuntimeError. A terceira parte verifica quais funcionários estão salvos.

Na parte final, fazemos uma primeira asserção que realmente foi salvo apenas um registro no banco, Pode ser confuso. Ué porque só um? Então, vamos por partes, a primeira parte foi salvo o John com sucesso totalizando um registro. Na segunda parte, apesar da operação de salvar a Beth estava certinha, logo depois ocorre uma exceção que faz todas as operações da transação serem revertida. E assim, finalizamos os testes checando se foi o John que realmente foi salvo, concluindo que as transações são realmente indivisíveis e evitando transações que possam introduzir dados inconsistentes. 

Porém, entretanto, toda via, temos que tomar cuidado com o que realmente as operações que realmente necessitam estarem juntas numa transação, por exemplo, uma transferência de valores, uma transação de X para Y, vai alterar valores de X e Y então é um caso bem interessante de estarem juntas. Já o caso do inserir funcionários no sistema podem não serem o melhor exemplo, visto que cada cadastro de funcionário pode ser independente. Faça a análise de seu cenário!

O C de Consistency(Consistência) é uma propriedade associada as regras de integridade definidas. Cada transação sempre deixa o estado consistente. Isso significa que todas as regras de integridade são aplicadas como validações de existência da coluna, de tipo de dado da coluna, de chave primária, de chave estrangeira, de unicidade da coluna e outras restrições que o banco de dados suporte.
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
- Na segunda tem uma operação de inserir a funcionária Mary Castle, porém no campo do salário dessa funcionária passa se uma data invés de um número;
- Por fim, a terceira operação tem uma transação de inserir o funcionário Bob Fox e o funcionário Alan Rock que tenta inserir em um campo chamado date uma data.

Em resumo, só o funcionário Dep Tunner foi inserido porque a segunda foi revertida porque o tipo de dado do campo salário foi passado incorretamente revertendo a transação, assim como na terceira transação que tenta inserir em um campo que não existe na tabela. Isso garante a consistência dos dados ao não permitir tipo de dados diferente ao mapeado ou campo não existente na base de dados. 

Temos que nos atentar essa garantia torna nossa tabela menos flexível e evitar restrições demais visto que elas podem afetar nossa performance. Lembre-se todas as regras de integridade são aplicadas a cada operação. Escolha bem suas regras e faça uma medição do impacto dela se necessário.

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

Porém, quando fazemos a verificação, a funcionária tem apenas o aumento de 10% em seu salário. O que ocorreu? E o aumento de 20% da segunda transação? Ocorreu que a primeira transação não obteve as alterações da segunda transação e sobrescreveu, causando uma inconsistência no banco de dados. As transações ocorreram de forma isolada e não houve um devido tratamento para a situação.

Há diversas maneiras de tratar a situação e diferentes configurações para um melhor controle do isolamento, na qual é melhor tratar em um próximo post sobre isso. Em todo caso, nos casos de acesso e escrita de mesma informações temos que nos atentar ao isolamento e assim obter a melhor situação entre obter ao melhor tratamento com a consistência ou com a concorrência.

O D de Durability(Durabilidade) é a propriedade que gerencia a recuperação do banco de dados em caso de falhas. Em caso de sucesso de uma transação manter o estado de forma permanente.
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
Podemos ver que na primeira conexão aberta logo a pós o comando de inserir um novo funcionário já forcei um commit e então derrubei o container. Para provar que o banco de dados está inacessível, tentamos acessar e não conseguimos. Então já começamos os procedimentos para inicializar o banco de dados novamente para consultar se os dados realmente foram salvos após desligamos forçadamente o container. Assim verificamos esta propriedade.

Então, após a transação for confirmada que foi está armazenado precisa ser alterado. A durabilidade entra em jogo garantindo que tudo seja armazenado em uma unidade de persistência não volátil e mesmo se o banco de dados falhar nada será perdido. 

Abordamos todos os conceitos do ACID e deixamos um gancho para discutir o isolamento com mais detalhes em outro post. O código foi criado para fins didáticos. Em circunstâncias normais, você tem um conjunto de conexões com o banco de dados abertas e vai solicitando conexões para as transações, diferentemente dos testes em que abrirmos e fecharmos diversas conexões. Espero que tenha aproveitado a jornada e se quiser novas abordagens explicando o ACID logo mais tem três links. Até!

Referências:

[ACID Databases – Atomicity, Consistency, Isolation & Durability Explained](https://www.freecodecamp.org/news/acid-databases-explained/)

[ACID Properties](https://medium.com/@mesfandiari77/acid-properties-dc233074050c)

[Building Robust Systems with ACID and Constraints](https://brandur.org/acid)