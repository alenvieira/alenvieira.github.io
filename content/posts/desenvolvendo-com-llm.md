---
title: "Desenvolvendo Software com assistência de LLM"
date: 2023-01-30T16:40:49-03:00
description: "Que tal desenvolvemos software com ajuda de uma IA?"
draft: false
---
>Existe apenas uma luz na ciência, e acendê-la em qualquer lugar é iluminar todos os lugares. Isaac Asimov

Conforme a reflexão do texto sobre Local-first, resolvi rumar no universo das LLMs rodando localmente para auxiliar no desenvolvimento de software. Mas, vamos por partes. Que diabos é uma LLM? Large language model ou LLM é um modelo de Inteligência Artificial(IA) na qual se tornou muito popular por ter bons resultados em diferentes áreas por meio de interações por chats. Entre as diversas áreas, ser uma assistente para os desenvolvedores de software terem uma melhor experiência no processo e focar mais na entrega do valor.

Há diversas ferramentas no mercado, algumas com planos gratuitos e outras mais robustas pagas, mas observei uma constante: a maioria precisa de interação com um modelo que está na nuvem. Prezando pela premissa local primeiro e que nem sempre podemos compartilhar códigos ou contextos do nosso problema sem quebrar alguma regra de privacidade, ou compliance, levando a considerar um estudo de algumas ferramentas que possam me ajudar a ter uma melhor produtividade quando estou desenvolvendo um software.

Estudando as soluções para isso, inferi que precisava, na verdade, precisaria de dois componentes:

- Um provedor local de modelos LLM: achei duas ferramentas legais, o [LM Studio](https://lmstudio.ai/) e o [Ollama](https://ollama.ai/) ambas ótimas. A verificar qual lhe agrada mais, ou a compatibilidade com seu sistema.
- Uma ferramenta se integrar como interface, o caso com a IDE(Integrated Development Environment): Para Vscode e ferramentas da Jetbrains existe o [Continue](https://continue.dev/), que interage além de com os provedores locais também os hospedados na nuvem.

Opa! Show! O que realmente posso fazer com esse conjunto?

- Autocompletar ou dar sugestões.
- Pedir para tentar corrigir uma parte específica explicando o problema.
- Tentar otimizar um determinado trecho do projeto.
- Pegar algum payload de uma requisição/resposta para gerar uma estrutura inicial.
- Gerar documentação ou explicação sobre algum trecho para fins de criação de uma especificação.

Imaginação é o limite! Pode-se automatizar uma prévia de code review pela IA mediante uma customização de engenharia de prompt como, por [exemplo](https://github.com/mattzcarey/code-review-gpt/blob/main/packages/code-review-gpt/src/review/prompt/prompts.ts), do projeto "[Code Review GPT](https://github.com/mattzcarey/code-review-gpt)". 

Pontos relevantes:

- Exige bastante da sua máquina. De preferência, use uma placa gráfica boa ou pelo menos 16 GB de RAM para rodar um CPU, entendendo que terá uma velocidade inferior.
- Apesar de alguns modelos serem open source. Nem todos podem ser utilizados para trabalhos de fins comerciais, isso depende da licença utilizada pelo modelo. [Aqui](https://blog.continue.dev/what-llm-to-use/) pode ter uma perspectiva dos modelos existentes. 
- Você terá que instalar, fazer configurações e customizações das ferramentas. Além de seleção de modelos e diversos testes para ver o que pode te atender.
- É preciso saber balancear o contexto fornecido para não ser insuficiente ou demais, saber fornecer instruções precisa para evitar ambiguidades.
- Lembre-se, é um assistente. A responsabilidade do código é sua, você sempre que avaliar as sugestões ou ter ponto crítico do que foi gerado.

É isso! Vamos testar um assistente com LLM?