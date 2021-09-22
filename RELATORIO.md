# RELATÓRIO DO DESENVOLVIMENTO
***
## dificuldades encontradas
Encontrei dificuldade em realizar os testes automatizados e na criação da documentação formal do projeto, pois não tenho muita experiência com essas tarefas.
Achei que são tarefas interessantes e tenho todo interesse de continuar estudando.

***

## justificativas dos métodos utilizados
Como no site oficial da [Câmara dos Deputados](https://www.camara.leg.br/busca-portal/proposicoes/pesquisa-simplificada) uma __API__ é consumida para retornar 
um *dicionário* com a resposta referente aos parâmetros passados, achei um caminho prático para realizar a busca requisitada, pois consegui na mesma *response* raspar: *título da proposição, id da proposição e data de apresentação da proposição*.
<br></br>
Para fazer o hash md5 dos __PDFs de Inteiro Teor__ achei como método mais seguro, salvá-los temporariamente, fazer o __hash md5__ e deletar após exportar o *DataFrame*
***

## fontes de pesquisa acessadas
Cursos da __Udemy__ e documentação oficial das *libs* usadas.
