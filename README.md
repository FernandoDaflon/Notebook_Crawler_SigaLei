# Notebook_Crawler_SigaLei
### Notebook do Jupyter com crawler que retorna csv ou json com md5 dos PDFs de Inteiro Teor dos ultimos 3 dias do site da câmara
***
## Explicação do Projeto

* __PROBLEMA__ :arrow_right: Conferir se os documentos de *Inteiro Teor* das proposições (*PEC, PLP ou PL*) mais recentes foram baixados corretamente nos últimos três dias.
* __SOLUÇÃO__ :arrow_right: Sistema que receba o tipo de proposição desejada (*PEC, PLP ou PL*) e retorne uma lista de hash MD5 gerada a partir dos documentos de Inteiro Teor das proposições apresentadas nos últimos três dias na Câmara dos Deputados. Essa hash é utilizada para verificar se aquele documento já foi baixado ou não.

***

## Como executar o projeto:

Rodando o notebook no [jupyter](https://jupyter.org/) ou no [google colab](https://colab.research.google.com) e passando *PEC, PLP ou PL* no campo __*input*__ e apertar a tecla __ENTER__

***
## Explicando o Crawler:

1) No site oficial da [Câmara dos Deputados](https://www.camara.leg.br/busca-portal/proposicoes/pesquisa-simplificada) uma __API__ é consumida para retornar um *dicionário* com a resposta referente aos parâmetros passados.

## * acessa_api_camara()

Faz um __POST__ com os parametros da API (__*order, ano, pagina, tiposDeProposicao*__) no site da Camara e recebe um __JSON de resposta__.

Passa como parametros do payload:

* o ano_atual → como há recesso parlamentar no final do ano, nao ha riscos de perder propostas nos 3 primeiros dias do ano.
* o tipo de proposicao → *PL, PLP ou PEC*.
* o numero da pagina → a api exporta cum conjunto de dados limitados por pagina, com isso, passamos o número de cada página.
* a string "data" → para ordernar a resposta pela *data* mais recente.

Passa como parametros do headers:

* application/json → informando o Content-Type
* um user agent → da lib fake-useragent. O método *random* passa user-agents de forma aleatéria (google, firefox, IE, Safari ...)

```
def acessa_api_camara(ano_atual, tipo_proposicao, numero_da_pagina):
    ua = UserAgent()
    url = "https://www.camara.leg.br/api/v1/busca/proposicoes/_search"
    payload = {
        "order":"data",
        "ano":ano_atual,
        "pagina":numero_da_pagina,
        "tiposDeProposicao": tipo_proposicao
    }
    headers = {
        'Content-Type': "application/json",
        'User-Agent': ua.random    
        }
    response = requests.post(url, data=json.dumps(payload), headers=headers)
    return response.json()
```
***
2) Com a resposta da API do site oficial da Câmara dos Deputados é feito o *scrape* da __*data da apresentação da proposição*__, do __*título*__(ou nome)__*da proposição*__ e __*id da proposição*__. 
   <br></br>
   O id da proporsição é passado no link de cada proposição `  f'https://www.camara.leg.br/proposicoesWeb/fichadetramitacao?idProposicao={id_preposicao}'  `
   <br></br>
   É feito um request nesse link e no __html__ de resposta encontramos o link do nosso documento alvo, o __PDF de Inteiro Teor__. 
   <br></br>
   O download do __PDF de Inteiro Teor__ é feito em um arquivo temporário e é extraído o hash MD5 desse arquivo. 
   <br></br>
   Uma lista é retornada. 

## * retorna_props()

Recebe a resposta em JSON da API da Camara e extrai dados **data de apresentação, titulo da proposição e id da proposição**.o

Ex.:

data de apresentação | titulo da proposição | id da proposição
-- | -- | --
2021-09-17T15:25:00 | PL 3211/2021 | 2299232

A *Id da proposição* é usada para completar o link onde está localizado o pdf de __*Inteiro Teor*__ e fazer o download.
Cada pd é salvo temporariamente e desse arquivo temporario é extraído o hash md5.

Com isso, o md5 de cada proposicao é formando listas com **data de apresentação, titulo da proposição, id da proposição e md5**.

Ex.:

data de apresentação | titulo da proposição | id da proposição | md5
-- | -- | -- | --
2021-09-17 | PL 3211-2021 | 2299232 | 4d9eef77f177f4e9b4113f9163b575e5

A função retorna uma listas com essas listas.

```
def retorna_props(qtd, resp_json, dias_3):
    lista_preps = []
    i = 0
    while i < qtd:
        data = resp_json['hits']['hits'][i]['_source']['dataApresentacao'][:10]
        data = datetime.strptime(data, '%Y-%m-%d').date()
        titulo = resp_json['hits']['hits'][i]['_source']['titulo'].replace('/','-')
        id_preposicao = resp_json['hits']['hits'][i]['_id']
        i += 1
        if data >= dias_3:
            url_prep = f'https://www.camara.leg.br/proposicoesWeb/fichadetramitacao?idProposicao={id_preposicao}'
            resp = requests.get(url=url_prep)
            tree = html.fromstring(html=resp.text)
            link_pdf = tree.xpath('//*[@id="content"]/h3[1]/span[2]/a/@href')[0]
            elo = link_pdf[link_pdf.find('codteor'):]
            url_pdf = f'https://www.camara.leg.br/proposicoesWeb/prop_mostrarintegra?{elo}.pdf'
            chunk_size = 2000
            r = requests.get(url_pdf, stream=True)
            salva_pdf = 'salva_pdf'
            with open(f'{salva_pdf}.pdf', 'wb') as fd:
                for chunk in r.iter_content(chunk_size):
                    fd.write(chunk)
            path = f'{salva_pdf}.pdf'
            with open(path, 'rb') as opened_file:
                content = opened_file.read()
                md5 = hashlib.md5()
                md5.update(content)
                md5_pdf = md5.hexdigest()          
            lista_preps.append([data.strftime('%Y-%m-%d'), titulo, id_preposicao, md5_pdf])
        else:
            pass
    
    return lista_preps
```
***
3) A lista que recebemos precisa conter todas as proposições dos ultimos __3 dias__, mas nem sempre essa lista está contida apenas na primeira página que o crawler faz o *scrape*.
<br></br>
É retornado uma lista comlistas de todas as proposições dos últimos 3 dias de todas as páginas.


## * limita_props_dos_3_ultimos_dias()

Chama a função __acessa_api_camara()__ e recebe o __json com os dados das proposições__.

Busca a quantidade total de paginas com proposições e aloca na variavel *qtd_pags*.

Busca a última data da primeira página. Caso seja mais do que os 3 dias, retorna uam lista da __funcao retorna_props()__

Caso seja menos do que 2 dias, vai para a página seguinte e faz a mesma busca até que a data seja maior que 3 dias,
fazerndo um break e retornando a lista da função __retorna_props()__

```
def limita_props_dos_3_ultimos_dias(tipo_proposicao):    
    n_i = 1
    resp_json = acessa_api_camara(ano_atual, tipo_proposicao, n_i)
    qtd_total = resp_json['aggregations']['ano']['buckets'][0]['doc_count']
    qtd = len(resp_json['hits']['hits'])
    qtd_pags = ceil(qtd_total / qtd)
    datau = resp_json['hits']['hits'][(qtd-1)]['_source']['dataApresentacao'][:10]
    datau = datetime.strptime(datau, '%Y-%m-%d').date()
    if datau >= dias_3:
        lista_preposicoes = []
        while n_i < qtd_pags:        
            resp_json = acessa_api_camara(ano_atual, tipo_proposicao, n_i)
            n_i += 1
            qtd_paginado = len(resp_json['hits']['hits'])
            rr = retorna_props(qtd_paginado, resp_json, dias_3)
            if rr == []:
                break        
            lista_preposicoes.append(rr)
        return list(itertools.chain.from_iterable(lista_preposicoes))
    else:
        return retorna_props(qtd, resp_json, dias_3)
```
***
4) A lista de listas é convertida em um *DataFrame*.
<br></br>
Um *DataFrame Pandas* é retornado.

## * exporta_dataframe()

Exporta um dataframe para o tipo de proposição *PL, PLP ou PEC*.

Remove o pdf usado para fazer o hash md5 da pasta.

Monta e retorna um dataframe com a lista recebida.

```
def exporta_dataframe(tipo_proposicao):
    try:
        salva_pdf = 'salva_pdf'
        path = f'{salva_pdf}.pdf'        
        os.remove(path)
    except:
        pass
    try:
        sigalei = limita_props_dos_3_ultimos_dias(tipo_proposicao)        
        df = pd.DataFrame(sigalei, columns = ['DATA', 'PROJETO', 'INDEX_PROJETO', 'MD5'])
    except:    
        sigalei = limita_props_dos_3_ultimos_dias(tipo_proposicao)
        loop = len(sigalei)
        print(len)
        x = 0
        df_list = []
        while x < loop:
            df_loop = pd.DataFrame(sigalei[x], columns = ['DATA', 'PROJETO', 'INDEX_PROJETO', 'MD5'])
            df_list.append(df_loop)
            x +=1
        df = pd.concat(df_list)
    return df    
```
***
5) Salva o *DataFrame* em __CSV__.

## * salva_csv()

Salva o dataframe em CSV

```
def salva_csv(df, p):
    return df.to_csv(f'{p}.csv', index=False, sep=';')
```


6) Salva o *DataFrame* em __JSON__.

## * salva_json()

Salva o dataframe em JSON

```
def salva_json(df, p):    
    retorno_json = df.to_json(orient='records')
    with open(f'{p}.json', 'w') as f:
        json.dump(json.loads(retorno_json) , f)
```
